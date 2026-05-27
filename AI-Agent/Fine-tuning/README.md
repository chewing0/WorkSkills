---
tags:
  - LLM
  - AI-Agent
  - Fine-tuning
  - 学习路线
created: 2026-05-25
description: LLM Fine-tuning 基础知识目录，覆盖微调定位、数据设计、SFT、LoRA/QLoRA、偏好对齐、训练工程、评估与上线
---

# Fine-tuning

> Fine-tuning 的目标不是“把知识塞进模型”这么简单，而是在明确任务、数据、成本和风险的前提下，稳定改变模型的行为分布。它适合让模型更会按某种格式、风格、流程和领域任务工作；不适合替代知识库、权限系统和事实校验。

---

# 1. 学习顺序

建议按下面顺序读：

```text
00 -> 01 -> 02 -> 03 -> 04 -> 05 -> 06
```

| 顺序 | 文档 | 应该带走的核心问题 |
|---:|---|---|
| 0 | [[00-Fine-Tuning-Overview|Fine-tuning 总览]] | 微调到底在改变什么，什么时候该微调，什么时候不该？ |
| 1 | [[01-Data-Design-And-Formatting|数据设计与格式]] | 为什么数据质量比训练技巧更重要，指令数据应该怎么构造？ |
| 2 | [[02-SFT-And-Instruction-Tuning|SFT 与指令微调]] | SFT 如何把 base model 或 chat model 调成更稳定的任务助手？ |
| 3 | [[03-PEFT-LoRA-QLoRA-And-Adapters|PEFT、LoRA、QLoRA 与 Adapter]] | 如何用更低成本微调模型，LoRA/QLoRA 的核心原理和边界是什么？ |
| 4 | [[04-Preference-Alignment-RLHF-DPO-And-RM|偏好对齐、RLHF、DPO 与 Reward Model]] | SFT 之后为什么还要偏好对齐，RLHF/DPO 分别解决什么问题？ |
| 5 | [[05-Training-Systems-And-Optimization|训练工程与优化]] | 训练时显存、并行、优化器、checkpoint、混合精度和稳定性怎么理解？ |
| 6 | [[06-Evaluation-Deployment-And-Iteration|评估、部署与迭代]] | 如何判断微调真的变好了，并安全上线和持续迭代？ |

如果只做应用开发，优先读 `00、01、02、03、06`。如果要做训练平台或对齐算法，再重点读 `04、05`。

---

# 2. 核心主线

Fine-tuning 的完整链路：

```text
明确目标
  │
  ├─► 判断是否真的需要微调
  ├─► 选择基座模型和训练方式
  ├─► 构造高质量数据
  ├─► SFT 学任务格式和行为
  ├─► PEFT 降低训练成本
  ├─► 偏好对齐改善偏好和安全边界
  ├─► 评估能力、回归和副作用
  └─► 上线监控、收集失败样本、继续迭代
```

学习时始终问四个问题：

1. 想改变模型的什么行为？
2. 这个问题能否用 prompt、RAG、工具或后处理解决？
3. 数据是否足够代表目标任务和失败边界？
4. 微调后如何证明收益大于成本和风险？

---

# 3. 与其他目录的边界

- [[../LLM-Basic/README|LLM 基础]]讲模型结构、训练目标、生成策略和评估基础。
- [[../Prompt-Engineering/README|Prompt Engineering]]通过上下文引导本次推理，不改变模型参数。
- [[../RAG/README|RAG]]通过检索补充外部知识，适合新鲜知识和可溯源问答。
- Fine-tuning 改变模型参数或附加参数，适合长期稳定改变行为、格式、风格和领域任务能力。
- Agent 工具调用和工作流不应完全交给微调，权限、状态和外部动作仍应由系统控制。

一句话区分：

> Prompt 改“这次怎么做”，RAG 给“这次用什么证据”，Fine-tuning 改“模型以后更倾向怎么做”。

---

# 4. 面试复习地图

时间有限时，优先掌握这些问题：

1. Fine-tuning、Prompt、RAG 的适用边界分别是什么？
2. 为什么说微调的核心瓶颈通常是数据，而不是训练代码？
3. SFT 的 loss 通常算在哪些 token 上，为什么要 mask 用户部分？
4. Base model SFT 和 chat model 继续 SFT 有什么差别？
5. LoRA 为什么能降低训练成本，rank、alpha、target modules 各影响什么？
6. QLoRA 为什么能在低显存下训练，量化和 LoRA 分别负责什么？
7. RLHF、Reward Model、PPO、DPO 的关系是什么？
8. 微调如何造成灾难性遗忘、过拟合和安全能力退化？
9. 微调模型应该如何评估，不只看 loss？
10. 微调后的模型如何上线、回滚和继续迭代？

---

# 5. 阅读建议

每个微调方案都要放回这张图：

```text
问题是否适合微调？
  │
  ├─► 数据是否覆盖目标行为？
  ├─► 格式是否匹配模型 chat template？
  ├─► 训练方式是 full fine-tune、LoRA 还是 QLoRA？
  ├─► 评估集是否覆盖能力、格式、安全和回归？
  ├─► 上线后是否能监控失败样本？
  └─► 出问题是否能回滚？
```

微调不是越多越好。真正好的微调方案通常很克制：目标清楚、数据干净、训练成本可控、评估可复现。

---

# 6. 延伸搜索清单

下面这些属于细枝末节或进阶方向，主线掌握后再查：

- LoRA rank stabilization
- DoRA
- AdaLoRA
- LoRA merge / multi-adapter routing
- QLoRA NF4 / double quantization
- LoftQ
- GaLore
- ZeRO-1 / ZeRO-2 / ZeRO-3
- FSDP
- Tensor Parallel / Pipeline Parallel
- FlashAttention training
- Gradient checkpointing
- Packing / sequence packing
- Chat template
- Loss masking
- Preference data
- Reward hacking
- PPO clipping
- DPO / IPO / KTO / ORPO / SimPO
- Constitutional AI
- Catastrophic forgetting
- Model merging
- Continual learning
- Safety regression eval
- lm-evaluation-harness

