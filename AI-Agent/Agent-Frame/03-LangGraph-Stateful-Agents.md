---
tags:
  - LLM
  - AI-Agent
  - LangGraph
  - StateGraph
created: 2026-05-27
description: LangGraph 与状态图 Agent 基础，讲清 StateGraph、节点、边、条件分支、循环、checkpoint、Human-in-the-loop 和适用场景
---

> **核心考点**：LangGraph 的核心价值是把 Agent 从“模型自由循环”变成“显式状态图”。节点负责执行，边负责流转，state 负责承载上下文，checkpoint 负责恢复和人工介入。

---

# 1. 为什么需要 LangGraph

复杂 Agent 常有：

- 多步骤。
- 多工具。
- 分支判断。
- 循环重试。
- 人工确认。
- 状态恢复。
- 中断后继续。

如果用普通 chain 写，会很快变成嵌套 if/while 和隐藏状态。

LangGraph 用图表达流程：

```text
start -> planner -> tool_or_answer
tool_or_answer -> tool -> planner
tool_or_answer -> final
```

它让 Agent 的控制流显式化。

---

# 2. StateGraph 的核心概念

## 2.1 State

State 是全图共享的结构化数据。

例如：

```json
{
  "messages": [],
  "plan": [],
  "tool_results": {},
  "attempts": 0,
  "final_answer": null
}
```

节点读取 state，返回 state update。

## 2.2 Node

Node 是一个执行单元：

- 调用模型。
- 调用工具。
- 检索文档。
- 校验输出。
- 请求人工确认。

## 2.3 Edge

Edge 决定节点之间怎么流转。

普通边：

```text
A -> B
```

条件边：

```text
if needs_tool -> tool_node
else -> final_node
```

---

# 3. 为什么显式 State 很重要

显式 state 能解决：

- 多轮状态丢失。
- 工具结果难追踪。
- 重试次数不清。
- 中断无法恢复。
- 流程不可审计。

相比把所有东西塞进 messages，结构化 state 更适合程序控制。

messages 可以是 state 的一部分，但不应是唯一 state。

---

# 4. 循环和停止条件

LangGraph 支持循环：

```text
agent -> tool -> agent -> tool -> agent
```

但循环必须有停止条件：

- 任务完成。
- 达到最大步数。
- 工具失败次数过多。
- 需要人工。
- 状态没有变化。

否则图只是把死循环写得更优雅。

---

# 5. Checkpoint 和持久化

Checkpoint 允许：

- 中断后恢复。
- 人工确认后继续。
- 回放执行过程。
- 调试历史状态。

适合：

- 长任务。
- 需要审批的任务。
- 多轮异步任务。
- 生产 Agent。

没有 checkpoint 的 Agent，很难在真实系统中可靠运行。

---

# 6. Human-in-the-loop

LangGraph 适合在图中插入人工节点：

```text
高风险动作 -> 等待人工确认 -> 执行或取消
```

人工节点需要清楚展示：

- Agent 准备做什么。
- 参数是什么。
- 影响范围。
- 依据是什么。
- 可选动作。

人工确认不是让人看一坨日志，而是看经过整理的决策摘要。

---

# 7. LangGraph 适合的场景

适合：

- 多步工具调用。
- 有循环和分支的 Agent。
- 需要状态恢复。
- 需要人工确认。
- 需要 trace 和可视化流程。
- 生产级客服、代码、运维、数据分析 Agent。

不一定需要：

- 简单单轮问答。
- 简单 RAG。
- 无状态流水线。

---

# 8. 设计 LangGraph 的思路

先画业务状态，而不是先写节点。

```text
任务有哪些阶段？
每个阶段需要哪些输入？
哪些节点会改变 state？
哪些操作有风险？
哪些地方可能失败？
失败后怎么走？
什么时候结束？
```

再设计节点：

- model node。
- tool node。
- validation node。
- human node。
- final node。

好的图应该让人一眼看出流程，而不是把复杂逻辑藏进一个巨大的 node。

---

# 9. 常见误区

## 9.1 用 LangGraph 就不会失控

不对。图只是显式控制流。你仍然要设计停止条件、权限、校验和评估。

## 9.2 所有逻辑都塞进一个节点

这样失去图的意义。节点应边界清楚，便于调试和复用。

## 9.3 State 越大越好

不一定。State 太大难维护。应保存关键结构化信息，长文本用引用或外部存储。

## 9.4 简单任务也上图

简单链路用普通 chain 更轻。LangGraph 适合复杂状态和控制流。

---

# 10. 面试 Q&A

## Q1：LangGraph 解决了 LangChain Agent 的什么问题？

它让复杂 Agent 的状态、分支、循环和 checkpoint 显式化，适合多步、可恢复、需要人工确认的生产 Agent。

## Q2：StateGraph 的核心组件是什么？

State、Node、Edge。State 存储图状态，Node 执行动作并更新状态，Edge 决定流转，条件边支持分支。

## Q3：为什么 checkpoint 重要？

它支持中断恢复、人工确认、回放和审计，避免长任务失败后只能从头开始。

## Q4：什么时候不需要 LangGraph？

简单 prompt、单轮 RAG、无状态流水线通常不需要。引入图会增加复杂度。

---

# 11. 相关链接

- [[README|Agent 框架目录]]
- [[01-Framework-Abstractions-And-Design]]
- [[06-Framework-Selection-Engineering-And-Migration]]
- [[../Agent-Corepart/05-State-Workflow-And-Human-In-The-Loop|状态、工作流与 Human-in-the-Loop]]

