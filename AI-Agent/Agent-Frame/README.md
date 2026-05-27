---
tags:
  - LLM
  - AI-Agent
  - Agent-Framework
  - 学习路线
created: 2026-05-27
description: Agent 开发框架基础知识目录，覆盖框架定位、通用抽象、LangChain、LangGraph、AutoGen、CrewAI、MetaGPT、低代码平台、选型与工程落地
---

# Agent 开发框架

> Agent 框架不是魔法，也不是 Agent 能力的来源。框架的价值在于把模型调用、Prompt、工具、状态、记忆、流程、回调、评估和部署这些重复工程抽象出来，让复杂 Agent 更容易组织、调试和维护。

---

# 1. 学习顺序

建议按下面顺序读：

```text
00 -> 01 -> 02 -> 03 -> 04 -> 05 -> 06
```

| 顺序 | 文档 | 应该带走的核心问题 |
|---:|---|---|
| 0 | [[00-Agent-Framework-Overview|Agent 框架总览]] | Agent 框架解决什么问题，什么时候不需要框架？ |
| 1 | [[01-Framework-Abstractions-And-Design|框架通用抽象与设计]] | Chain、Tool、Memory、State、Graph、Callback 等抽象到底在解决什么？ |
| 2 | [[02-LangChain-And-LCEL|LangChain 与 LCEL]] | LangChain 适合什么，LCEL 为什么是 Runnable 编排而不是简单链式调用？ |
| 3 | [[03-LangGraph-Stateful-Agents|LangGraph 与状态图 Agent]] | 为什么复杂 Agent 更适合用图和显式状态来组织？ |
| 4 | [[04-Multi-Agent-Frameworks-AutoGen-CrewAI-MetaGPT|多 Agent 框架：AutoGen、CrewAI、MetaGPT]] | 多 Agent 框架解决什么问题，又会带来什么复杂度？ |
| 5 | [[05-Low-Code-Agent-Platforms-And-Ecosystem|低代码 Agent 平台与生态]] | Dify、Coze、Flowise 等平台适合什么场景，和代码框架如何取舍？ |
| 6 | [[06-Framework-Selection-Engineering-And-Migration|框架选型、工程落地与迁移]] | 如何按业务复杂度、团队能力和生产要求选择框架？ |

如果只做应用开发，优先读 `00、01、03、06`。如果关注多 Agent 协作，再读 `04`。如果做快速业务原型或内部工具，再读 `05`。

---

# 2. 核心主线

Agent 框架通常围绕这些问题展开：

```text
模型如何调用？
  │
  ├─► Prompt 如何管理？
  ├─► 工具如何定义和执行？
  ├─► 状态如何保存和恢复？
  ├─► 多步骤流程如何编排？
  ├─► 多 Agent 如何通信协作？
  ├─► 过程如何观测和评估？
  └─► 如何部署、扩展和回滚？
```

框架不是越多越好。好的选型应该从任务形态出发：

- 单步问答：可能不需要 Agent 框架。
- RAG 问答：检索链路和评估更重要。
- 多步工具调用：需要状态、工具和错误恢复。
- 高风险业务流程：需要 workflow/state machine。
- 多角色协作：才考虑多 Agent 框架。
- 快速内部应用：低代码平台可能更合适。

---

# 3. 与其他目录的边界

- [[../Agent-Corepart/README|Agent 核心组件]]讲 Agent 的规划、记忆、工具、状态、执行和评估。
- Agent-Frame 讲不同框架如何把这些组件工程化。
- [[../Prompt-Engineering/README|Prompt Engineering]]讲单次模型输入如何设计。
- [[../RAG/README|RAG]]讲知识检索和证据生成链路。
- Agent 工程化与部署会更关注推理服务、并发、沙箱、监控和成本控制。

一句话区分：

> Agent-Corepart 是原理和组件，Agent-Frame 是把这些组件组织起来的工程骨架。

---

# 4. 面试复习地图

时间有限时，优先掌握这些问题：

1. 为什么 Agent 框架不能替代 Agent 设计？
2. Chain、Agent、Tool、Memory、State、Graph、Callback 分别抽象了什么？
3. LangChain 的优势和常见问题是什么？
4. LCEL 的 Runnable、pipe、parallel、parser 思路是什么？
5. LangGraph 为什么适合有状态、多分支、可恢复的 Agent？
6. AutoGen、CrewAI、MetaGPT 的多 Agent 协作范式有何差异？
7. Dify/Coze 这类低代码平台适合什么，不适合什么？
8. 什么时候用框架，什么时候手写 workflow 更稳？
9. Agent 框架上线时要补哪些框架外能力？
10. 如何避免被框架 lock-in？

---

# 5. 阅读建议

读框架文档时，不要先背 API，而要问：

```text
这个框架把什么复杂性隐藏了？
  │
  ├─► 隐藏后是否还能观测？
  ├─► 出错后是否能恢复？
  ├─► 状态是否显式？
  ├─► 工具权限是否可控？
  ├─► 评估和 trace 是否完整？
  └─► 迁移成本是否可接受？
```

框架越高级，越要关注可控性。原型阶段追求速度，生产阶段追求可观察、可测试、可恢复。

---

# 6. 延伸搜索清单

下面这些先不用背，主线理解后再搜索：

- LangChain Runnable
- LCEL
- LangGraph StateGraph
- Pregel execution model
- AutoGen ConversableAgent
- GroupChat
- CrewAI Process
- MetaGPT SOP
- Dify workflow
- Coze bot
- Flowise
- Haystack
- LlamaIndex workflow
- Semantic Kernel
- Instructor
- Pydantic AI
- OpenAI Agents SDK
- MCP server
- A2A protocol
- Agent trace
- LangSmith
- Langfuse
- OpenTelemetry for LLM
- Human-in-the-loop workflow
- Durable execution
- Temporal workflow

