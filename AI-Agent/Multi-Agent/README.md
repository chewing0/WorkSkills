---
tags:
  - LLM
  - AI-Agent
  - Multi-Agent
  - 学习路线
created: 2026-05-27
description: Multi-Agent 基础知识目录，覆盖多智能体定位、协作架构、通信状态、任务分配、冲突共识、共享知识、生产治理与评估
---

# Multi-Agent

> Multi-Agent 不是“多开几个聊天机器人”，而是把一个复杂目标拆成多个具备角色、权限、工具、状态和通信协议的执行单元，再通过协作机制把它们组织成一个系统。它的价值不在于热闹，而在于让复杂任务可以分工、并行、互检和治理。

---

# 1. 学习顺序

建议按下面顺序读：

```text
00 -> 01 -> 02 -> 03 -> 04 -> 05 -> 06
```

| 顺序 | 文档 | 应该带走的核心问题 |
|---:|---|---|
| 0 | [[00-Multi-Agent-Overview|Multi-Agent 总览]] | 多 Agent 解决什么问题，为什么也会制造新的复杂度？ |
| 1 | [[01-Collaboration-Patterns-And-Architectures|协作模式与系统架构]] | 主从、层级、扁平、辩论、黑板等架构分别适合什么任务？ |
| 2 | [[02-Communication-Protocols-And-Shared-State|通信协议、消息与共享状态]] | Agent 之间传什么、怎么传、如何避免上下文污染和状态混乱？ |
| 3 | [[03-Task-Decomposition-Allocation-And-Orchestration|任务拆解、分配与调度]] | 任务如何拆给不同 Agent，静态分工和动态调度如何取舍？ |
| 4 | [[04-Conflict-Consensus-And-Quality-Control|冲突、共识与质量控制]] | 多 Agent 意见不一致时如何裁决，如何避免群体幻觉？ |
| 5 | [[05-Memory-Knowledge-And-Tool-Coordination|记忆、知识与工具协同]] | 多 Agent 如何共享知识、隔离权限、协调工具调用和外部行动？ |
| 6 | [[06-Production-Multi-Agent-Safety-Cost-And-Evaluation|生产治理、安全、成本与评估]] | 多 Agent 系统上线时如何控制成本、延迟、风险和可观测性？ |

如果只想快速建立主线，优先读 `00、01、03、06`。如果在设计具体系统，再重点读 `02、04、05`。

---

# 2. 核心主线

多 Agent 系统可以按下面这条线理解：

```text
复杂目标
  │
  ├─► 拆成子任务：哪些步骤可以并行，哪些必须串行？
  ├─► 定义角色：每个 Agent 负责什么，不能做什么？
  ├─► 分配工具：谁能检索、谁能写文件、谁能执行代码、谁能审批？
  ├─► 设计通信：消息格式、共享状态、上下文边界、协议约束
  ├─► 调度协作：编排、转交、重试、超时、预算、停止条件
  ├─► 冲突裁决：投票、评审、仲裁、置信度、证据要求
  └─► 评估治理：成功率、过程质量、成本、延迟、安全、可追踪性
```

判断一个多 Agent 设计是否合理，可以先问一个问题：

> 新增一个 Agent 是降低了复杂度，还是只是把一个难问题拆成了几个更难调试的问题？

真正有效的多 Agent 通常有清楚的分工边界、明确的通信协议、可控的工具权限和可追踪的执行过程。

---

# 3. 和其他目录的边界

- [[../Agent-Corepart/README|Agent 核心组件]]讲单个 Agent 的规划、记忆、工具、状态、执行和评估。
- [[../Agent-Frame/README|Agent 开发框架]]讲 LangGraph、AutoGen、CrewAI、MetaGPT、Dify 等框架如何组织 Agent。
- Multi-Agent 关注多个 Agent 之间的角色划分、通信协作、调度、冲突和系统治理。
- [[../RAG/README|RAG]]可以作为多个 Agent 共享或专用的知识工具。
- [[../Prompt-Engineering/README|Prompt Engineering]]仍然重要，因为每个 Agent 的角色、目标和输出契约都要靠 Prompt 或系统指令固定下来。

一句话区分：

> 单 Agent 的核心是“如何完成一个循环”，Multi-Agent 的核心是“如何让多个循环协同而不失控”。

---

# 4. 复习思维导图

```text
Multi-Agent
  ├─ 为什么需要
  │   ├─ 专业分工
  │   ├─ 并行处理
  │   ├─ 互相评审
  │   └─ 权限隔离
  ├─ 怎么协作
  │   ├─ Orchestrator-Worker
  │   ├─ Hierarchical
  │   ├─ Peer-to-Peer
  │   ├─ Debate/Critic
  │   └─ Blackboard
  ├─ 怎么通信
  │   ├─ 消息格式
  │   ├─ 共享状态
  │   ├─ 任务状态
  │   ├─ 上下文压缩
  │   └─ 协议约束
  ├─ 怎么调度
  │   ├─ 任务拆解
  │   ├─ 静态分配
  │   ├─ 动态路由
  │   ├─ 并行合并
  │   └─ 停止条件
  ├─ 怎么保证质量
  │   ├─ 评审 Agent
  │   ├─ 投票与仲裁
  │   ├─ 证据约束
  │   ├─ 回归测试
  │   └─ trace 评估
  └─ 怎么上线
      ├─ 权限隔离
      ├─ 成本预算
      ├─ 延迟控制
      ├─ 可观测性
      └─ 人工审批
```

---

# 5. 面试复习地图

时间有限时，优先掌握这些问题：

1. 多 Agent 相比单 Agent 的收益和代价是什么？
2. Orchestrator-Worker、层级式、扁平协作、辩论式分别适合什么场景？
3. 为什么多 Agent 不能只靠自然语言互相聊天？
4. Agent 之间共享全部上下文有什么风险？
5. 静态任务分配和动态任务分配如何取舍？
6. 多 Agent 意见冲突时如何裁决？
7. 多 Agent 如何避免重复工作、循环讨论和成本爆炸？
8. 多 Agent 中工具权限应该如何隔离？
9. 如何评估一个多 Agent 系统，而不是只评估最终答案？
10. 什么时候应该放弃多 Agent，改用普通 workflow 或单 Agent？

---

# 6. 延伸搜索清单

下面这些先不用背，主线理解后再搜索：

- Orchestrator-Worker
- Supervisor Agent
- Hierarchical Multi-Agent
- Blackboard Architecture
- Contract Net Protocol
- Debate Agents
- Critic Agent
- Reviewer Agent
- Reflexion
- Self-Consistency
- AutoGen GroupChat
- CrewAI Process
- MetaGPT SOP
- LangGraph multi-agent
- Swarm
- Handoff
- MCP
- A2A protocol
- Agent Card
- Message envelope
- Shared memory
- Distributed task planning
- Multi-agent credit assignment
- Emergent communication
- Agent trace evaluation
- AgentBench
- ChatDev
- CAMEL
- SWE-bench multi-agent
