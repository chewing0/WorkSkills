---
tags:
  - LLM
  - AI-Agent
  - Agent-Core
  - 学习路线
created: 2026-05-27
description: Agent 核心组件基础知识目录，覆盖 Agent 总览、规划、记忆、工具使用、行动执行、状态工作流、评估安全与可观测性
---

# Agent 核心组件

> LLM Agent 的核心不是“让模型自己想办法”，而是把模型放进一个有目标、有状态、有工具、有权限、有反馈的控制系统里。模型负责语言理解、规划和决策的一部分；系统负责状态、工具、权限、执行、校验和观测。

---

# 1. 学习顺序

建议按下面顺序读：

```text
00 -> 01 -> 02 -> 03 -> 04 -> 05 -> 06
```

| 顺序 | 文档 | 应该带走的核心问题 |
|---:|---|---|
| 0 | [[00-Agent-Core-Overview|Agent 核心总览]] | Agent 和普通 Chat/RAG 应用有什么本质区别？ |
| 1 | [[01-Planning-And-Task-Decomposition|规划与任务拆解]] | Agent 如何把目标拆成可执行步骤，什么时候需要动态调整计划？ |
| 2 | [[02-Memory-And-Context-Management|记忆与上下文管理]] | 对话历史、工作状态、长期记忆和 RAG 上下文分别解决什么问题？ |
| 3 | [[03-Tool-Use-And-Function-Calling|工具使用与 Function Calling]] | 模型如何选择工具、生成参数，系统如何校验和执行？ |
| 4 | [[04-Action-Execution-And-Sandbox|行动执行与安全边界]] | API 调用、代码执行、文件操作为什么必须有权限、审计和回滚？ |
| 5 | [[05-State-Workflow-And-Human-In-The-Loop|状态、工作流与 Human-in-the-Loop]] | 为什么生产 Agent 需要状态机、checkpoint 和人工确认？ |
| 6 | [[06-Agent-Evaluation-Safety-And-Observability|Agent 评估、安全与可观测性]] | 如何评估一个 Agent 是否可靠，而不是只看最终回答？ |

如果只做应用开发，优先读 `00、03、05、06`。如果做复杂自动化或多步 Agent，再重点读 `01、02、04`。

---

# 2. 核心主线

一个 Agent 可以抽象成循环：

```text
目标
  │
  ├─► 观察：当前用户输入、状态、工具结果、环境反馈
  ├─► 思考：理解目标，规划下一步，判断是否需要工具
  ├─► 行动：调用工具、检索、执行 API、读写文件、请求人工确认
  ├─► 反馈：读取行动结果，更新状态
  └─► 停止：达到目标、无法继续、需要人工、超过预算
```

这条链路里的核心组件：

- Planning：决定做什么、先后顺序和何时调整。
- Memory：保存当前任务状态和可复用知识。
- Tool Use：把模型的意图转成受控工具调用。
- Action：真正执行外部操作，并处理权限和失败。
- State：记录任务进度，支持恢复、分支和审计。
- Evaluation：评估过程和结果，发现循环、幻觉、越权和成本问题。

---

# 3. 与其他目录的边界

- [[../LLM-Basic/README|LLM 基础]]讲模型如何生成、上下文窗口、推理和评估。
- [[../Prompt-Engineering/README|Prompt Engineering]]讲如何组织输入，让模型稳定执行某一步任务。
- [[../RAG/README|RAG]]讲如何检索外部知识，Agent 可以把 RAG 当作知识工具。
- [[../Fine-tuning/README|Fine-tuning]]讲如何通过训练改变模型行为，Agent 仍需要工具权限、状态和评估。
- Agent-Corepart 讲把 LLM、Prompt、RAG、工具和状态组合成可执行系统的核心组件。

一句话区分：

> Prompt 让模型更好地回答一步，RAG 给模型证据，Fine-tuning 改模型行为倾向，Agent 把多步决策和外部行动串成系统。

---

# 4. 面试复习地图

时间有限时，优先掌握这些问题：

1. Agent 和普通 Chatbot 的区别是什么？
2. ReAct、Plan-and-Execute、状态机式 Agent 的差别是什么？
3. 为什么复杂 Agent 不能只靠模型自由循环？
4. 短期记忆、长期记忆、RAG、状态管理分别是什么？
5. Function Calling 为什么不等于工具安全？
6. 工具 schema 应该如何设计，参数错误如何处理？
7. 高风险 Action 为什么要权限、幂等、审计和人工确认？
8. Agent 陷入死循环、工具失败、幻觉严重时如何治理？
9. Agent 评估为什么要看 trace，而不只看最终答案？
10. 生产 Agent 如何控制成本、延迟和失败恢复？

---

# 5. 阅读建议

读 Agent 核心组件时，始终把问题放回这张图：

```text
目标是否清楚？
  │
  ├─► 当前状态是否可见？
  ├─► 下一步是否可执行？
  ├─► 工具参数是否可校验？
  ├─► 行动是否有权限和回滚？
  ├─► 失败是否能恢复？
  ├─► 是否知道何时停止？
  └─► 全过程是否可追踪和评估？
```

真正可靠的 Agent 不是“更会想”，而是每一步都可控、可观察、可恢复。

---

# 6. 延伸搜索清单

下面这些先不用背，主线理解后再搜索：

- ReAct
- Plan-and-Execute
- Plan-and-Solve
- Reflexion
- LATS
- Toolformer
- Function Calling
- JSON Schema for tools
- LangGraph state graph
- Checkpointing
- Human-in-the-Loop
- Long-term memory
- Episodic memory
- Semantic memory
- Working memory
- Vector memory
- Memory consolidation
- Tool sandbox
- Idempotency key
- Agent trace
- Agent trajectory evaluation
- AgentBench
- WebArena
- SWE-bench
- Voyager
- AutoGPT
- Multi-agent orchestration
- MCP
- A2A

