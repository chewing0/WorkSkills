---
tags:
  - LLM
  - AI-Agent
  - Agent-Engineering
  - Deployment
created: 2026-05-27
description: Agent 部署模式、流式响应与异步工作流，解释在线同步、Streaming、后台任务、批处理、Webhook、灰度和回滚
---

# 部署模式、流式响应与异步工作流

Agent 的部署方式要匹配任务形态。简单问答可以同步返回，长时间研究适合异步，生成式体验需要流式输出，批量文档处理适合离线任务。把所有 Agent 都做成一个同步 HTTP 接口，是很多系统早期踩坑的来源。

部署模式的核心问题是：用户需要什么时候看到什么结果，系统需要如何处理等待、失败和恢复。

---

# 1. 在线同步

在线同步是最简单的模式：客户端发请求，服务端在一个连接内完成处理并返回结果。

适合场景：

- 单轮或少轮模型调用。
- 工具调用少且稳定。
- 响应时间可控制在用户可接受范围内。
- 失败后可以让用户重新提交。

同步模式的优点是实现简单、交互清楚。缺点是长任务容易超时，占用连接和服务资源，也不利于后台重试。

Agent 场景中，同步模式适合轻量任务，例如意图识别、短文本改写、简单 RAG 问答、工具查询后总结。

---

# 2. Streaming

流式响应让用户逐步看到输出，常见实现包括 SSE、WebSocket 或 HTTP chunk。

Streaming 的价值不是让模型更快完成，而是改善感知延迟。用户不必等完整答案生成完才看到结果。

但 Agent 的 Streaming 比普通聊天复杂。因为 Agent 中间可能会规划、检索、调用工具、等待工具结果，再继续生成。此时可以流式展示这些事件：

```text
thinking_summary -> tool_call_started -> tool_call_finished -> partial_answer -> final_answer
```

需要注意，不建议把模型隐藏推理过程原样暴露或记录为产品输出。更好的方式是展示简短、可审计的过程摘要，例如“正在检索政策文档”“已找到 3 条相关订单记录”“正在生成答复”。

Streaming 还要处理取消。用户关闭页面或点击停止时，系统应能取消后续模型调用和工具调用，或把后台任务标记为 cancelled。

---

# 3. 异步任务

异步任务适合长时间、多步骤、可恢复的 Agent。

典型流程：

```text
POST /tasks -> 返回 task_id
后台 Worker 执行 Agent
GET /tasks/{id} -> 查询状态
Webhook / WebSocket / SSE -> 推送进度和结果
```

异步模式的优点是可靠。任务可以排队、重试、恢复、暂停、等待人工审批，也可以在服务重启后继续执行。

它适合：

- 长报告生成。
- 大文档解析和索引。
- 批量数据分析。
- 多 Agent 协作。
- 需要人工审批的高风险流程。
- 外部系统响应时间不确定的流程。

异步任务需要任务状态机。只返回一个 task_id 不够，还要有 created、running、waiting、blocked、failed、done、cancelled 等状态，以及失败原因和当前进度。

---

# 4. 批处理

批处理关注吞吐和成本，而不是单个请求的实时体验。

适合场景：

- 大量文档摘要。
- 离线分类和打标签。
- 批量生成 embedding。
- 离线评估集运行。
- 日志分析和质量回放。

批处理可以更充分利用模型服务的 batching 能力，也更容易设置重试和断点续跑。

但批处理要注意数据版本。比如同一批文档使用哪个 prompt 版本、哪个模型版本、哪个知识库版本，都要记录下来。否则结果很难复现。

---

# 5. Webhook 与外部回调

Agent 经常要和外部系统协作：支付、工单、消息平台、审批系统、爬虫任务、CI/CD。很多外部系统不是立即返回结果，而是稍后回调。

Webhook 模式下，Agent 创建外部任务后进入 waiting 状态。收到回调后，系统验证签名、更新状态，再恢复 Agent 流程。

关键点包括：

- 回调必须鉴权和验签。
- 回调处理要幂等，避免重复通知导致重复执行。
- 回调要关联 task_id 或 trace_id。
- 长时间无回调要超时处理。
- 回调数据不要直接进入 prompt，要先校验和清洗。

Webhook 把 Agent 从“单次请求”变成“跨系统流程”，因此状态和审计更重要。

---

# 6. 灰度发布与回滚

Agent 发布不只是部署代码，还包括 prompt、模型、工具 schema、检索策略、路由策略和评估规则的变化。

灰度时可以按这些维度逐步放量：

- 用户或租户。
- 流量比例。
- 任务类型。
- 模型版本。
- Prompt 版本。
- 工具版本。

每次灰度都要对比质量、成本、延迟和错误率。如果新版本只是回答更长，但事实性下降或工具错误增加，就不应该继续放量。

回滚也要覆盖非代码配置。很多 Agent 事故来自 prompt 或模型配置变更，而不是代码变更。

---

# 7. 部署环境

常见部署环境包括：

- 本地或内网服务：适合原型、内部工具、私有数据。
- 容器化部署：适合标准 Web 服务和 Worker。
- Kubernetes：适合多服务、高并发、自动扩缩容。
- Serverless：适合轻量、间歇任务，但长任务和冷启动要谨慎。
- GPU 集群：适合自托管模型服务。
- 边缘或本地模型：适合低延迟、隐私或离线场景。

Agent Runtime 和模型服务可以分开部署。即使模型使用外部 API，Agent Runtime 仍然需要自己的部署、状态和观测体系。

---

# 8. 延伸搜索清单

- Server-Sent Events
- WebSocket
- HTTP chunked streaming
- Async workflow
- Webhook
- Task queue
- Batch inference
- Offline evaluation
- Canary release
- Blue-green deployment
- Feature flag
- Prompt versioning
- Model versioning
- Rollback
- Kubernetes
- Serverless
- Worker pool
- Backpressure
