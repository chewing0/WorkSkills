---
tags:
  - LLM
  - Context-Window
  - KV-Cache
created: 2026-05-21
description: 上下文窗口与 KV Cache 详解，从模型工作记忆出发讲清 token 预算、prefill、decode、KV Cache 显存、长上下文成本和 Agent 场景管理
---

> **核心考点**：上下文窗口是模型一次推理能使用的工作记忆，不是无限记忆。KV Cache 缓存的是每层历史 token 的 Key/Value，不是缓存最终答案。长 prompt 主要拖慢 prefill，长输出主要拖慢 decode，高并发和长上下文主要吃 KV Cache 显存。

---

# 1. 核心问题：上下文窗口为什么是模型的工作记忆

上下文窗口（context window）是模型一次请求中能处理的最大 token 数。

它包括：

```text
system prompt
+ chat template 特殊 token
+ 历史消息
+ 当前用户输入
+ RAG 文档
+ 工具 schema
+ 工具返回结果
+ 预留输出 token
```

所以不要把上下文窗口理解成“用户问题最大长度”。它是整次请求的总预算。

例子：

```text
模型窗口: 32K tokens
system prompt: 1K
历史对话: 6K
RAG 文档: 18K
工具 schema: 2K
用户问题: 1K
预留输出: 4K
总计: 32K
```

一旦超过窗口，要么报错，要么截断，要么压缩。

---

# 2. 上下文窗口不是无限记忆

窗口变大只表示模型能接收更多 token，不代表一定能用好。

长上下文有三类问题：

| 问题 | 含义 |
|---|---|
| 放得进 | 输入和输出是否超过窗口 |
| 算得动 | attention 和 KV Cache 是否承受得住 |
| 用得好 | 模型是否真的能找到并利用关键信息 |

很多应用失败不是因为“放不进去”，而是因为：

- RAG 文档太多，证据被稀释。
- 关键信息埋在中间。
- 历史对话挤掉当前任务。
- 工具结果过长，模型抓不住重点。
- 输出 token 没预留够，答案被截断。

---

# 3. 为什么长上下文贵

Transformer Attention 和序列长度有关。

训练或 prefill 时，完整序列内部要互相计算注意力，核心成本近似：

$$
O(n^2)
$$

推理时还要保存每层历史 token 的 K/V：

```text
上下文越长 -> KV Cache 越大
并发越高 -> KV Cache 乘以请求数
```

长上下文贵在：

- prefill 计算更多。
- attention 临时显存更多。
- KV Cache 显存更多。
- decode 读取历史 K/V 更重。
- 调度和批处理更难。

---

# 4. 基础机制：Prefill 与 Decode 的两段时间线

LLM 推理分两段：

```text
完整 prompt
  │
  ├─► prefill：一次处理所有输入 token，建立 KV Cache
  │
  └─► decode：每次生成一个新 token，并追加到 KV Cache
```

| 阶段 | 输入 | 主要特点 | 常见瓶颈 |
|---|---|---|---|
| Prefill | 完整 prompt | 并行度高，一次算很多 token | 算力、attention 计算 |
| Decode | 每次一个新 token | 串行生成，步步依赖上一步 | 显存带宽、KV Cache 读取 |

## 4.1 为什么首 token 慢

首 token 前要完成 prefill：

```text
读完整 prompt -> 计算所有层 hidden states -> 建立 KV Cache -> 才能出第一个 token
```

prompt 越长，首 token 越慢。

## 4.2 为什么后续 token 一个个出

第 $t+1$ 个 token 依赖第 $t$ 个 token，所以 decode 天然串行：

```text
生成 token_1 后，才能生成 token_2
生成 token_2 后，才能生成 token_3
```

这就是为什么流式输出看起来是“一个词一个词吐”。

---

# 5. KV Cache 缓存的到底是什么

每层 Attention 都会计算：

$$
Q = XW^Q,\quad K = XW^K,\quad V = XW^V
$$

生成第一个新 token 后，历史 token 的 K/V 不会再变。下一步只需要：

```text
新 token -> 计算新的 Q/K/V
历史 token -> 复用缓存的 K/V
新 Q attend 到 历史 K/V + 新 K/V
```

KV Cache 缓存的是：

- 每一层。
- 每个历史 token。
- 每个 KV head。
- 对应的 Key 和 Value 向量。

它不是缓存：

- 最终答案。
- attention 权重。
- logits。
- hidden state 的全部中间计算。

---

# 6. 没有 KV Cache 会怎样

假设 prompt 有 1000 个 token，要生成 100 个 token。

没有 KV Cache：

```text
第 1 步：算 1000 tokens
第 2 步：重新算 1001 tokens
第 3 步：重新算 1002 tokens
...
```

历史 token 会被反复计算，非常浪费。

有 KV Cache：

```text
prefill：算 1000 tokens，缓存 K/V
decode 第 1 步：只算 1 个新 token
decode 第 2 步：只算 1 个新 token
...
```

所以 KV Cache 是自回归推理的核心加速手段。

---

# 7. KV Cache 显存怎么算

近似公式：

$$
\text{KV bytes}
= B \times L \times 2 \times N_{layer} \times N_{kv\_head} \times d_{head} \times bytes
$$

含义：

- $B$：batch size 或并发序列数。
- $L$：每个序列缓存长度。
- $2$：K 和 V 两份。
- $N_{layer}$：层数。
- $N_{kv\_head}$：KV head 数。
- $d_{head}$：每个 head 维度。
- $bytes$：每个元素字节数，FP16/BF16 通常是 2。

这个公式说明：

```text
上下文长度翻倍 -> KV Cache 近似翻倍
并发翻倍 -> KV Cache 近似翻倍
层数越多 -> KV Cache 越大
KV head 越多 -> KV Cache 越大
```

---

# 8. 为什么 GQA/MQA 能省 KV Cache

普通 MHA 中，Query head 和 KV head 数量通常相同。

GQA/MQA 的思路是减少 KV head：

| 机制 | KV Cache |
|---|---|
| MHA | 每个 Q head 都有自己的 K/V，cache 大 |
| GQA | 多个 Q head 共享一组 K/V，cache 变小 |
| MQA | 所有 Q head 共享一组 K/V，cache 更小 |

它们的核心不是“让模型更聪明”，而是在效果和推理成本之间折中。

---

# 9. 同族概念分组：降低 KV Cache 成本的方法

## 9.1 PagedAttention

不同请求长度不同，KV Cache 直接连续分配会产生碎片。

PagedAttention 借鉴操作系统分页思想，把 KV Cache 分成 block 管理：

```text
请求 A: block 1 -> block 5 -> block 9
请求 B: block 2 -> block 3
```

它解决的是：

- 显存碎片。
- 动态长度请求。
- 高并发调度。

它不是新的 attention 数学机制。

## 9.2 KV Cache 量化

KV Cache 可以从 FP16/BF16 降到更低精度，减少显存和带宽。

收益：

- 更长上下文。
- 更高并发。
- 更低显存压力。

风险：

- 可能影响长上下文质量。
- 对数学、代码、精细引用任务要评估。

---

# 10. Lost in the Middle

很多模型对开头和结尾更敏感，对中间信息利用较差，这叫 lost in the middle。

原因包括：

- 训练数据中关键信息常出现在开头或结尾。
- 长上下文 attention 分布复杂。
- RAG 文档太多，干扰信息稀释证据。
- 模型未充分训练长距离检索任务。

缓解策略：

- 把用户问题和关键约束放近。
- RAG 结果按相关性排序。
- 关键证据加编号和标题。
- 不要塞太多低相关片段。
- 长文档先摘要或分块问答。

---

# 11. 工程含义：上下文管理策略

## 11.1 滑动窗口

只保留最近若干轮。

优点：简单。
缺点：早期约束和长期偏好会丢。

## 11.2 摘要记忆

把历史对话压成摘要。

优点：省 token。
缺点：摘要可能丢细节或引入错误。

## 11.3 检索式记忆

把历史或知识放入向量库，需要时检索。

优点：适合长期记忆。
缺点：检索失败时模型看不到相关信息。

## 11.4 分层上下文

推荐把上下文分层：

```text
系统规则
用户偏好
当前任务
检索证据
工具结果
输出格式
```

分层比简单拼接更可靠。

---

# 12. Agent 场景怎么管理上下文

Agent 上下文常包含：

- 用户目标。
- 当前计划。
- 已完成步骤。
- 工具调用记录。
- 工具返回结果。
- 错误日志。
- 中间文件摘要。
- 用户约束。

常见问题：

- 工具结果太长，挤掉任务目标。
- 错误日志太长，浪费窗口。
- 历史计划过期，模型继续执行旧计划。
- RAG 文档太多，模型忽略真正证据。

建议：

- 每轮保留“当前任务状态”。
- 工具结果先结构化摘要。
- 错误日志只保留关键错误栈。
- 长文件只放相关片段。
- 对任务状态做可恢复记录。

---

# 13. 常见误区

## 13.1 上下文窗口就是长期记忆

不对。上下文窗口只是本次推理可见的工作区。窗口外的信息模型看不到，除非被摘要、检索或重新放入 prompt。

## 13.2 KV Cache 缓存的是最终答案

不对。KV Cache 缓存的是每层历史 token 的 Key 和 Value，用来加速后续 decode，不是缓存自然语言答案。

## 13.3 RAG 文档越多越好

不对。无关文档会挤占窗口、增加 prefill 成本，还可能干扰模型注意力和引用判断。

## 13.4 长上下文能自动解决 Agent 记忆

不对。Agent 还需要状态摘要、工具结果裁剪、历史计划管理和失败日志压缩。单纯扩大窗口只会提高成本，不保证更可靠。

---

# 14. 面试 Q&A

## Q1：为什么长 prompt 首 token 慢？

**答**：首 token 前需要 prefill，模型要处理完整 prompt 并建立每层 KV Cache。prompt 越长，prefill 计算越多，所以 TTFT 越高。

## Q2：KV Cache 为什么能加速推理？

**答**：历史 token 的 Key/Value 在生成过程中不会改变。缓存后，每步 decode 只需计算新 token 的 Q/K/V，再用新 Q attend 到历史 K/V，避免重复计算历史序列。

## Q3：KV Cache 缓存的是答案吗？

**答**：不是。它缓存的是每层历史 token 的 Key 和 Value 向量，不是最终答案、logits 或 attention 权重。

## Q4：上下文窗口越大越好吗？

**答**：不一定。大窗口能放更多 token，但会增加 prefill、KV Cache 和调度成本，也不保证模型能稳定利用远距离信息。

## Q5：RAG 文档塞得越多越好吗？

**答**：不是。无关文档会增加成本、稀释注意力并诱发幻觉。应检索高相关片段，重排、去重，并要求引用。

## Q6：GQA/MQA 为什么能省显存？

**答**：它们减少 KV head 数量，让多个 Query head 共享 K/V，因此 KV Cache 更小，长上下文和高并发更省显存。

---

# 15. 相关链接

- [[README|LLM 基础目录]]
- [[01-Transformer|Transformer]]
- [[02-Tokenizer-Embedding|Tokenizer 与 Embedding]]
- [[07-Length-Extrapolation|长度外推]]
- [[08-Inference-Optimization|推理优化]]
