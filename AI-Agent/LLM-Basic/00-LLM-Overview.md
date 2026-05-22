---
tags:
  - LLM
  - AI-Agent
  - 基础概念
created: 2026-05-21
description: 大语言模型基础总览，从 token、logits、训练目标、推理流程到能力边界
---

# 1. 核心问题：LLM 到底在做什么

大语言模型（Large Language Model, LLM）最小的工作闭环是：

> 给定当前上下文，预测下一个 token 的概率分布，再用生成策略选出一个 token。

给定上下文：

```text
今天天气很好，我想去
```

模型不是直接“写完整答案”，而是先输出词表中每个候选 token 的分数：

```text
公园: 0.18
跑步: 0.08
散步: 0.07
...
```

解码策略选出一个 token 后，把它拼回上下文，模型再预测下一步：

```text
今天天气很好，我想去 公园
今天天气很好，我想去 公园 散步
今天天气很好，我想去 公园 散步 。
```

这就是理解 LLM 的入口：

```text
文本 -> token -> 向量 -> Transformer -> logits -> 概率 -> 下一个 token
```

后面所有章节都围绕这条链路展开。

---

# 2. 基础机制：从文本到输出的最小闭环

完整链路可以拆成 7 步：

```text
原始文本
  │
  ├─► Tokenizer：切成 token id
  │
  ├─► Embedding：token id 变成向量
  │
  ├─► Transformer Blocks：让每个 token 汇聚上下文信息
  │
  ├─► LM Head：hidden state 投影到词表维度
  │
  ├─► Logits：每个候选 token 的未归一化分数
  │
  ├─► Softmax / Filtering：变成概率并筛选候选
  │
  └─► Decoding：选择下一个 token
```

这些概念不要分开背，要放在一条流水线里理解：

| 概念 | 在链路中的位置 | 一句话理解 |
|---|---|---|
| token | 输入和输出单位 | 模型读写的最小离散片段 |
| embedding | token 进入模型的入口 | 把离散 id 变成连续向量 |
| hidden state | Transformer 内部表示 | 每层对 token 和上下文的理解 |
| logits | 输出概率前的分数 | 每个候选 token 的原始打分 |
| generation strategy | logits 后处理 | 决定选哪个 token、何时停止 |
| context window | 模型一次能看的范围 | 当前推理的工作记忆 |

这一闭环解释了很多现象：

- 输出为什么可能不稳定：因为可以从概率分布中采样。
- token 为什么影响成本：输入输出都按 token 计算。
- 上下文为什么有限：Transformer 一次只能处理窗口内 token。
- 幻觉为什么会发生：模型生成高概率文本，不等于查询事实数据库。

---

# 3. 同族概念分组：LLM 基础目录怎么串起来

这个目录里的概念可以分成 6 组，而不是 10 多个孤立名词：

| 概念组 | 解决的问题 | 对应文档 |
|---|---|---|
| 文本表示 | 字符串如何进入模型 | [[02-Tokenizer-Embedding]] |
| 上下文建模 | token 之间如何互相影响 | [[01-Transformer]]、[[03-LLM-Variants]] |
| 行为塑造 | 模型如何从续写器变成助手 | [[04-Training-Objectives]] |
| 输出控制 | 概率分布如何变成可用答案 | [[05-Generation-Strategies]] |
| 推理成本 | 长上下文和在线服务为什么贵 | [[06-Context-Window-KV-Cache]]、[[07-Length-Extrapolation]]、[[08-Inference-Optimization]] |
| 模型选择与验证 | 为什么不能只看参数和榜单 | [[09-Scaling-Laws]]、[[10-Model-Evaluation]] |

学习时建议把每个新名词放回它所属的概念组：

- BPE、WordPiece、SentencePiece 都是在解决“如何把文本切成 token”。
- GPT、BERT、T5 都是在解决“模型能看见哪些上下文”。
- SFT、RLHF、DPO 都是在解决“模型行为如何更像可用助手”。
- Top-p、Temperature、Beam 都是在解决“如何从 logits 选择输出”。
- FlashAttention、PagedAttention、Continuous Batching 都是在解决“推理瓶颈在哪里”。

这样读起来会比按名词背轻很多。

---

# 4. 训练与推理：同一个模型的两种状态

## 4.1 训练阶段

训练时，模型看到大量文本，学习在给定上下文时预测真实下一个 token。

Decoder-only 预训练常用目标：

$$
\mathcal{L} = -\sum_t \log P(x_t \mid x_{<t})
$$

训练阶段关心：

- loss 是否下降。
- 数据分布是否合适。
- 模型是否学到语言、知识、代码和任务格式。
- 是否通过 SFT 和偏好对齐变成可用助手。

## 4.2 推理阶段

推理时，模型逐 token 生成：

```text
prompt -> token1 -> token2 -> token3 -> ...
```

推理阶段关心：

- 首 token 慢不慢，也就是 prefill 成本。
- 后续 token 快不快，也就是 decode 成本。
- 上下文窗口够不够。
- KV Cache 是否占满显存。
- 生成策略是否稳定、事实、结构化。

一句话：

> 训练决定模型学到了什么，推理决定如何把模型能力稳定、低成本地用出来。

---

# 5. 工程含义：LLM 只是应用系统的一层

真实 AI Agent 或 LLM 应用通常不是“用户直接连模型”。

```text
用户输入
  │
  ├─► Prompt / Messages
  ├─► RAG / Memory / Tools
  ├─► Model Inference
  ├─► Output Parser
  ├─► Guardrails / Safety
  ├─► Evaluation / Logging
  └─► UI / API Response
```

模型之外还需要：

- RAG：补充外部知识和引用来源。
- 工具调用：让模型获取实时信息或执行操作。
- 状态管理：维护多轮任务进度。
- 输出解析：保证 JSON、函数参数、表格等结构可用。
- 安全策略：限制越权、隐私、危险操作。
- 评估监控：持续发现幻觉、失败和成本问题。

所以入门 LLM 不只是学模型结构，还要知道模型在工程系统里处于什么位置。

---

# 6. 能力边界：LLM 为什么有用，也为什么会错

LLM 擅长：

- 文本理解、改写、摘要、分类。
- 代码生成和解释。
- 模式迁移、few-shot 学习。
- 对话式任务分解。
- 调用工具完成外部操作。

LLM 容易出错：

- 精确事实，尤其是新近或长尾事实。
- 长距离信息检索，尤其是上下文很长时。
- 严格数学计算。
- 多步骤状态一致性。
- 权限、安全、隐私边界模糊的任务。
- 没有工具时的实时信息查询。

原因不是“模型没用”，而是它的目标是生成高概率文本。它可以压缩知识和模式，但不天然具备数据库查询、形式化证明、权限审计和长期记忆。

---

# 7. 常见误区

## 7.1 LLM 是知识库

不准确。LLM 参数中压缩了大量训练分布中的模式和知识，但它不是可精确查询、可更新、可溯源的数据库。

## 7.2 logits 就是概率

不是。logits 是 softmax 前的未归一化分数。Temperature、Top-p、logit bias 等生成策略通常都会在 logits 到 token 之间生效。

## 7.3 上下文窗口等于长期记忆

不是。上下文窗口只是本次推理可见的工作区。窗口外的信息需要通过摘要、检索或外部存储重新放进来。

## 7.4 参数越大就越适合所有应用

不一定。业务还要看延迟、成本、上下文、工具调用、格式稳定、领域知识和失败风险。

---

# 8. 面试 Q&A

## Q1：LLM 的输出为什么每次可能不一样？

因为模型每一步输出的是 token 概率分布。如果使用 temperature、Top-p、Top-k 等采样策略，同一个概率分布可能采样出不同 token。使用 greedy 或低 temperature 可以提高确定性。

## Q2：token 和字、词有什么区别？

token 是 tokenizer 切分出的模型输入单位。英文中一个 token 可能是词、词根或子词；中文中可能是一个字、一个词或字节片段。token 数直接影响上下文长度、成本和延迟。

## Q3：为什么说 LLM 会幻觉？

因为 LLM 的训练目标是根据上下文生成高概率文本，不是查询真实数据库。缺少知识、上下文不足、提示诱导或采样随机性都可能导致模型生成看似合理但不真实的内容。

## Q4：LLM 和传统搜索引擎有什么不同？

搜索引擎检索已有网页或文档；LLM 根据参数和上下文生成文本。LLM 可以总结和改写，但不保证事实新鲜和可溯源。因此很多应用会结合 RAG，让模型基于检索结果回答。

## Q5：学 LLM 基础最重要的主线是什么？

最重要的是把链路串起来：文本先被切成 token，token 变成 embedding，Transformer 建模上下文，LM Head 输出 logits，生成策略选 token，推理系统控制上下文、缓存、延迟和成本。

---

# 9. 相关链接

- [[README|LLM 基础目录]]
- [[02-Tokenizer-Embedding|Tokenizer 与 Embedding]]
- [[01-Transformer|Transformer]]
- [[04-Training-Objectives|训练目标与对齐]]
- [[05-Generation-Strategies|生成策略]]
- [[06-Context-Window-KV-Cache|上下文窗口与 KV Cache]]
