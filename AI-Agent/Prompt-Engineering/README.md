---
tags:
  - LLM
  - AI-Agent
  - Prompt-Engineering
  - 学习路线
created: 2026-05-24
description: Prompt Engineering 基础知识目录，覆盖提示词结构、上下文设计、示例学习、推理拆解、结构化输出、工具调用、安全与评估
---

# Prompt Engineering

> 这个目录用于梳理 LLM 应用开发中最基础、最常用、也最容易被误解的 Prompt Engineering。目标不是背技巧清单，而是理解：Prompt 如何影响模型行为，为什么有些写法稳定，有些写法只在 demo 里好看。

---

# 1. 学习顺序

建议按下面顺序读：

```text
00 -> 01 -> 02 -> 03 -> 04 -> 05
```

| 顺序 | 文档 | 应该带走的核心问题 |
|---:|---|---|
| 0 | [[00-Prompt-Engineering-Overview|Prompt Engineering 总览]] | Prompt 为什么能影响 LLM 行为，它的边界在哪里？ |
| 1 | [[01-Prompt-Structure-And-Instructions|Prompt 结构与指令设计]] | 一个稳定 prompt 应该由哪些部分组成，指令如何写才不含糊？ |
| 2 | [[02-Context-Examples-And-Task-Framing|上下文、示例与任务表述]] | Zero-shot、Few-shot 和上下文材料到底在改变什么？ |
| 3 | [[03-Reasoning-Decomposition-And-Planning|推理、拆解与规划]] | 如何让模型处理多步任务，而不是把“请认真思考”当咒语？ |
| 4 | [[04-Structured-Output-Tool-Use-And-Workflows|结构化输出、工具调用与工作流]] | 如何让模型输出可被程序消费的结果，并安全地使用工具？ |
| 5 | [[05-Prompt-Safety-Evaluation-And-Iteration|Prompt 安全、评估与迭代]] | 如何防 prompt 注入，如何用评估集持续改 prompt？ |

如果只做业务应用开发，优先读 `00、01、02、04、05`。如果准备 Agent 面试或复杂工作流设计，再重点读 `03、04、05`。

---

# 2. 核心主线

Prompt Engineering 的本质不是“写一句神奇的话让模型变聪明”，而是：

> 把任务目标、上下文、约束、示例和输出契约组织成模型容易遵循、系统容易校验的输入。

它连接三件事：

```text
用户真实意图
  │
  ├─► Prompt：任务、上下文、约束、示例、输出格式
  │
  ├─► Model：基于上下文生成下一个 token
  │
  └─► System：解析、校验、工具调用、评估、重试
```

理解 Prompt Engineering 时，始终问四个问题：

1. 模型需要完成什么任务？
2. 模型完成任务需要哪些信息？
3. 哪些输出是允许的，哪些输出是错误的？
4. 系统如何判断这次输出是否可用？

如果一个 prompt 只能让人觉得“写得很像提示词”，但不能回答这四个问题，它通常不够工程化。

---

# 3. 与其他目录的边界

- [[../LLM-Basic/README|LLM 基础]]讲模型为什么能生成、如何解码、上下文和评估是什么。
- Prompt Engineering 讲如何组织模型输入，让模型行为更稳定、更贴近任务。
- RAG 讲如何把外部知识检索进上下文，Prompt 只负责让模型正确使用这些上下文。
- Fine-tuning 讲通过训练改变模型参数，Prompt 只改变本次推理上下文。
- Agent 组件讲 Planning、Memory、Tool Use，Prompt Engineering 是这些组件和模型交互的接口层。

一句话区分：

> Prompt 能改变模型“这次怎么做”，不能可靠改变模型“本来不会什么”。

---

# 4. 面试复习地图

时间有限时，优先掌握这些问题：

1. Prompt Engineering 为什么不是简单的“咒语工程”？
2. system prompt、user prompt、上下文材料、示例、输出格式分别解决什么问题？
3. Zero-shot 和 Few-shot 的差异是什么，示例为什么会改变模型输出？
4. 为什么 role playing 有时有用，但不能替代任务定义和输出约束？
5. Chain-of-Thought、任务拆解、自检、Self-Consistency 分别适合什么场景？
6. 为什么结构化输出不能只靠“请输出 JSON”？
7. Function Calling 和普通 JSON 输出的区别是什么？
8. Prompt 注入为什么在 RAG 和工具调用场景里尤其危险？
9. 如何构建 prompt 的评估集，而不是凭感觉反复改？
10. Prompt、模型参数、RAG、工具 schema 和后置校验分别该负责什么？

---

# 5. 阅读建议

读 Prompt Engineering 时，不要只记模板。每看到一个技巧，都把它放回下面这张图：

```text
任务定义
  │
  ├─► 指令是否清晰？
  ├─► 上下文是否足够？
  ├─► 示例是否代表目标分布？
  ├─► 约束是否可执行？
  ├─► 输出是否可校验？
  └─► 失败是否可回归测试？
```

真正有用的 prompt 往往不华丽，但边界清楚、输入干净、输出稳定、失败可定位。

---

# 6. 延伸搜索清单

下面这些属于细枝末节或进阶方向，先不用背，感兴趣时再搜索：

- prompt compression
- automatic prompt optimization
- soft prompt / prefix tuning
- prompt ensembling
- prompt chaining
- constitutional AI
- adversarial prompting
- jailbreak benchmark
- prompt leakage
- instruction hierarchy
- XML prompting
- DSPy
- Guidance / Outlines / LMQL
- constrained decoding
- semantic router
- synthetic data for prompt eval
- rubric-based LLM judge
- retrieval prompt injection
- multi-agent debate prompting
- Reflexion / LATS / Tree-of-Thoughts / Graph-of-Thoughts

