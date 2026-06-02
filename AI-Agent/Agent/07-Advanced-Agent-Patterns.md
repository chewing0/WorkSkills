---
tags:
  - LLM
  - AI-Agent
  - Agent
created: 2026-06-01
description: Agent 进阶模式与协作范式，解释 Reflection、Self-Refine、Tree of Thoughts、多 Agent、Agentic RAG 和模式取舍
---

# 进阶模式与协作范式

理解 Agent 基础后，会遇到很多模式名：ReAct、Reflection、Self-Refine、Tree of Thoughts、多 Agent、Agentic RAG。不要先背名字，要先问：这个模式解决的是规划问题、检索问题、质量问题、协作问题，还是恢复问题？

模式的价值在于解决具体瓶颈，而不是让系统看起来更复杂。

---

# 1. Reflection 与 Self-Refine

Reflection 让模型回看自己的输出或行动轨迹，发现问题并改进。

适合：

- 初稿质量不稳定。
- 代码或写作需要自审。
- 工具结果显示部分失败。
- 需要生成改进建议。

但 Reflection 不等于可靠验证。模型自我检查仍可能错。关键任务要引入外部证据、测试或人工审核。

Self-Refine 更偏“生成 -> 反馈 -> 修改”的循环。要限制轮次，并要求反馈具体可执行，否则容易陷入无意义润色。

---

# 2. Tree of Thoughts

Tree of Thoughts 让模型探索多个候选推理路径，再选择或合并。

它适合搜索空间较大的推理任务，例如谜题、规划、方案比较。

代价是成本高，延迟大，而且评价函数难设计。对于普通业务流程，workflow 或多候选生成加评审通常更实用。

---

# 3. 多候选与评审

很多时候，不需要复杂 ToT，只需要多个候选加 Judge：

```text
生成方案 A/B/C
  -> 评审标准打分
  -> 选择或合并
```

适合创意、方案设计、写作、代码修复。评审标准要明确，例如正确性、成本、风险、可维护性、引用证据。

如果评审只说“哪个更好”，结果会不稳定。

---

# 4. 多 Agent 协作

多 Agent 是把任务拆给多个角色，例如 Planner、Researcher、Coder、Reviewer。

它的价值在于：

- 专业分工。
- 并行处理。
- 互相评审。
- 权限隔离。

代价是：

- 通信成本。
- 状态同步。
- 冲突裁决。
- trace 复杂。
- 更难评估。

不要因为任务复杂就自动上多 Agent。先看是否真的需要多个角色、多个工具权限或多个视角。

---

# 5. Agentic RAG

Agentic RAG 是让 Agent 参与检索决策，而不是固定走一次检索。

它可能做：

- 判断是否需要检索。
- 改写 query。
- 多轮检索。
- 根据结果决定补充检索。
- 选择知识源。
- 对证据做评审。

它适合复杂问题、信息不足、多知识源、需要多跳检索的场景。

风险是成本和循环。必须设置预算、停止条件和 trace。

---

# 6. Critic、Reviewer 与 Judge

Critic/Reviewer/Judge 都是质量控制角色，但重点不同：

- Critic：指出问题。
- Reviewer：按标准审查。
- Judge：在候选中裁决。

它们最好基于明确 rubric 工作，而不是泛泛地“检查一下”。高风险任务还要结合测试、检索证据和规则校验。

---

# 7. 模式选择

| 任务瓶颈 | 可选模式 |
|---|---|
| 不知道下一步做什么 | ReAct / Planning |
| 任务复杂但可拆 | Plan-and-Execute |
| 输出质量不稳定 | Self-Refine / Reviewer |
| 需要多方案比较 | 多候选 + Judge |
| 需要探索推理路径 | Tree of Thoughts |
| 需要多角色分工 | 多 Agent |
| 需要动态检索 | Agentic RAG |
| 高风险动作 | Workflow + Human-in-the-loop |

模式不是越高级越好。越高级的模式越需要预算、状态、评估和安全边界。

---

# 8. 延伸搜索清单

- Reflexion
- Self-Refine
- Tree of Thoughts
- Graph of Thoughts
- LATS
- Debate agents
- Critic agent
- Reviewer agent
- Judge agent
- Multi-agent orchestration
- Agentic RAG
- Self-RAG
- Corrective RAG
- Adaptive RAG
- Swarm
- Handoff
