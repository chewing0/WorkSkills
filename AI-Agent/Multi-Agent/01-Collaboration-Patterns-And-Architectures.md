---
tags:
  - LLM
  - AI-Agent
  - Multi-Agent
created: 2026-05-27
description: Multi-Agent 协作模式与系统架构，解释主从式、层级式、扁平协作、辩论评审、黑板系统和流水线移交
---

# 协作模式与系统架构

Multi-Agent 的架构设计，本质是在回答一个问题：多个 Agent 之间的控制权如何流动？

有些系统由一个协调者统一分配任务，有些系统让多个 Agent 平等讨论，有些系统按层级逐级分解，有些系统让生成者和评审者互相制衡。不同模式没有绝对优劣，关键看任务是否开放、风险是否高、是否需要并行，以及最终责任应该归谁。

---

# 1. Orchestrator-Worker

Orchestrator-Worker 是最常见、也最容易落地的多 Agent 模式。

```text
用户目标
  │
  ▼
Orchestrator
  ├─► Worker A
  ├─► Worker B
  ├─► Worker C
  └─► 汇总、裁决、返回结果
```

Orchestrator 负责理解目标、拆解任务、选择 Worker、合并结果和判断是否结束。Worker 只负责自己被分配的子任务。

这个模式适合任务边界相对清楚的场景，例如研究报告、代码生成、数据分析、客服工单分派。它的优点是可控，责任集中，trace 容易读。缺点是 Orchestrator 容易成为瓶颈：它如果拆错任务，下面的 Worker 再努力也只能在错误方向上优化。

设计时要注意，Orchestrator 不应该只是“把用户问题转发给所有人”。它需要维护任务状态，记录每个子任务的输入、输出、依赖、失败原因和是否需要重试。

---

# 2. 层级式架构

层级式可以看作多层 Orchestrator-Worker。

```text
总协调 Agent
  ├─► 研究组 Leader
  │   ├─► 检索 Agent
  │   └─► 事实核验 Agent
  ├─► 实现组 Leader
  │   ├─► 编码 Agent
  │   └─► 测试 Agent
  └─► 发布组 Leader
      ├─► 文档 Agent
      └─► 风险审查 Agent
```

层级式适合大任务，因为它把复杂度分层压缩。总协调 Agent 不必直接管理所有细节，只需要管理各组 Leader；每个 Leader 再管理自己的局部任务。

它的问题也很明显：信息在层级之间传递时会被摘要、过滤和误解。上层如果拿到的是过度简化的摘要，可能做出错误决策；下层如果拿不到全局目标，可能局部最优。

因此，层级式架构需要明确两类信息：向下传递的是目标、约束和完成标准；向上传递的是结论、证据、风险和未解决问题。

---

# 3. 扁平协作

扁平协作中，多个 Agent 地位接近，可以互相发送消息、补充信息、提出修改建议。

这种模式适合探索性任务，例如头脑风暴、方案比较、创意生成、复杂问题讨论。它能产生多样观点，但也最容易失控，因为没有明确裁决者时，讨论可能变成循环。

扁平协作要成立，通常需要额外规则：

- 规定每个 Agent 的发言轮次和最长消息长度。
- 规定每轮讨论的目标，例如提出方案、质疑方案、合并方案。
- 设置 Moderator 或 Judge 负责收敛。
- 设置停止条件，例如达到共识、超过预算、置信度足够、进入人工审批。

没有这些规则，扁平协作只是“多个模型互相聊天”，工程上很难稳定复现。

---

# 4. Debate 与 Critic

Debate 模式让多个 Agent 提出不同观点或候选答案，再通过评审、投票或仲裁选择结果。

```text
问题
  ├─► Agent A：方案 A
  ├─► Agent B：方案 B
  ├─► Agent C：反例与风险
  └─► Judge：证据审查与最终裁决
```

它适合开放性推理、方案评估、需求评审、代码审查、事实核验等任务。它的价值不是让 Agent “吵赢”，而是让系统暴露不同假设。

Critic 模式更常见：一个 Agent 负责生成，另一个 Agent 负责审查。审查可以按规则、测试、证据或风险清单进行。Critic 的 Prompt 不能只写“检查是否有问题”，而要给出具体标准，例如事实是否有来源、代码是否可运行、是否满足需求、是否有安全风险。

Debate 和 Critic 的局限是，它们仍可能共享同一类模型偏差。如果所有 Agent 都没有外部证据，只是语言推理，那么多个 Agent 的一致意见也可能是共同幻觉。

---

# 5. Blackboard 架构

Blackboard 架构来自传统 AI 系统，核心思想是多个 Agent 不直接长时间对话，而是在一个共享工作区读写中间结果。

```text
共享黑板
  ├─ 任务列表
  ├─ 已知事实
  ├─ 候选方案
  ├─ 证据与引用
  ├─ 风险与待办
  └─ 决策记录

Agent A / Agent B / Agent C 围绕黑板协作
```

这种模式适合需要长期积累中间状态的任务，例如研究、规划、代码项目、故障排查。它比纯聊天更稳定，因为关键状态被结构化保存。

黑板不是随便共享一段文本。好的黑板应该有 schema，例如 facts、assumptions、open_questions、decisions、artifacts、risks。每次写入都应记录来源和时间，必要时记录置信度。

---

# 6. Pipeline 与 Handoff

Pipeline 是最确定的协作模式：一个 Agent 的输出成为下一个 Agent 的输入。

```text
需求分析 -> 方案设计 -> 实现 -> 测试 -> 审查 -> 总结
```

它不像开放式多 Agent 那么灵活，但更容易评估和上线。很多业务系统实际应该使用 pipeline，而不是自由对话式 Agent 群。

Handoff 是动态转交：当前 Agent 判断自己无法或不应继续处理，于是把任务交给另一个 Agent。客服场景很典型：通用客服 Agent 识别到退款问题，转交给售后 Agent；识别到高风险投诉，再转交人工。

Handoff 的关键是转交包要完整，包括用户目标、已完成动作、当前状态、失败原因、下一步建议和权限要求。否则接手 Agent 会重复询问或重新推理。

---

# 7. 如何选择架构

可以用下面的判断来选：

| 任务特征 | 优先模式 |
|---|---|
| 子任务清楚，最终责任明确 | Orchestrator-Worker |
| 任务庞大，需要分层管理 | 层级式架构 |
| 需要多观点探索 | 扁平协作加 Moderator |
| 需要质量审查或风险控制 | Generator-Critic |
| 需要持续积累中间状态 | Blackboard |
| 流程稳定、可测试 | Pipeline |
| 需要按能力或权限转接 | Handoff |

工程上常见的不是单一模式，而是组合模式。例如，一个总 Orchestrator 调用多个 Worker；每个 Worker 把结果写入黑板；关键输出再交给 Critic 审查；高风险步骤进入人工审批。

---

# 8. 延伸搜索清单

- Orchestrator-worker architecture
- Supervisor agent
- Hierarchical agents
- Peer-to-peer agents
- Debate agents
- Generator critic pattern
- Blackboard system
- Pipeline agent workflow
- Handoff protocol
- Moderator agent
- Judge agent
- Contract Net Protocol
- MapReduce agents
- Mixture of agents
