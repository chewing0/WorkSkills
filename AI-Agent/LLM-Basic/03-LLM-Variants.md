---
tags:
  - LLM
  - 模型架构
  - 面试八股
created: 2026-05-21
description: 主流 LLM 架构差异详解，从注意力可见性和训练目标出发，讲清 Encoder-only、Decoder-only、Encoder-Decoder、Dense/MoE、Base/Instruct/Chat 与模型选型
---

> **核心考点**：模型架构差异不是模型名字的差异，而是三个根问题的差异：**谁能看见谁、训练时让模型做什么、推理时模型怎样输出**。BERT、GPT、T5、Llama、Qwen、DeepSeek 的区别，都可以从这三个问题推出来。

---

# 1. 核心问题：不同 LLM 架构到底差在哪里

所有 Transformer 语言模型都在处理 token 序列：

```text
token_1, token_2, token_3, ..., token_n
```

每一层模型都在做两件事：

1. **读上下文**：当前 token 从哪些 token 拿信息？
2. **更新表示**：拿到信息后，当前 token 的向量怎么变化？

不同架构的核心区别，首先不是参数量，也不是模型名，而是第一个问题：

> 当前 token 能看见哪些 token？

这叫 **注意力可见性**。

理解了注意力可见性，就能理解：

- 为什么 BERT 擅长理解。
- 为什么 GPT 擅长生成。
- 为什么 T5 适合翻译、摘要、改写。
- 为什么 Decoder-only 成为通用 Chat/Agent 模型主流。

---

# 2. 基础机制：三种“看见方式”

假设输入有 4 个 token：

```text
1 2 3 4
```

## 2.1 双向可见：适合理解

```text
        key position
        1  2  3  4
query 1 ✓  ✓  ✓  ✓
query 2 ✓  ✓  ✓  ✓
query 3 ✓  ✓  ✓  ✓
query 4 ✓  ✓  ✓  ✓
```

每个位置都能看完整句子。这就是 Encoder-only 的基本模式。

它像“阅读理解”：先把题目和文章全部看完，再判断答案。

## 2.2 只能看过去：适合生成

```text
        key position
        1  2  3  4
query 1 ✓  ✗  ✗  ✗
query 2 ✓  ✓  ✗  ✗
query 3 ✓  ✓  ✓  ✗
query 4 ✓  ✓  ✓  ✓
```

第 3 个 token 只能看 1、2、3，不能看第 4 个。这就是 Decoder-only 的基本模式。

它像“从左到右写文章”：写到当前字时，未来的字还不存在。

## 2.3 先读完整输入，再逐步写输出

Encoder-Decoder 把“读”和“写”拆开：

```text
输入序列 -> Encoder 双向读完
              │
              ▼
输出序列 <- Decoder 一步步生成，并通过 Cross-Attention 回看输入
```

它像“翻译”：先读完整英文句子，再逐步写中文译文。

---

# 3. 三种架构一句话

| 架构 | 一句话理解 | 代表模型 | 最适合 |
|---|---|---|---|
| Encoder-only | 只读，不天然写长文本 | BERT、RoBERTa、DeBERTa | 理解、分类、抽取、rerank |
| Decoder-only | 边读边写，只看过去 | GPT、Llama、Qwen、DeepSeek、Mistral | 对话、写作、代码、Agent |
| Encoder-Decoder | Encoder 读，Decoder 写 | T5、BART、原始 Transformer | 翻译、摘要、改写、Seq2Seq |

这里的“只读”和“写”是帮助理解的说法，不是说 Encoder-only 完全不能生成，而是说它的训练方式和结构不天然适合自回归长文本生成。

---

# 4. Encoder-only：为什么 BERT 擅长理解

Encoder-only 模型只保留 Transformer Encoder。它的关键能力是：**对整段输入做双向上下文表示**。

## 4.1 它到底在学什么

BERT 类模型常用 MLM（Masked Language Modeling）：

```text
输入: 我 喜欢 [MASK] 语言模型
目标: 大
```

模型可以同时看 `[MASK]` 左右两边：

```text
左边: 我 喜欢
右边: 语言模型
```

所以它学到的是：

> 给定完整上下文，某个位置应该是什么，或者这段文本整体表示什么。

这非常适合“理解”，因为理解任务通常允许模型先看完整输入。

## 4.2 一个分类任务怎么做

情感分类例子：

```text
[CLS] 这家店服务很好，下次还来。 [SEP]
```

BERT 读完整句话后，取 `[CLS]` 位置的 hidden state 接一个分类头：

```text
hidden([CLS]) -> linear -> positive / negative
```

所以 Encoder-only 常见用法是：

- 在句子级表示上接分类头。
- 在每个 token 上接序列标注头。
- 输入 query + document 后输出相关性分数。

## 4.3 为什么适合 rerank

RAG 中常见两段式检索：

```text
query -> embedding 召回 top 100 文档
query + doc -> reranker 精排 top 5
```

Cross-Encoder reranker 会把 query 和 document 拼在一起，让它们充分双向交互：

```text
[CLS] query [SEP] document [SEP]
```

这比单纯向量相似度更细，但也更慢。

## 4.4 为什么不适合直接做 ChatGPT

关键不是“BERT 不够大”，而是目标和可见性不匹配。

聊天生成要求：

```text
已生成 token -> 预测下一个 token -> 再预测下一个 token
```

但 BERT 训练时看到的是双向上下文：

```text
左文 + [MASK] + 右文
```

它习惯“补空”，不是“从左到右连续写”。因此 BERT 很适合理解和打分，但不天然适合开放式对话生成。

---

# 5. Decoder-only：为什么 GPT 类模型擅长生成

Decoder-only 模型只保留因果自注意力。它的核心训练目标是：

$$
P(x_t \mid x_{<t})
$$

也就是根据历史 token 预测下一个 token。

## 5.1 训练时发生了什么

给定一句话：

```text
Transformer 的核心机制是 注意力
```

训练样本可以看成很多预测任务：

```text
Transformer -> 的
Transformer 的 -> 核心
Transformer 的 核心 -> 机制
Transformer 的 核心 机制 -> 是
Transformer 的 核心 机制 是 -> 注意力
```

实际训练中会并行计算这些位置的 loss，但通过 causal mask 保证每个位置不能偷看未来。

## 5.2 为什么一个简单目标能学到很多能力

next token prediction 看起来只是“猜下一个词”，但要猜得好，模型必须压缩大量规律：

- 要猜语法，就要学语言结构。
- 要猜事实，就要学世界知识。
- 要猜代码，就要学程序模式。
- 要猜题解，就要学推理步骤。
- 要猜对话，就要学问答习惯。

这就是 Decoder-only 能扩展成通用模型的根本原因。

## 5.3 为什么它能做理解任务

分类任务可以改写成生成任务。

原始分类：

```text
判断情感：这家店服务很好，下次还来。
```

生成式格式：

```text
请判断情感，只输出 positive 或 negative：
这家店服务很好，下次还来。

答案：
```

模型生成：

```text
positive
```

抽取、摘要、改写、问答、工具调用也都能转成“生成下一个 token”。这就是 Decoder-only 统一任务接口的原因。

## 5.4 为什么推理时需要 KV Cache

生成时，模型每次只多生成一个 token：

```text
step 1: prompt -> y1
step 2: prompt + y1 -> y2
step 3: prompt + y1 + y2 -> y3
```

历史 token 的 Key/Value 不会变，所以可以缓存。Decoder-only 的自回归结构非常适合 KV Cache。

这也是它能成为大规模在线聊天模型主流的工程原因之一。

## 5.5 它的限制

Decoder-only 不是万能的：

- 生成是串行的，decode 阶段天然慢。
- 对事实没有内置数据库保证。
- 长上下文中间信息可能被忽略。
- 结构化输出需要 schema、parser、retry。
- 高风险任务需要 RAG、工具、校验和安全策略。

所以一个好 Agent 系统不是“只丢给 GPT”，而是围绕 Decoder-only 模型构建检索、工具、状态和评估。

---

# 6. Encoder-Decoder：为什么适合输入到输出转换

Encoder-Decoder 把任务拆成两步：

```text
读输入 -> 写输出
```

## 6.1 以翻译为例

输入：

```text
I love natural language processing.
```

Encoder 双向读完整句子，得到一组输入表示。

Decoder 生成中文：

```text
我 -> 喜欢 -> 自然 -> 语言 -> 处理
```

生成每个中文 token 时，Decoder 一方面看已经生成的中文，另一方面通过 Cross-Attention 回看英文输入。

## 6.2 Cross-Attention 在干什么

普通 Decoder-only self-attention 是：

```text
输出 token 之间互相看历史
```

Encoder-Decoder 的 Cross-Attention 是：

```text
Decoder 当前状态 -> 查询 Encoder 输出
```

用 Q/K/V 语言说：

- Query 来自 Decoder。
- Key/Value 来自 Encoder。

这让 Decoder 在生成时知道源文本哪些部分重要。

## 6.3 T5 的统一思想

T5 把所有任务都改写成 text-to-text：

```text
translate English to Chinese: I love NLP.
```

输出：

```text
我喜欢自然语言处理。
```

分类也可以变成：

```text
sst2 sentence: this movie is great
```

输出：

```text
positive
```

这个思想很重要：任务形式统一后，模型架构和训练流程可以统一。

## 6.4 为什么通用 Chat 没有主要走这条路

不是 Encoder-Decoder 不强，而是通用 Chat/Agent 有几个特点：

- 多轮历史可以自然拼成一个长 token 序列。
- 工具调用也可以表示成生成结构化 token。
- 训练数据可以统一成 next token prediction。
- KV Cache 和推理服务生态主要围绕 Decoder-only 优化。

因此 Decoder-only 更适合成为通用接口。

---

# 7. 三大架构对比：把基础差异放在一张表

| 问题 | Encoder-only | Decoder-only | Encoder-Decoder |
|---|---|---|---|
| token 能看见谁 | 全部输入 token | 当前和历史 token | Encoder 看全部输入，Decoder 看历史输出并看 Encoder |
| 像什么 | 阅读理解 | 从左到右写文章 | 先读题，再写答案 |
| 常见训练目标 | MLM | Next Token Prediction | Seq2Seq / Span Corruption |
| 推理输出 | 常接任务头 | 自回归生成 | 自回归生成 |
| 是否天然适合聊天 | 否 | 是 | 可以，但不如 Decoder-only 生态统一 |
| 典型用途 | 分类、抽取、rerank、embedding | 对话、代码、Agent、通用生成 | 翻译、摘要、改写 |

记住这张表，比背模型名字更重要。

---

# 8. 为什么现在通用 LLM 多是 Decoder-only

可以从四个角度理解。

## 8.1 数据角度

互联网上大部分文本天然就是序列：

```text
token_1 token_2 token_3 ...
```

Decoder-only 的 next token prediction 可以直接利用这些数据。

## 8.2 任务角度

很多任务都能转成生成：

| 任务 | 生成式写法 |
|---|---|
| 分类 | 生成标签 |
| 抽取 | 生成字段 JSON |
| 摘要 | 生成摘要 |
| 问答 | 生成答案 |
| 代码 | 生成代码 |
| 工具调用 | 生成函数名和参数 |

这让同一个模型可以覆盖很多应用。

## 8.3 工程角度

Decoder-only 推理虽然逐 token 串行，但有成熟优化：

- KV Cache。
- Continuous Batching。
- PagedAttention。
- FlashAttention。
- Speculative Decoding。
- 量化。

这些工程优化让它非常适合在线服务。

## 8.4 对齐角度

SFT、RLHF、DPO、工具调用训练、Chat Template 等生态大多围绕 Decoder-only 展开。生态越成熟，越容易继续强化它的主流地位。

---

# 9. GPT、BERT、T5 的核心区别

| 模型家族 | 架构 | 训练目标 | 本质能力 | 典型输出方式 |
|---|---|---|---|---|
| BERT | Encoder-only | MLM | 把完整输入编码成好用表示 | 分类头、抽取头、相关性分数 |
| GPT | Decoder-only | Next Token Prediction | 根据历史上下文继续生成 | token by token 生成 |
| T5 | Encoder-Decoder | Text-to-Text / Span Corruption | 读输入并生成目标输出 | Seq2Seq 生成 |

一句话：

```text
BERT：读懂一句话。
GPT：接着往下写。
T5：读完输入，再写输出。
```

---

# 10. 现代 Decoder-only 配方

现代 LLM 通常不是原始 Transformer 的简单复刻，而是经过工程验证的一套组合。

常见配方：

```text
Decoder-only
  + Causal Mask
  + RoPE
  + PreNorm / RMSNorm
  + SwiGLU
  + MHA/GQA/MQA/MLA
  + KV Cache
  + FlashAttention
  + SFT / DPO / RLHF
```

## 10.1 为什么常用 RoPE

绝对位置 embedding 直接告诉模型“这是第几个 token”。RoPE 把相对位置关系放进 Q/K 的内积里，更适合自回归和长度扩展。

直觉：

> 生成时模型更关心“当前 token 和前面 token 相距多远”，而不只是“它是第几个 token”。

## 10.2 为什么常用 RMSNorm + SwiGLU

RMSNorm：

- 比 LayerNorm 稍简化。
- 在大模型中训练稳定。
- 常与 PreNorm 结构搭配。

SwiGLU：

- 是一种门控 MLP。
- 比普通 ReLU/GELU FFN 表达更灵活。
- 现代 Decoder-only LLM 中非常常见。

## 10.3 为什么出现 GQA/MQA/MLA

普通 MHA 中，每个 attention head 都有自己的 K/V。长上下文推理时，KV Cache 会很大。

GQA/MQA/MLA 这类方法的共同目标之一是：

> 减少或压缩 K/V 相关缓存，降低长上下文和高并发推理成本。

| 机制 | 直觉 |
|---|---|
| MHA | 每个 Query head 都有自己的 K/V，表达强但 cache 大 |
| MQA | 多个 Query head 共享一组 K/V，cache 小 |
| GQA | 多个 Query head 分组共享 K/V，质量和效率折中 |
| MLA | 用低秩 latent 方式压缩 K/V 相关信息 |

---

# 11. 同族概念分组：模型家族怎么理解

模型版本更新很快，不建议死记某一版参数。更好的方式是记“家族路线”。

## 11.1 GPT 类

GPT 类代表 Decoder-only Causal LM 路线。

理解重点：

- 预训练目标是 next token prediction。
- Chat 能力来自后续指令微调和偏好对齐。
- 工具调用、结构化输出、多模态能力都是在基础生成接口上继续扩展。

## 11.2 BERT 类

BERT 类代表 Encoder-only 理解路线。

理解重点：

- 双向看完整输入。
- 输出通常不是直接生成长文本，而是接任务头或打分。
- 在 rerank、分类、抽取、安全审核中仍然很有价值。

## 11.3 T5/BART 类

T5/BART 类代表 Encoder-Decoder 转换路线。

理解重点：

- Encoder 负责读输入。
- Decoder 负责写输出。
- Cross-Attention 把两者连接起来。
- 翻译、摘要、改写非常自然。

## 11.4 Llama/Qwen/Mistral 类

这类开源模型常体现现代 Decoder-only 配方。

理解重点：

- 架构上多采用 RoPE、RMSNorm、SwiGLU、GQA 等组合。
- 生态上支持微调、量化、本地部署。
- 适合企业私有化、RAG、Agent、代码助手等应用。

## 11.5 DeepSeek / Mixtral 等 MoE 路线

这类模型常强调效率和大容量。

理解重点：

- MoE 不是每个 token 用全部专家。
- 总参数量大，不等于每 token 激活参数同样大。
- 训练和部署更复杂，但性价比可能很好。

---

# 12. Dense 与 MoE：从 FFN 开始理解

Transformer Block 里有 Attention 和 FFN。很多研究和工程实践认为，FFN 承载了大量模式和知识。

Dense 模型：

```text
每个 token -> 同一个 FFN
```

MoE 模型：

```text
每个 token -> Router -> 少量专家 FFN -> 合并
```

## 12.1 为什么 MoE 可以“参数大但计算不等比例变大”

假设有 64 个专家，但每个 token 只选 2 个专家。

```text
总专家参数 = 64 个专家
单 token 激活 = 2 个专家
```

所以：

- 总容量很大。
- 单 token 计算接近激活专家数量。

这就是 MoE 的核心吸引力。

## 12.2 MoE 难在哪里

MoE 的难点也来自 Router：

- 如果路由不均衡，少数专家过载。
- 分布式训练需要跨卡通信。
- 小 batch 推理时专家利用率可能不好。
- 监控和调优比 Dense 复杂。

所以 MoE 不是“无脑更好”，它是容量、成本和工程复杂度之间的交换。

---

# 13. Base、Instruct、Chat、Reasoning：这不是架构差异

很多模型名里会带 Base、Instruct、Chat、Reasoning。它们更多表示训练阶段和行为特征，不是底层架构类型。

| 类型 | 怎么理解 | 适合 |
|---|---|---|
| Base | 预训练后的“续写器”，能力强但不一定听指令 | 继续训练、研究 |
| Instruct | 学过指令数据，知道怎么完成任务 | 问答、摘要、抽取 |
| Chat | 针对多轮对话和角色格式优化 | 助手、客服、Agent |
| Reasoning | 强化复杂推理和长链路解题 | 数学、代码、规划 |

一个常见误区：

> Base 模型不是“低配 Chat 模型”，而是行为目标不同。

Base 模型可能知识很强，但没有被训练成一个稳定听话的助手。

---

# 14. Embedding、Reranker、多模态模型不是同一类东西

工程上还要区分“模型用途”。

## 14.1 生成模型

输入文本，输出文本。用于：

- 对话。
- 摘要。
- 抽取。
- 写代码。
- 工具调用。

## 14.2 Embedding 模型

输入一段文本，输出一个向量。

用于：

- 语义检索。
- 聚类。
- 相似度计算。
- RAG 召回。

它不负责生成答案。

## 14.3 Reranker

输入 query 和 document，输出相关性分数。

用于：

- 对召回结果精排。
- 提升 RAG 证据质量。

Reranker 通常比 embedding 更准，但更慢。

## 14.4 多模态模型

能处理图像、音频、视频或文档截图等输入。

用于：

- OCR + 理解。
- 图表问答。
- UI 自动化。
- 多模态 Agent。

---

# 15. 工程含义：选模型要从任务倒推

模型选型先问任务，不要先问“哪个模型最强”。

## 15.1 一个简单决策树

```text
任务需要生成长文本吗？
  ├─ 否 -> 分类/打分/检索？
  │       ├─ 分类：Encoder 或小型 Instruct
  │       ├─ 检索：Embedding
  │       └─ 精排：Reranker
  │
  └─ 是 -> 是否输入输出转换很明确？
          ├─ 是：Encoder-Decoder 或 Decoder-only 都可评估
          └─ 否：优先 Chat/Instruct Decoder-only
```

## 15.2 常见场景

| 场景 | 推荐思路 |
|---|---|
| 意图分类 | 小模型或 Encoder 分类模型 |
| 企业 RAG | Embedding + Reranker + Chat 模型 |
| 客服 Agent | 稳定 Chat 模型 + 工具调用能力 |
| 代码助手 | 代码能力强的 Decoder-only 模型 |
| 批量摘要 | 成本合适的生成模型，低温采样 |
| 数学推理 | Reasoning 模型或推理后训练模型 |
| 私有化部署 | 开源 Dense/MoE，结合硬件和许可证 |

## 15.3 不要只看这些

不要只看：

- 参数量。
- 排行榜排名。
- 上下文长度。
- 开源/闭源标签。

还要看：

- 业务数据表现。
- 延迟和成本。
- 中文/领域术语覆盖。
- JSON 和工具调用稳定性。
- 安全和合规。
- 是否容易部署和观测。

---

# 16. 常见误区

## 16.1 “BERT 过时了”

不对。BERT 类模型不再是通用聊天主流，但在分类、抽取、rerank、安全审核等任务上仍有价值。

## 16.2 “Decoder-only 理解能力一定差”

不对。大规模 Decoder-only 可以把理解任务转成生成任务，并通过海量数据学到很强的表示能力。只是它的机制不是“天然双向阅读”。

## 16.3 “MoE 参数越大一定越强”

不一定。MoE 还受专家质量、路由、训练数据、激活参数、通信效率和部署条件影响。

## 16.4 “长上下文模型就适合所有 RAG”

不一定。长上下文能放更多材料，但检索、重排、引用、事实校验仍然重要。

## 16.5 “Chat 模型能替代所有小模型”

不一定。分类、路由、审核、embedding、rerank 往往用专用小模型更便宜、更稳定。

---

# 17. 面试 Q&A

## Q1：BERT、GPT、T5 的本质区别是什么？

**答**：BERT 是 Encoder-only，双向看完整输入，常用 MLM，适合理解和打分；GPT 是 Decoder-only，只看历史 token，使用 next token prediction，适合自回归生成；T5 是 Encoder-Decoder，Encoder 读输入，Decoder 写输出，适合翻译、摘要、改写等输入输出转换任务。

## Q2：为什么 Decoder-only 成为通用 LLM 主流？

**答**：因为 next token prediction 能利用几乎所有文本和代码数据；生成接口能统一问答、分类、抽取、代码和工具调用；KV Cache 让自回归推理可工程化；SFT、DPO/RLHF、工具调用和 Agent 生态也主要围绕 Decoder-only 构建。

## Q3：为什么 BERT 不能直接做 ChatGPT？

**答**：BERT 的双向注意力和 MLM 目标让它擅长补空和理解完整输入，但聊天模型要从左到右连续生成，不能看未来 token。二者训练目标和推理形式不一致。

## Q4：Encoder-Decoder 的 Cross-Attention 有什么作用？

**答**：Cross-Attention 让 Decoder 在生成每个输出 token 时读取 Encoder 的输入表示。Query 来自 Decoder，Key/Value 来自 Encoder，所以它非常适合翻译、摘要这类“读一个输入，再写一个输出”的任务。

## Q5：Dense 和 MoE 的核心区别是什么？

**答**：Dense 模型每个 token 都走同一套参数；MoE 模型通过 Router 为每个 token 选择少量专家。MoE 可以在较低激活计算下扩大总参数容量，但带来路由、负载均衡、通信和部署复杂度。

## Q6：Base、Instruct、Chat、Reasoning 有什么区别？

**答**：Base 主要是预训练模型，像续写器；Instruct 学过指令数据，能完成任务；Chat 针对多轮对话和角色格式优化；Reasoning 强化复杂推理和长链路解题。它们通常是训练阶段和行为差异，不是三种完全不同架构。

## Q7：RAG 系统应该只选一个最强 Chat 模型吗？

**答**：通常不应该。RAG 往往需要 embedding 做召回、reranker 做精排、Chat 模型做最终回答，还要有引用、校验和评估。只换一个更强 Chat 模型不能解决所有检索和事实性问题。

## Q8：同样是 7B 模型，为什么效果差很多？

**答**：参数量只是一个维度。tokenizer、训练数据质量、训练 token 数、架构细节、上下文长度、后训练数据、对齐方法、推理参数都会影响最终效果。

## Q9：什么时候用小模型？

**答**：分类、路由、审核、简单抽取、embedding、rerank 等任务常用小模型更划算。Agent 系统里可以用小模型处理简单步骤，把大模型留给复杂推理和生成。

## Q10：如何判断一个模型是否适合企业 Agent？

**答**：看中文和领域术语能力、工具调用稳定性、JSON/schema 遵循、长上下文利用率、RAG 忠实性、延迟、成本、私有化和合规要求。不要只看通用排行榜。

---

# 18. 记忆口诀

```text
BERT：完整读，适合理解。
GPT：看过去，逐步生成。
T5：先读输入，再写输出。
Decoder-only 主流：数据统一、接口统一、KV Cache 友好。
Dense：所有 token 走同一套参数。
MoE：token 选专家，总参数大，激活参数少。
Base/Instruct/Chat：不是架构名，是训练和行为阶段。
```

---

# 19. 相关链接

- [[README|LLM 基础目录]]
- [[01-Transformer|Transformer]]
- [[02-Tokenizer-Embedding|Tokenizer 与 Embedding]]
- [[04-Training-Objectives|训练目标与对齐]]
- [[06-Context-Window-KV-Cache|上下文窗口与 KV Cache]]
- [[08-Inference-Optimization|推理优化]]
- [[09-Scaling-Laws|Scaling Laws]]
- [[10-Model-Evaluation|模型评估]]

---

> **参考**
> - Vaswani et al. ["Attention Is All You Need"](https://arxiv.org/abs/1706.03762) (2017)
> - Devlin et al. ["BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding"](https://arxiv.org/abs/1810.04805) (2018)
> - Brown et al. ["Language Models are Few-Shot Learners"](https://arxiv.org/abs/2005.14165) (2020)
> - Raffel et al. ["Exploring the Limits of Transfer Learning with a Unified Text-to-Text Transformer"](https://arxiv.org/abs/1910.10683) (2020)
> - Touvron et al. ["Llama 2: Open Foundation and Fine-Tuned Chat Models"](https://arxiv.org/abs/2307.09288) (2023)
> - Qwen Team. ["Qwen2.5 Technical Report"](https://arxiv.org/abs/2412.15115) (2024)
> - DeepSeek-AI. ["DeepSeek-V3 Technical Report"](https://arxiv.org/abs/2412.19437) (2024)
