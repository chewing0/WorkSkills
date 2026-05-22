---
tags:
  - LLM
  - Tokenizer
  - Embedding
  - 面试八股
created: 2026-05-21
description: Tokenizer 与 Embedding 详解，覆盖 BPE、WordPiece、SentencePiece、特殊 token、chat template、token 预算、embedding table、LM Head、weight tying 与工程常见坑
---

> **核心考点**：LLM 不能直接处理字符串。文本先被 tokenizer 切成 token id，再通过 embedding table 变成向量。Tokenizer 决定 token 数、上下文成本、多语言/代码效率、特殊格式边界；Embedding 决定 token 进入模型时的初始表示，LM Head 决定 hidden state 如何映射回词表。

---

# 1. 核心问题：Tokenizer 为什么决定模型怎么读文本

神经网络处理的是数字张量，不是原始字符串。LLM 的输入链路是：

```text
文本
  │
  ├─► tokenizer.encode
  │      └─ token ids
  │
  ├─► embedding lookup
  │      └─ token vectors
  │
  └─► Transformer
```

例子：

```text
文本:    我喜欢大语言模型
tokens:  我 | 喜欢 | 大 | 语言 | 模型
ids:     234 | 8921 | 17 | 5301 | 781
```

注意：这只是示意。真实模型的切分结果取决于它自己的 tokenizer。

Tokenizer 会影响：

- 同一文本占多少 token。
- 上下文窗口是否够用。
- API 计费。
- prefill 延迟。
- KV Cache 显存。
- 中文、代码、公式、表格的处理效率。
- chat template、工具调用、结构化输出是否稳定。

---

# 2. Token、字符、词、字节的区别

| 单位 | 例子 | 特点 |
|---|---|---|
| 字符 | `你`、`a`、`!` | 人类可见字符，不一定等于模型单位 |
| 字节 | UTF-8 bytes | 任意文本都能表示，但序列可能很长 |
| 词 | `language`、`模型` | 语义直观，但词表巨大 |
| 子词 | `lang`、`uage`、`##ing` | 现代 LLM 常用折中 |
| token | tokenizer 输出单位 | 可能是字、词、子词、空格、符号或字节片段 |

一个 token 不等于一个汉字，也不等于一个英文单词。

例如英文：

```text
unbelievable -> un | believ | able
```

中文：

```text
大语言模型 -> 大 | 语言 | 模型
```

代码：

```text
get_user_profile -> get | _user | _profile
```

---

# 3. 基础机制：Tokenizer 的基本流程

典型流程：

```text
原始文本
  │
  ├─► normalization：大小写、Unicode、空格等规范化
  ├─► pre-tokenization：按空格/标点/字节等粗切
  ├─► subword segmentation：BPE/WordPiece/Unigram 等
  ├─► special tokens：加入 BOS/EOS/role/tool 等
  └─► token ids
```

不同 tokenizer 的差异可能来自：

- 是否区分大小写。
- 如何处理空格。
- 如何处理中文。
- 是否 byte-level。
- 是否有 byte fallback。
- 词表大小。
- 训练语料语言分布。
- 特殊 token 和 chat template。

---

# 4. 同族概念分组：子词切分算法家族

BPE、WordPiece、SentencePiece、Unigram 不应该分开硬背。它们都在解决同一个问题：

> 如果按词切，词表会爆炸；如果按字符或字节切，序列会太长。子词算法要在“词级语义”和“字符级泛化”之间找折中。

先看切分粒度：

| 粒度 | 例子 | 优点 | 缺点 |
|---|---|---|---|
| 字符级 | 每个字母或汉字一个 token | 不会 OOV，简单 | 序列长，语义弱 |
| 词级 | 每个单词一个 token | 语义直观 | 词表巨大，生僻词困难 |
| 子词级 | `un`, `believ`, `able` | 平衡词表和泛化 | 切分不一定符合直觉 |
| 字节级 | UTF-8 bytes | 任意文本都能表示 | token 更碎，学习更难 |

现代 LLM 多使用子词或字节级子词方案。

## 4.1 BPE：从小片段逐步合并

BPE（Byte Pair Encoding）从基础符号开始，反复合并语料中最常见的相邻片段。

简化训练过程：

```text
l o w
l o w e r
n e w e s t

统计相邻 pair 频率:
l o 频繁 -> 合并为 lo
lo w 频繁 -> 合并为 low
e r 频繁 -> 合并为 er
```

最终得到一组 merge rules。

编码时：

```text
lowest -> low | est
```

优点：

- 高频词可以成为完整 token。
- 低频词可以拆成子词。
- 词表规模可控。
- 对英文和代码效果好。

局限：

- 切分可能不符合语义。
- 多语言效率取决于训练语料。
- 没有 byte fallback 时，罕见字符处理麻烦。

## 4.2 Byte-level BPE 与 Byte Fallback：先保证任何文本都能表示

Byte-level BPE 从字节开始，而不是从 Unicode 字符开始。好处是任何文本都能表示，不容易出现 UNK。

例子：

```text
罕见字符 -> UTF-8 bytes -> byte tokens
```

Byte fallback 的思想是：

> 如果某个字符或片段不在词表中，就退回到字节级表示。

优点：

- 几乎不会 OOV。
- 能处理 emoji、罕见汉字、特殊符号、混合编码。

缺点：

- 罕见字符会被切得很碎。
- token 数增加，成本上升。

## 4.3 WordPiece：更偏语言模型似然的子词构建

WordPiece 与 BPE 类似，也构建子词词表，但合并标准更偏向语言模型似然提升。BERT 系列常见。

常见表示：

```text
playing -> play ##ing
unwanted -> un ##want ##ed
```

`##` 表示这个子词不是词开头。

特点：

- 适合 BERT 类 Encoder-only 模型。
- 对英文词形变化友好。
- `[UNK]` 在传统 WordPiece 中更常见。

## 4.4 SentencePiece 与 Unigram：把空格也当成模型问题

SentencePiece 是一个 tokenizer 工具框架，常用于 T5、Llama 等模型。它的特点是不依赖语言特定分词器，直接从原始文本学习子词。

常见方案：

- BPE。
- Unigram Language Model。

### 空格处理

SentencePiece 常用特殊符号表示空格，例如：

```text
Hello world -> ▁Hello | ▁world
```

`▁` 表示词前空格。

这使得 tokenizer 能在没有天然空格的语言和有空格的语言之间保持统一处理。

### Unigram

Unigram 不是反复合并 pair，而是假设一个候选子词词表，然后通过概率模型选择最可能的切分，并逐步剪枝词表。

直觉：

- BPE 是“从小到大合并”。
- Unigram 是“从大候选词表中删到合适大小”。

## 4.5 BPE、WordPiece、SentencePiece 怎么比较

| 方案 | 常见模型 | 核心思想 | 特点 |
|---|---|---|---|
| BPE | GPT 类、很多开源 LLM | 高频 pair 合并 | 简单高效，代码友好 |
| Byte-level BPE | GPT-2 等 | 从字节出发合并 | 几乎无 OOV |
| WordPiece | BERT 类 | 基于似然的子词构建 | 常见 `##` 标记 |
| SentencePiece | T5、Llama 类 | 原始文本子词学习框架 | 多语言友好，常用 `▁` 表空格 |
| Unigram | SentencePiece 可选 | 子词概率模型剪枝 | 可产生多种候选切分 |

面试不一定要推公式，重点是理解：

> 这些方法都是为了解决词表太大和未知词问题，在“词级语义”和“字符级泛化”之间做折中。

---

# 5. 同族概念分组：特殊 Token 与 Chat Template

模型通常有一些特殊 token，用来表示结构和控制信息。

| token | 含义 |
|---|---|
| BOS | 序列开始 |
| EOS | 序列结束 |
| PAD | batch padding |
| UNK | 未知 token |
| MASK | 掩码 token，BERT 类模型常用 |
| SEP | 句子或段落分隔 |
| CLS | 分类聚合位，BERT 类常用 |
| system/user/assistant | Chat 消息角色 |
| tool/function | 工具调用边界 |

特殊 token 不是普通文本。它们通常有固定 id，并在训练中被模型赋予结构含义。

## 5.1 BOS/EOS

BOS 表示序列开始，EOS 表示序列结束。

生成时，如果模型输出 EOS，通常表示停止。

## 5.2 PAD 与 Attention Mask

batch 中不同样本长度不同，需要 padding：

```text
样本1: A B C D
样本2: A B PAD PAD
```

PAD 不应该被真实 token 关注，所以要配合 attention mask。

## 5.3 Chat Role Token

Chat 模型通常不是直接训练在纯文本上，而是使用角色模板：

```text
<system>你是一个严谨的助手。</system>
<user>解释 KV Cache。</user>
<assistant>KV Cache 是...</assistant>
```

不同模型的角色 token 和模板不同，不能混用。

---

## 5.4 Chat Template

Chat template 把多轮 messages 转成模型实际看到的 token 序列。

用户以为输入的是：

```json
[
  {"role": "system", "content": "你是一个助手。"},
  {"role": "user", "content": "解释 Transformer。"}
]
```

模型实际看到的可能是：

```text
<|system|>
你是一个助手。
<|user|>
解释 Transformer。
<|assistant|>
```

为什么重要：

- 模型是在特定模板上训练的。
- 模板错了会导致角色混乱。
- 工具调用边界依赖模板。
- 多轮对话的停止位置依赖模板。
- system/user/assistant 的优先级通过模板体现。

常见坑：

- 用 A 模型的 chat template 调 B 模型。
- 忘记添加 assistant generation prompt。
- 把工具结果当 user 消息。
- 特殊 token 被转义成普通文本。
- 手写模板少了 EOS 或换行。

---

# 6. 工程含义：词表大小影响效率和泛化

词表大小记为 $V$。Embedding table 参数量：

$$
V \times d_{model}
$$

例如：

```text
V = 100000
d_model = 4096
embedding 参数量 = 409,600,000
```

如果使用 FP16，大约占：

```text
409.6M * 2 bytes ≈ 819MB
```

词表越大：

- embedding 和 LM Head 参数越多。
- 输出 logits 维度越大。
- 高频多语言 token 覆盖更好。
- 但训练和推理的输出层成本更高。

词表越小：

- 参数更少。
- 罕见词和多语言文本可能切得更碎。
- 长文本 token 数增加。

---

# 7. 工程含义：扩展词表与新增特殊 Token

微调或应用中，有时要新增特殊 token：

```text
<tool_call>
</tool_call>
<image>
<domain_term>
```

新增 token 后要做两件事：

1. 更新 tokenizer 词表。
2. 扩展模型 embedding / LM Head 矩阵。

如果新增 token 没有训练：

- embedding 是随机初始化或平均初始化。
- 模型不懂它的语义。
- 生成和识别都可能不稳定。

所以新增 token 后通常需要继续训练或 SFT，让模型学会这些 token 的用法。

---

# 8. 基础机制：Embedding 是什么

Tokenizer 输出 token id 后，模型通过 embedding table 查表，把离散 id 变成连续向量：

$$
E \in \mathbb{R}^{V \times d_{model}}
$$

如果 token id 是 $i$：

$$
x_i = E[i]
$$

这里的 embedding 是模型参数，在训练中学习。

它会编码：

- 词义。
- 语法。
- 子词关系。
- 格式符号。
- 代码模式。
- 特殊 token 角色。

---

# 9. 同族概念分组：Embedding Table 与 Embedding Model

这是很容易混淆的点。

| 名称 | 含义 | 输出 | 用途 |
|---|---|---|---|
| Embedding table | LLM 内部 token id 查表矩阵 | token 初始向量 | 进入 Transformer |
| Embedding model | 专门把文本映射成句向量的模型 | 文本向量 | 检索、聚类、RAG |

LLM 内部 embedding table：

```text
token id -> token vector
```

RAG embedding model：

```text
一段文本 -> 一个向量
```

二者不是一回事。

---

# 10. 基础机制：输入 Embedding、位置编码与 Hidden State

模型输入通常不是只有 token embedding。

早期 Transformer 常见：

$$
x_i = token\_embedding_i + position\_embedding_i
$$

现代 RoPE 模型中，位置信息通常不直接加到输入 embedding，而是在每层 Attention 中作用于 Q/K。

进入 Transformer 后，每层都会更新 token 表示：

```text
input embedding -> layer 1 hidden state -> ... -> final hidden state
```

重要区别：

- **Input embedding**：同一个 token id 初始向量固定。
- **Hidden state**：同一个 token 在不同上下文中会变。

例子：

```text
苹果 很 甜
苹果 公司 发布 新品
```

“苹果”的 input embedding 相同，但经过上下文后 hidden state 不同。

---

# 11. 基础机制：LM Head 与 Weight Tying

模型最后需要把 hidden state 映射回词表 logits：

$$
logits = h W_{vocab}
$$

其中：

$$
W_{vocab} \in \mathbb{R}^{d_{model} \times V}
$$

很多语言模型共享输入 embedding 和输出 LM Head 权重：

$$
W_{vocab} = E^T
$$

这叫 weight tying。

优点：

- 减少参数量。
- 输入 token 和输出 token 使用同一语义空间。
- 对语言模型通常有效。

不是所有模型都必须 tie。具体取决于架构和实现。

---

# 12. 工程含义：Tokenizer 对中文的影响

中文没有天然空格，tokenizer 质量会直接影响效率。

可能切分：

```text
大语言模型 -> 大 | 语言 | 模型
大语言模型 -> 大 | 语 | 言 | 模 | 型
大语言模型 -> 大语言 | 模型
```

影响：

- token 数影响成本。
- 罕见人名、地名、专业术语可能切得很碎。
- 中英混合文本依赖训练语料覆盖。
- 如果中文语料不足，中文文本 token 效率会差。

实际评估模型时，除了看 benchmark，还要看你的业务文本 token 数。

---

# 13. 工程含义：Tokenizer 对代码的影响

代码包含：

- 缩进。
- 下划线。
- 驼峰命名。
- 括号。
- 操作符。
- 长变量名。
- 路径。
- 日志。

好的代码 tokenizer 会更高效地表示：

```text
def get_user_profile(user_id):
    return db.query(User).filter_by(id=user_id).first()
```

如果 tokenizer 对代码不友好：

- 长函数很快占满上下文。
- 变量名被切得很碎。
- patch/diff 更难生成稳定。
- 代码模型训练效率变差。

---

# 14. 工程含义：Token 预算

上下文窗口按 token 计算，不按字符或字数计算。

```text
总 token =
  system prompt
  + chat template 特殊 token
  + 历史对话
  + 用户输入
  + RAG 文档
  + 工具 schema
  + 工具结果
  + 预留输出 token
```

常见策略：

- 为输出预留 token。
- 对历史对话摘要。
- RAG chunk 不要过大。
- 工具 schema 精简。
- 工具结果结构化。
- 长表格和日志先压缩。
- 对不同模型分别估算 token 数。

## 14.1 输入 token vs 输出 token

输入 token 主要影响：

- prefill 时间。
- prompt 成本。
- KV Cache 初始大小。

输出 token 主要影响：

- decode 时间。
- completion 成本。
- 用户等待时间。

---

# 15. 工程含义：Tokenizer 与安全

Tokenizer 也会影响安全策略。

例子：

- 敏感词可能被切成多个 token。
- 同形异体字、Unicode 混淆可能绕过规则。
- 空格、零宽字符可能改变匹配。
- Base64、URL 编码、emoji 组合可能隐藏内容。

因此安全过滤不能只做简单字符串匹配。常见策略：

- 文本规范化。
- Unicode 归一化。
- 多粒度匹配。
- 模型审核。
- 工具侧权限控制。

---

# 16. 常见误区与工程坑

| 问题 | 影响 | 建议 |
|---|---|---|
| 估算 token 用错 tokenizer | 成本和截断判断错误 | 每个模型用自己的 tokenizer |
| Chat template 不匹配 | 角色混乱、输出异常 | 使用模型官方模板 |
| 忘记 EOS | 模型停不住 | 正确配置 stop/EOS |
| PAD 没有 mask | padding 污染注意力 | 配 attention mask |
| 新增 token 未训练 | 模型不懂新 token | 继续训练或 SFT |
| JSON 特殊字符被切碎 | 输出不稳定 | 用 schema/function calling |
| 中文/代码 token 效率差 | 成本高、上下文短 | 选适配语料的模型 |
| 手写 prompt 忽略模板 token | 实际 token 超预算 | 用 tokenizer 真实 encode |

---

# 17. 面试 Q&A

## Q1：为什么同样文本在不同模型中 token 数不同？

**答**：不同模型使用不同 tokenizer，词表、合并规则、空格处理、多语言语料和特殊 token 不同。同一句话可能被切成不同数量的 token，因此成本、延迟和上下文占用也不同。

## Q2：BPE 为什么能处理没见过的词？

**答**：BPE 使用子词或字节片段表示文本。没见过的词可以拆成已知子词组合；如果是 byte-level 或有 byte fallback，还可以退回字节表示，因此不需要为每个完整词都建词表。

## Q3：BPE、WordPiece、SentencePiece 的区别是什么？

**答**：BPE 基于高频 pair 合并；WordPiece 也做子词切分，但合并标准更偏语言模型似然，BERT 常用；SentencePiece 是从原始文本训练子词的工具框架，不依赖空格分词，常用于多语言模型。

## Q4：Embedding 是静态还是动态的？

**答**：输入 embedding table 是静态参数，同一个 token id 初始向量固定。但经过 Transformer 上下文建模后，每个位置的 hidden state 是动态的，会随上下文变化。

## Q5：Embedding table 和 embedding 模型有什么区别？

**答**：Embedding table 是 LLM 内部把 token id 查成向量的参数矩阵；embedding 模型是把一段文本编码成向量的独立模型，常用于语义检索、聚类和 RAG。

## Q6：为什么 token 数会影响推理成本？

**答**：输入 token 增加会提高 prefill 计算和 KV Cache 占用；输出 token 增加会延长 decode 时间。上下文窗口、延迟和 API 计费通常都按 token 计算。

## Q7：为什么 Chat 模型需要 chat template？

**答**：Chat 模型训练时看到的是带 system/user/assistant 边界的序列。chat template 把 messages 转成模型熟悉的格式。模板不匹配会导致角色混乱、停止错误或工具调用失败。

## Q8：新增特殊 token 后为什么还要训练？

**答**：新增 token 的 embedding 通常没有语义，可能是随机初始化。模型不知道它何时出现、代表什么、如何生成，所以需要继续训练或 SFT 学会使用。

## Q9：为什么中文模型要关注 tokenizer？

**答**：中文没有天然空格，如果 tokenizer 对中文覆盖不足，一个词会被切成很多 token，导致成本高、上下文变短、训练和推理效率下降。

## Q10：为什么不能用字符数估算上下文？

**答**：上下文窗口按 token 计算。英文、中文、代码、emoji、空格和特殊符号的 token/token-per-char 比例不同，必须用目标模型 tokenizer 实际 encode 才准确。

---

# 18. 记忆口诀

```text
Tokenizer：文本切 token。
Token id：离散编号。
Embedding table：id 查向量。
Hidden state：上下文后的动态表示。
LM Head：向量映射回词表 logits。
Chat template：多轮消息真正进模型的样子。
Token 预算：输入、历史、RAG、工具、输出都要算。
```

---

# 19. 相关链接

- [[README|LLM 基础目录]]
- [[00-LLM-Overview|LLM 总览]]
- [[01-Transformer|Transformer]]
- [[03-LLM-Variants|主流模型架构差异]]
- [[05-Generation-Strategies|生成策略]]
- [[06-Context-Window-KV-Cache|上下文窗口与 KV Cache]]

---

> **参考**
> - Sennrich et al. ["Neural Machine Translation of Rare Words with Subword Units"](https://arxiv.org/abs/1508.07909) (2015)
> - Kudo and Richardson. ["SentencePiece: A simple and language independent subword tokenizer and detokenizer for Neural Text Processing"](https://arxiv.org/abs/1808.06226) (2018)
> - Schuster and Nakajima. "Japanese and Korean Voice Search" (2012)
