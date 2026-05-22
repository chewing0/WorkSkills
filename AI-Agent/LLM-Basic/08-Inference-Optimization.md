---
tags:
  - LLM
  - Inference
  - Optimization
created: 2026-05-21
description: LLM 推理优化基础，覆盖 FlashAttention、PagedAttention、连续批处理、量化、投机解码和服务指标
---

# 1. 核心问题：推理优化为什么要先找瓶颈

LLM 推理优化最容易学成“技术名词清单”：FlashAttention、PagedAttention、量化、Speculative Decoding、Continuous Batching 都知道一点，但不知道什么时候该用。

更好的理解方式是先问：

> 当前慢在哪里？是输入太长，输出太长，显存不够，还是调度吞吐不够？

一次推理请求大致经历：

```text
请求进入
  │
  ├─► 拼接 prompt / messages / RAG 文档 / 工具结果
  │
  ├─► tokenizer 编码
  │
  ├─► prefill：一次性处理所有输入 token，生成首 token 所需状态
  │
  ├─► decode：一次生成一个输出 token
  │
  ├─► detokenize / 流式返回
  │
  └─► 业务后处理：JSON 校验、工具调用、重试、日志
```

优化也应围绕这条链路展开：

| 瓶颈 | 典型表现 | 主要原因 | 常见优化 |
|---|---|---|---|
| prefill 算力 | 首 token 很慢，长 prompt 更明显 | 长输入 attention 和大矩阵计算重 | FlashAttention、prompt 压缩、prefix cache、chunked prefill |
| decode 带宽 | 后续 token 慢，TPOT 高 | 每步都要读权重和 KV Cache，且串行生成 | Continuous batching、量化、Speculative decoding |
| KV Cache 显存 | 并发上不去，长上下文 OOM | 每层每个历史 token 都要存 K/V | GQA/MQA、PagedAttention、KV quantization |
| 调度吞吐 | GPU 利用率波动，请求排队 | 请求长度不一，普通 batch 浪费 | continuous batching、队列策略、限流和路由 |

一句话：

> LLM 推理优化不是“用了某个技术就快”，而是把请求生命周期中最贵的部分挪掉、压小、并行化或调度得更紧凑。

---

# 2. 服务指标：不要只看“快不快”

推理服务里常见指标如下：

| 指标 | 含义 | 用户感知 |
|---|---|---|
| TTFT | Time To First Token，首 token 延迟 | 用户多久看到开始回复 |
| TPOT | Time Per Output Token，每个输出 token 平均耗时 | 回复流式速度是否顺滑 |
| End-to-end Latency | 从请求到完整响应结束 | 整体等待时间 |
| Throughput | 单位时间处理 token 数或请求数 | 服务承载能力 |
| Concurrency | 同时服务多少请求 | 高峰期是否排队 |
| Memory | 权重、KV Cache、临时激活占用 | 能否放下模型和并发 |
| Cost | 每千 token 或每请求成本 | 产品能不能长期跑 |

这些指标之间经常冲突：

- 为了提高吞吐，服务端可能让请求在队列里等一小会儿再合批，TTFT 会变差。
- 为了降低显存，用 INT4 权重量化可能牺牲少量质量，也未必在所有硬件上显著加速。
- 为了降低 TPOT，使用 Speculative Decoding 需要额外草稿模型，系统复杂度上升。
- 为了减少幻觉，RAG 放入更多文档会增加 prefill 成本和 TTFT。

所以工程上要先明确目标：

| 场景 | 优先指标 | 典型策略 |
|---|---|---|
| 在线聊天 | TTFT、TPOT | 流式输出、prefix cache、合理上下文裁剪 |
| 批量总结 | Throughput、Cost | 大 batch、离线队列、低精度推理 |
| Agent | TTFT、稳定性、成本 | 控制工具结果长度、缓存系统提示词、限制循环 |
| 代码补全 | TPOT、低延迟 | 小模型、短上下文、prefix cache |
| 企业私有化 | Memory、Cost | 量化、GQA/MQA、并发控制、模型路由 |

---

# 3. 基础机制：prefill 和 decode 为什么瓶颈不同

## 3.1 prefill：把输入一次性“读完”

prefill 阶段处理用户输入和上下文中的所有 token。

如果 prompt 长度是 8k token，模型需要对这 8k token 做前向计算，并为每一层生成历史 token 的 K/V。这个阶段的特点是：

- 输入 token 已知，可以并行计算。
- 长上下文下 attention 成本高。
- 大矩阵乘法占比较高，常常更偏 compute-bound。
- 首 token 必须等 prefill 完成后才能生成，所以 TTFT 受它影响很大。

这解释了为什么同一个模型在两种请求上体验差异很大：

```text
短 prompt + 长输出：
  首 token 来得快，但输出过程可能慢。

长 prompt + 短输出：
  首 token 等很久，但后续很快结束。
```

对 RAG 和 Agent 尤其重要：检索文档、工具返回、历史日志都属于输入 token，会直接拉高 prefill 成本。

## 3.2 decode：一次只能生成一个 token

decode 阶段每一步只生成一个新 token：

```text
已有 token: A B C
  └─► 生成 D

已有 token: A B C D
  └─► 生成 E
```

因为下一个 token 依赖前一个 token，所以单个请求内部很难把多个未来 token 完全并行化。decode 的特点是：

- 每步只有一个新 token，但要经过完整模型层。
- 每步都需要读取模型权重。
- 每步都要读取历史 token 的 KV Cache。
- 输出越长，decode 次数越多。

这也是为什么长文本生成贵：不是一次前向生成完整文章，而是几百、几千次小步前向。

## 3.3 为什么首 token 慢，后续 token 串行

可以把推理想成“先读题，再逐字作答”：

- prefill 是读题，题越长，读得越久。
- decode 是作答，每写一个字都要根据前文再判断下一个字。
- KV Cache 是草稿纸，记录已经读过的中间状态，避免每次重读全文。

这条直觉比背指标更重要。很多线上问题本质上都是：

- prompt 太长，所以 TTFT 高。
- 输出太长，所以端到端延迟高。
- 并发太大，所以 KV Cache 爆显存。
- 请求长度差异太大，所以 GPU 利用率低。

---

# 4. 同族概念分组：按瓶颈理解优化技术

从这里开始，不按技术名死背，而是把每个技术放回它解决的瓶颈：

| 技术 | 主要瓶颈 | 一句话边界 |
|---|---|---|
| FlashAttention | prefill attention IO | 优化 attention kernel，不改变数学结果 |
| PagedAttention | KV Cache 分配 | 管理缓存内存，不是新 attention |
| Continuous Batching | 调度吞吐 | 动态合批 decode step |
| 量化 | 权重、激活、KV 显存和带宽 | 省显存不一定等于加速 |
| Speculative Decoding | decode 串行 | 草稿模型先写，大模型验收 |
| Prefix Cache | 重复 prefill | 复用相同前缀的计算结果 |

## 4.1 FlashAttention：优化 IO，不改变 Attention 数学

普通 self-attention 的核心是：

$$
\text{Attention}(Q,K,V)=\text{softmax}\left(\frac{QK^T}{\sqrt{d}}\right)V
$$

直观上，它会产生一个大小接近 $n \times n$ 的注意力矩阵。序列越长，这个矩阵越大，显存读写压力越高。

FlashAttention 解决的问题不是“换一种 attention 公式”，而是：

> 用更聪明的分块计算方式，减少 GPU HBM 和 SRAM 之间的数据搬运，并避免完整 attention 矩阵落到显存。

它的关键点：

- 数学结果仍是精确 attention，不是近似算法。
- 主要优化显存占用和内存 IO。
- 对长序列 prefill 特别有价值。
- attention 理论复杂度仍是 $O(n^2)$，不是变成线性。

容易误解的点：

| 误解 | 正确认识 |
|---|---|
| FlashAttention 是一种新模型结构 | 它主要是 attention kernel 的实现优化 |
| 用了 FlashAttention 长上下文就免费 | 长度增加仍会带来计算和 KV Cache 成本 |
| FlashAttention 会改变模型输出 | 精确实现下数学结果等价，数值误差来自浮点实现 |

工程含义：

- 如果长 prompt 的 TTFT 很高，FlashAttention 往往能帮忙。
- 如果瓶颈是 decode 阶段读权重和 KV Cache，FlashAttention 不是唯一关键。
- 如果服务没有使用支持它的 kernel 或硬件，理论收益不会自动出现。

---

# 5. PagedAttention：管理 KV Cache，不是新的注意力机制

KV Cache 记录每层历史 token 的 Key 和 Value。它能避免 decode 时重复计算历史 token，但代价是显存占用随上下文长度、层数、并发数增长。

问题在于：线上请求长度高度不一致。

```text
请求 A：输入 2k，输出 50
请求 B：输入 12k，输出 800
请求 C：输入 500，输出 20
请求 D：输入 8k，输出 2k
```

如果为每个请求预留一大块连续 KV Cache，容易出现：

- 预留太多，浪费显存。
- 请求结束时间不同，产生碎片。
- 长短请求混在一起，调度困难。

PagedAttention 借鉴操作系统分页思想，把 KV Cache 拆成固定大小的 block 管理：

```text
逻辑 token 序列:
  token 1 token 2 token 3 ... token N

物理 KV blocks:
  block 7 -> block 2 -> block 19 -> ...
```

它解决的是 KV Cache 的分配、复用和碎片问题。

重点：

- PagedAttention 不是改变 attention 数学公式。
- 它让不同请求的 KV Cache 更容易动态增长和释放。
- 它提升高并发、变长请求下的显存利用率。
- vLLM 高吞吐的关键基础之一就是这种 KV Cache 管理方式。

和 [[06-Context-Window-KV-Cache]] 的关系：

- 06 解释 KV Cache 是什么。
- 这里解释为什么 KV Cache 管理会变成推理系统瓶颈。

---

# 6. Continuous Batching：LLM 不能像普通 HTTP 那样简单 batch

普通批处理常常假设：

```text
收集一批请求 -> 一起计算 -> 一起返回
```

但 LLM 请求有两个麻烦：

1. 输入长度不同，prefill 时间不同。
2. 输出长度不同，decode 步数不同。

如果等一批请求全部完成才开始下一批，会产生大量空转：

```text
请求 A：生成 20 token，早早结束
请求 B：生成 800 token，还在继续
请求 C：排队等待，但 GPU 某些位置已经空了
```

Continuous Batching 的思想是：

> 在每个 decode step 动态维护 batch，完成的请求移出，新请求加入。

这让 GPU 更接近持续满载。

它解决的问题：

- 提高吞吐。
- 降低因为长短请求混合带来的浪费。
- 更适合在线多用户并发。

它带来的复杂度：

- 调度器要管理不同请求的状态。
- KV Cache 要支持动态分配和释放。
- 新请求加入时可能需要先做 prefill，prefill 和 decode 要协调。
- 队列策略会影响公平性和 TTFT。

面试里可以这样答：

> 普通 batching 适合形状相近、一起开始一起结束的任务；LLM decode 是逐 token 串行，且每个请求长度不同，所以需要 continuous batching 在 step 级别动态合批，尽量减少 GPU 空转。

---

# 7. 量化：省显存和加速不是一回事

量化的核心是用更低精度表示模型计算中的数据。

常见类型：

| 类型 | 量化对象 | 主要收益 | 风险 |
|---|---|---|---|
| 权重量化 | 模型参数 | 降低模型显存，减少权重读取带宽 | 精度下降、硬件不一定加速 |
| 激活量化 | 中间激活 | 降低计算和显存压力 | 校准复杂，数值稳定性风险 |
| KV Cache 量化 | 历史 K/V | 降低长上下文和高并发显存 | 长文本质量、注意力精度受影响 |

常见精度：

- FP16/BF16：主流推理基线。
- FP8：新硬件上常见，适合高吞吐推理。
- INT8：相对稳健，常用于权重或矩阵乘。
- INT4：显存节省明显，但质量和 kernel 支持更敏感。

需要分清两件事：

## 7.1 量化能省显存

例如 FP16 权重每个参数约 2 bytes，INT4 权重约 0.5 bytes。模型权重显存可以明显下降。

这意味着：

- 单卡能放下更大模型。
- 同一模型能留出更多空间给 KV Cache。
- 私有化部署成本下降。

## 7.2 量化不一定自动加速

加速取决于硬件和 kernel 是否真的高效支持低精度计算。

可能出现：

- 权重变小了，但计算时要反量化，速度提升有限。
- INT4 显存省了，但某些 GPU 上低精度 kernel 不够快。
- batch 很小时，调度和内存访问才是瓶颈。
- batch 很大时，计算吞吐和带宽瓶颈又不同。

所以工程判断应是：

> 量化先看显存收益，再看目标硬件上的实测延迟和质量损失。

---

# 8. Speculative Decoding：草稿模型先写，大模型验收

自回归 decode 之所以慢，是因为目标模型一次只能确认一个 token。Speculative Decoding 试图减少目标模型调用次数。

基本流程：

```text
1. 小模型快速生成多个候选 token：
   A B C D

2. 大模型一次性验证这些 token 是否可以接受。

3. 被接受的 token 直接输出。

4. 从第一个不接受的位置继续生成。
```

直觉像是：

> 草稿模型先写一段，大模型不是逐字重写，而是批量验收。

收益来自：

- 小模型便宜，可以快速提出多个 token。
- 大模型可以并行验证一段候选。
- 如果候选接受率高，就减少了大模型 decode 步数。

限制也很明显：

- 草稿模型和目标模型分布越接近，接受率越高。
- 高创造性采样时，候选更难被接受。
- 复杂系统需要维护两个模型或额外草稿头。
- 如果草稿质量低，额外开销可能抵消收益。
- 对短输出请求收益有限，因为还没摊薄启动成本。

适合场景：

- 长输出。
- 低温度、较确定的生成。
- 目标模型很大，草稿模型便宜。
- 服务端已经具备复杂调度能力。

---

# 9. Prefix Cache：重复输入不要反复算

很多线上请求共享前缀：

- 同一个 system prompt。
- 同一套工具说明。
- 同一段角色设定。
- 同一个文档模板。
- 代码补全里的文件前缀。

Prefix Cache 的思想是缓存共享前缀的 prefill 结果，后续请求复用。

```text
system prompt + 工具说明 + 用户问题 A
system prompt + 工具说明 + 用户问题 B
system prompt + 工具说明 + 用户问题 C
```

前半部分相同，就不应每次从零 prefill。

工程注意：

- 前缀必须 token 级完全一致，空格、换行、工具列表顺序都可能影响命中。
- 缓存占用显存或内存，需要淘汰策略。
- RAG 文档如果每次不同，命中率会下降。
- Agent 里固定工具说明适合缓存，动态工具结果通常不适合长期缓存。

Prefix Cache 主要改善 TTFT 和 prefill 成本，对长输出 decode 的 TPOT 影响较小。

---

# 10. 并行与路由：不是所有请求都该进同一个模型

当单卡放不下或吞吐不够时，会用并行策略：

| 策略 | 思路 | 适合问题 |
|---|---|---|
| Tensor Parallel | 把单层矩阵切到多张卡 | 大模型单卡放不下 |
| Pipeline Parallel | 把不同层放到不同卡 | 超大模型训练或推理 |
| Data Parallel / Replica | 多份模型副本处理不同请求 | 提高请求吞吐 |
| Expert Parallel | MoE 专家分布到多卡 | MoE 模型 |

但并行不是免费：

- 卡间通信会增加延迟。
- 小 batch 下通信开销可能很明显。
- 多卡部署提高成本和故障面。
- pipeline 需要足够 batch 才能填满流水线。

另一个常见优化是模型路由：

```text
简单分类 / 格式转换 -> 小模型
复杂推理 / 高风险回答 -> 大模型
代码或数学任务 -> 专用模型
```

这和 [[09-Scaling-Laws]] 里的启发一致：不是所有任务都需要最大模型。好的系统会用路由、缓存、检索和小模型，把大模型调用留给真正需要的地方。

---

# 11. 工程含义：诊断表

| 现象 | 更可能的瓶颈 | 排查方向 | 优先策略 |
|---|---|---|---|
| 首 token 很慢 | prefill | prompt 长度、RAG 文档、系统提示词 | 裁剪上下文、prefix cache、FlashAttention |
| 输出流很慢 | decode | TPOT、batch、权重读取、KV 读取 | continuous batching、量化、Speculative decoding |
| 并发稍高就 OOM | KV Cache | max context、batch、输出长度 | GQA/MQA、KV quantization、PagedAttention |
| GPU 利用率忽高忽低 | 调度 | 请求长短差异、队列策略 | continuous batching、分队列调度 |
| 成本高但质量没提升 | 模型选择 | 大模型是否过度使用 | 模型路由、小模型、缓存 |
| JSON 经常失败 | 生成策略 | 解码约束、schema、重试 | 约束解码、function calling、后校验 |

一个实用排查顺序：

```text
1. 看请求 token 分布：输入多长，输出多长。
2. 分开看 TTFT 和 TPOT，不只看总延迟。
3. 看 GPU 显存：权重、KV Cache、临时内存各占多少。
4. 看 batch 和队列：GPU 是否持续有活干。
5. 再决定用 FlashAttention、PagedAttention、量化还是投机解码。
```

---

# 12. 常见误区

## 12.1 “FlashAttention 让 attention 复杂度变线性”

不对。FlashAttention 优化 IO 和显存访问，attention 计算本身仍和序列长度平方相关。

## 12.2 “PagedAttention 是一种更强的 attention”

不对。它主要是 KV Cache 的分页式内存管理，不改变模型注意力公式。

## 12.3 “量化一定会加速”

不一定。量化一定可能省显存，但是否加速取决于硬件、kernel、batch size 和瓶颈类型。

## 12.4 “吞吐越高，用户体验越好”

不一定。吞吐高可能来自排队合批，但用户更关心 TTFT 和流式速度。线上服务要同时看吞吐和延迟分位数。

## 12.5 “上下文窗口越大，服务就越强”

长上下文会增加 prefill 和 KV Cache 成本。真正有效的系统还需要上下文选择、压缩、RAG 和引用机制。

---

# 13. 面试 Q&A

## Q1：LLM 推理为什么分 prefill 和 decode？

prefill 处理全部输入 token，生成每层历史 token 的 KV Cache，并得到首 token 所需状态。decode 在此基础上一次生成一个新 token，每一步都依赖上一步输出。前者更适合并行，常影响 TTFT；后者逐 token 串行，常影响 TPOT 和长输出成本。

## Q2：FlashAttention 解决什么问题？

它通过分块和 IO-aware kernel 减少 attention 计算中的显存读写，避免完整注意力矩阵落显存。它不改变 attention 数学结果，也不会把 $O(n^2)$ attention 变成线性复杂度。

## Q3：PagedAttention 和 FlashAttention 有什么区别？

FlashAttention 优化 attention kernel 的计算和 IO；PagedAttention 优化 KV Cache 的内存分配和碎片管理。前者主要改善 attention 计算效率，后者主要改善高并发、变长请求下的显存利用率。

## Q4：为什么 LLM 服务需要 Continuous Batching？

因为 LLM 请求输入和输出长度差异很大，普通 batch 会因为短请求早结束、长请求拖住整批而浪费 GPU。Continuous Batching 在 decode step 级别动态加入和移除请求，让 GPU 更持续地工作。

## Q5：量化如何影响成本和质量？

权重量化可以明显降低模型显存，KV Cache 量化可以提高长上下文和高并发能力。但低精度会带来数值误差，可能影响困惑度、事实性、格式稳定性或长上下文注意力。是否加速要看硬件和 kernel。

## Q6：Speculative Decoding 为什么能加速？

它让草稿模型先生成多个候选 token，再由目标大模型批量验证。如果候选接受率高，目标模型就不用一步一步生成所有 token，从而降低 decode 步数。它适合长输出、低温度、草稿模型和目标模型分布接近的场景。

## Q7：线上优化应该先看哪个指标？

先把总延迟拆成 TTFT 和 TPOT。TTFT 高通常先查输入长度、prefill 和 prefix cache；TPOT 高通常查 decode、batch、量化和 KV Cache；并发上不去则重点查显存和调度。

---

# 14. 记忆口诀

```text
首字慢，看 prefill；
续字慢，看 decode；
并发低，看 KV；
吞吐差，看 batch；
显存紧，先量化；
重复多，用 prefix；
长输出，想 speculative。
```

---

# 15. 相关链接

- [[05-Generation-Strategies]]
- [[06-Context-Window-KV-Cache]]
- [[07-Length-Extrapolation]]
- [[09-Scaling-Laws]]
- [[10-Model-Evaluation]]

---

# 16. 参考资料

- FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness
- FlashAttention-2: Faster Attention with Better Parallelism and Work Partitioning
- vLLM: Easy, Fast, and Cheap LLM Serving with PagedAttention
- Fast Inference from Transformers via Speculative Decoding
- TensorRT-LLM, vLLM, SGLang, TGI 等推理框架文档
