---
tags:
  - LLM
  - AI-Agent
  - Agent
created: 2026-06-01
description: Agent 框架选择与工程化落地，解释 LangChain、LangGraph、多 Agent 框架、低代码平台、运行时架构和生产工程边界
---

# 框架选择与工程化落地

Agent 框架的价值不是让模型更聪明，而是把模型调用、工具、状态、流程、回调、评估和部署这些重复工程组织起来。框架能加速开发，但不能替代系统设计。

选框架前，先判断任务形态；上线前，再补齐框架之外的工程能力。

---

# 1. 框架解决什么

常见框架能力包括：

- 封装模型调用。
- 管理 prompt 模板。
- 定义工具和 tool calling。
- 编排 chain、workflow 或 graph。
- 管理 memory 和 state。
- 记录 callback 和 trace。
- 接入向量库、检索器和外部工具。
- 支持 human-in-the-loop 或 checkpoint。

这些能力能减少样板代码，但也会隐藏细节。生产系统必须知道框架在什么时候调用模型、传入什么上下文、如何处理错误、状态保存在哪里。

---

# 2. LangChain

LangChain 更像一个 LLM 应用开发工具箱。它覆盖模型、prompt、retriever、tool、agent、chain、callback 等生态能力。

适合：

- 快速接入多种模型和工具。
- 做 RAG、简单 Agent、原型验证。
- 复用大量集成组件。

需要注意：

- 抽象层较多时，调试可能变复杂。
- 生产路径要清楚 trace 和错误处理。
- 不要为了使用框架而把简单流程复杂化。

---

# 3. LangGraph

LangGraph 更适合有状态、多分支、可恢复的 Agent。

它的核心思想是把 Agent 编排成图：

```text
State -> Node -> Edge -> Next Node -> State
```

适合：

- 多步骤工具调用。
- 需要循环但要控制循环。
- 需要 checkpoint。
- 需要 human-in-the-loop。
- 需要明确状态和分支。

复杂 Agent 往往更适合图和状态机，因为它们比纯聊天历史更可控。

---

# 4. 多 Agent 框架和低代码平台

AutoGen、CrewAI、MetaGPT 等更关注多角色协作。它们适合探索多 Agent 对话、角色分工、评审协作，但也会带来通信成本、状态复杂度和评估难度。

Dify、Coze、Flowise 等低代码平台适合快速搭建内部应用、知识库问答、流程型 Agent 和运营可配置场景。优点是快，缺点是深度定制、复杂权限、细粒度状态和工程治理可能受限。

选择时要问：

- 任务是否真的需要多 Agent？
- 状态是否需要持久化和恢复？
- 工具权限是否复杂？
- 是否需要接入现有业务系统？
- 是否能接受平台锁定？

---

# 5. 什么时候手写 workflow

如果业务路径稳定、风险高、流程强约束，手写 workflow 往往更稳。

例如：

- 退款审批。
- 合同审查。
- 生产配置修改。
- 数据库写操作。
- 合规流程。

这类场景里，LLM 可以负责理解和生成，但流程控制应由确定性代码承担。

手写 workflow 的代价是开发成本高，但可测试、可审计、可恢复。

---

# 6. 生产工程能力

框架之外，生产 Agent 仍需要：

- API 接入层：鉴权、限流、租户识别。
- Model Gateway：模型路由、重试、降级、版本记录。
- Tool Gateway：工具权限、参数校验、审计。
- State Store：会话、任务、checkpoint、工具结果。
- Queue/Worker：异步任务、重试、补偿。
- Observability：trace、日志、指标、成本。
- Evaluation：评估集、回归测试、线上反馈。
- Release：灰度、回滚、配置版本。

这些不是框架自动替你完成的，它们决定 Agent 是否能长期运行。

---

# 7. 选型建议

可以按复杂度选择：

| 任务形态 | 优先选择 |
|---|---|
| 单步问答 | 直接模型调用 |
| 知识库问答 | RAG workflow |
| 简单工具调用 | Function calling + 少量状态 |
| 多步有状态任务 | LangGraph 或状态机 |
| 固定高风险流程 | 手写 workflow |
| 快速内部应用 | Dify/Coze/Flowise |
| 多角色探索 | AutoGen/CrewAI 等 |

从简单开始，等复杂度真实出现再引入更重的框架。

---

# 8. 延伸搜索清单

- LangChain
- LCEL
- LangGraph
- StateGraph
- AutoGen
- CrewAI
- MetaGPT
- Dify
- Coze
- Flowise
- Semantic Kernel
- Pydantic AI
- OpenAI Agents SDK
- Agent runtime
- Model gateway
- Tool gateway
- Temporal
- Celery
