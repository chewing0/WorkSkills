---
tags:
  - LLM
  - AI-Agent
  - 学习路线
created: 2026-05-21
description: LLM 基础知识目录，覆盖模型结构、训练目标、生成策略、上下文、推理优化与评估
---

# LLM 基础

> 这个目录用于梳理 AI Agent 开发者必须理解的大语言模型基础。目标不是做论文综述，而是把“模型为什么这样工作、工程上怎么用、面试怎么回答”串起来。

---

# 1. 学习顺序

完全零基础建议按下面顺序读：

```text
00 -> 02 -> 01 -> 03 -> 04 -> 05 -> 06 -> 07 -> 08 -> 09 -> 10
```

原因是：先知道文本如何变成 token，再进入 Transformer；先理解模型如何生成，再理解上下文、推理优化、规模规律和评估。

| 顺序 | 文档 | 应该带走的核心问题 |
|---:|---|---|
| 0 | [[00-LLM-Overview|LLM 总览]] | LLM 如何从文本得到 logits，再逐 token 生成回答？ |
| 1 | [[02-Tokenizer-Embedding|Tokenizer 与 Embedding]] | 字符串如何变成 token id 和向量，为什么 token 数影响成本？ |
| 2 | [[01-Transformer|Transformer]] | Transformer 如何让每个 token 汇聚上下文信息？ |
| 3 | [[03-LLM-Variants|主流模型架构差异]] | Encoder、Decoder、Encoder-Decoder 的“看见方式”有什么本质区别？ |
| 4 | [[04-Training-Objectives|训练目标与对齐]] | 预训练、SFT、偏好对齐分别在修什么问题？ |
| 5 | [[05-Generation-Strategies|生成策略]] | 模型给出概率分布后，解码策略如何把概率变成可用输出？ |
| 6 | [[06-Context-Window-KV-Cache|上下文窗口与 KV Cache]] | 上下文窗口为什么是工作记忆，KV Cache 为什么能加速推理？ |
| 7 | [[07-Length-Extrapolation|长度外推]] | 长上下文为什么不只是“把窗口调大”？ |
| 8 | [[08-Inference-Optimization|推理优化]] | 推理优化应该先找 prefill、decode、KV 还是调度瓶颈？ |
| 9 | [[09-Scaling-Laws|Scaling Laws]] | 参数量、数据量、算力应该如何在预算下分配？ |
| 10 | [[10-Model-Evaluation|模型评估]] | 评估如何定位模型在目标任务上的失败模式？ |

如果已经熟悉 tokenizer，可以从 [[01-Transformer|Transformer]] 开始；如果只做应用开发，至少先读 `00、02、05、06、10`。

---

# 2. 面试复习地图

时间有限时，优先掌握这些能串起多篇文档的问题：

1. LLM 为什么本质上是 next token prediction？
2. token、embedding、hidden state、logits 分别处在生成链路的哪一步？
3. Self-Attention 的 Q/K/V、Mask、多头分别解决什么问题？
4. 为什么现在通用 LLM 多用 Decoder-only？
5. SFT、RLHF、DPO 都属于后训练，但它们分别改变什么？
6. Temperature、Top-p、Top-k 为什么都不是“提高智商”的参数？
7. 上下文窗口、KV Cache、prefill、decode 的关系是什么？
8. 长上下文为什么要区分“放得进、算得动、用得好”？
9. FlashAttention、PagedAttention、Continuous Batching 分别优化哪类瓶颈？
10. PPL、Benchmark、业务黄金集、LLM-as-a-Judge 分别适合评估什么？

---

# 3. 与其他目录的边界

- 本目录讲 LLM 本体基础。
- Prompt 技巧放到 `Prompt Engineering`。
- RAG 检索、切分、重排放到 `RAG`。
- LoRA、QLoRA、训练框架细节放到 `Fine-tuning`。
- Tool use、Memory、Planning 放到 `Agent 核心组件`。
- vLLM、TGI、SGLang、服务治理放到 `Agent 工程化与部署`。

---

# 4. 阅读建议

读每篇时不要只背名词，建议始终问三件事：

1. 这个概念解决什么问题？
2. 同一类概念之间的差异是什么？
3. 工程上用错会造成什么后果？

能回答这三个问题，才算真正把 LLM 基础学进去了。
