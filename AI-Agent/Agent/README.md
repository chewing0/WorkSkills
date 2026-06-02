---
tags:
  - LLM
  - AI-Agent
  - Agent
  - 学习路线
created: 2026-06-01
description: Agent 基础知识目录，覆盖 Agent 总览、循环与状态、工具行动、记忆上下文、工作流安全、框架工程、评估运营和进阶协作
---

# Agent

> Agent 不是“更长的 Prompt”，也不是“让模型自己随便想办法”。它是把 LLM 放进一个有目标、有状态、有工具、有权限、有反馈、有停止条件的执行系统里，让模型在受控边界内完成多步任务。

---

# 1. 学习顺序

建议按下面顺序读：

```text
00 -> 01 -> 02 -> 03 -> 04 -> 05 -> 06 -> 07
```

| 顺序 | 文档 | 应该带走的核心问题 |
|---:|---|---|
| 0 | [[00-Agent-Overview|Agent 总览]] | Agent 和普通 Chat/RAG/Workflow 的本质区别是什么？ |
| 1 | [[01-Agent-Loop-Planning-And-State|Agent 循环、规划与状态]] | Agent 如何把目标推进成可执行步骤，又如何知道何时停止？ |
| 2 | [[02-Tool-Use-And-Action-Execution|工具使用与行动执行]] | Function Calling 为什么不等于工具安全，工具调用应如何被系统接管？ |
| 3 | [[03-Memory-Context-And-Knowledge|记忆、上下文与知识]] | 对话历史、工作状态、长期记忆和 RAG 到底分别解决什么？ |
| 4 | [[04-Workflow-Human-In-The-Loop-And-Safety|工作流、人工介入与安全边界]] | 为什么生产 Agent 需要状态机、checkpoint、审批、沙箱和权限？ |
| 5 | [[05-Agent-Frameworks-And-Engineering|框架选择与工程化落地]] | 什么时候用框架，什么时候手写 workflow，生产系统要补哪些能力？ |
| 6 | [[06-Agent-Evaluation-Observability-And-Operations|评估、可观测性与运营]] | 如何判断 Agent 可靠，而不是只看最终回答是否像样？ |
| 7 | [[07-Advanced-Agent-Patterns|进阶模式与协作范式]] | ReAct、Plan-and-Execute、Reflection、多 Agent、Agentic RAG 等模式如何取舍？ |

如果只做业务 Agent 应用，优先读 `00、01、02、04、06`。如果要做平台化或复杂自动化，再重点读 `03、05、07`。

---

# 2. 核心主线

一个 Agent 可以抽象成下面的控制循环：

```text
目标
  │
  ├─► 观察：用户输入、当前状态、历史结果、外部环境
  ├─► 判断：理解目标，决定是否需要拆解、检索、调用工具或请求人工
  ├─► 行动：调用工具、查询知识、执行 API、读写文件、生成回复
  ├─► 反馈：读取工具结果，更新状态，检查是否偏离目标
  └─► 停止：完成目标、预算耗尽、失败不可恢复、需要人工确认
```

这条循环里有两个关键边界：

- 模型负责语义理解、规划建议、参数生成、结果解释。
- 系统负责权限、状态、工具执行、校验、审计、重试、回滚和停止条件。

真正可靠的 Agent 不是“模型更自由”，而是“模型的自由被放在清楚的工程边界里”。

---

# 3. 与其他目录的边界

- [[../LLM-Basic/README|LLM 基础]]讲模型如何生成、上下文窗口、KV Cache、推理优化和评估。
- [[../Prompt-Engineering/README|Prompt Engineering]]讲如何把单步任务、上下文和输出约束写清楚。
- [[../RAG/README|RAG]]讲如何检索外部知识并基于证据生成。
- [[../Fine-tuning/README|Fine-tuning]]讲如何通过训练改变模型行为倾向。
- Agent 关注如何把 LLM、Prompt、RAG、工具、状态和权限组合成可执行系统。

一句话区分：

> Prompt 让模型更好地完成一步，RAG 给模型证据，Fine-tuning 改模型行为，Agent 把多步决策和外部行动组织成系统。

---

# 4. 复习思维导图

```text
Agent
  ├─ 定位
  │   ├─ 不只是 Chat
  │   ├─ 不只是 RAG
  │   ├─ 不只是 Workflow
  │   └─ LLM + 状态 + 工具 + 反馈
  ├─ 循环
  │   ├─ Observe
  │   ├─ Plan
  │   ├─ Act
  │   ├─ Reflect
  │   └─ Stop
  ├─ 核心组件
  │   ├─ Planning
  │   ├─ Tool Use
  │   ├─ Memory
  │   ├─ State
  │   ├─ Workflow
  │   └─ Evaluation
  ├─ 工程边界
  │   ├─ Tool Gateway
  │   ├─ Permission
  │   ├─ Checkpoint
  │   ├─ Sandbox
  │   ├─ Human Approval
  │   └─ Trace
  ├─ 框架平台
  │   ├─ LangChain
  │   ├─ LangGraph
  │   ├─ AutoGen / CrewAI
  │   ├─ Dify / Coze
  │   └─ 手写 Workflow
  └─ 生产治理
      ├─ 成功率
      ├─ 成本
      ├─ 延迟
      ├─ 安全
      ├─ 可恢复
      └─ 可评估
```

---

# 5. 面试复习地图

时间有限时，优先掌握这些问题：

1. Agent 和普通 Chatbot、RAG、Workflow 的区别是什么？
2. Agent 的 Observe-Plan-Act-Reflect 循环如何工作？
3. 为什么复杂 Agent 不能只靠模型自由循环？
4. ReAct、Plan-and-Execute、状态机式 Agent 的差异是什么？
5. Function Calling 为什么不等于工具安全？
6. 短期记忆、长期记忆、工作状态、RAG 分别是什么？
7. 为什么生产 Agent 需要 checkpoint、trace 和 human-in-the-loop？
8. Agent 陷入循环、工具失败、幻觉严重时如何治理？
9. 如何选择 LangGraph、LangChain、低代码平台或手写 workflow？
10. Agent 的评估为什么要看过程，而不只看最终答案？

---

# 6. 延伸搜索清单

下面这些先不用背，主线理解后再搜索：

- ReAct
- Plan-and-Execute
- Plan-and-Solve
- Reflexion
- Self-Refine
- LATS
- Tree of Thoughts
- Function Calling
- Toolformer
- Tool Gateway
- Agent Runtime
- LangGraph
- LangChain Agents
- OpenAI Agents SDK
- Semantic Kernel
- AutoGen
- CrewAI
- Dify
- Coze
- MCP
- A2A
- Agentic RAG
- Human-in-the-loop
- Checkpointing
- Durable execution
- Agent trace
- AgentBench
- WebArena
- SWE-bench
