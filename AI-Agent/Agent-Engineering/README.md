---
tags:
  - LLM
  - AI-Agent
  - Agent-Engineering
  - Deployment
  - 学习路线
created: 2026-05-27
description: Agent 工程化与部署基础知识目录，覆盖运行时架构、推理服务、部署模式、状态恢复、安全隔离、可观测性、成本延迟和可靠性
---

# Agent Engineering

> Agent 工程化不是把一个 prompt 包成接口，而是把模型、工具、状态、权限、队列、评估、监控、成本和部署流程组合成一个可运营的软件系统。Demo 关注“能不能跑通”，工程化关注“能不能稳定、可控、可恢复、可迭代地跑下去”。

---

# 1. 学习顺序

建议按下面顺序读：

```text
00 -> 01 -> 02 -> 03 -> 04 -> 05 -> 06 -> 07
```

| 顺序 | 文档 | 应该带走的核心问题 |
|---:|---|---|
| 0 | [[00-Agent-Engineering-Overview|Agent 工程化总览]] | 从 demo 到生产系统，Agent 多了哪些必须补齐的工程能力？ |
| 1 | [[01-Runtime-Architecture-And-Service-Boundaries|运行时架构与服务边界]] | Agent 服务、模型服务、工具服务、状态服务和任务队列如何分层？ |
| 2 | [[02-Model-Serving-And-Inference-Optimization|模型服务与推理优化]] | API 模型、自托管模型、推理引擎、批处理、KV Cache 和量化如何影响部署？ |
| 3 | [[03-Deployment-Patterns-Streaming-And-Async-Workflows|部署模式、流式响应与异步工作流]] | 在线同步、Streaming、异步任务、批处理和 Webhook 分别适合什么场景？ |
| 4 | [[04-State-Persistence-Checkpoint-And-Recovery|状态持久化、Checkpoint 与恢复]] | 对话状态、工作流状态、工具结果和长任务如何保存、恢复和审计？ |
| 5 | [[05-Security-Isolation-Permissions-And-Sandboxing|安全隔离、权限与沙箱]] | Prompt injection、工具越权、代码执行、密钥和沙箱如何治理？ |
| 6 | [[06-Observability-Evaluation-And-Operations|可观测性、评估与运营]] | Agent trace、日志、指标、评估集、告警和回归测试如何组织？ |
| 7 | [[07-Cost-Latency-Scaling-And-Reliability|成本、延迟、扩缩容与可靠性]] | 如何控制 token 成本、响应延迟、限流、重试、降级和容量？ |

如果只想先搭一个可上线的 Agent，优先读 `00、01、04、05、06`。如果要自托管模型或处理高并发，再重点读 `02、03、07`。

---

# 2. 核心主线

一个生产级 Agent 系统可以按这条链路理解：

```text
用户请求
  │
  ├─► 接入层：鉴权、限流、输入校验、会话识别
  ├─► Agent 运行时：规划、工具调用、状态更新、停止条件
  ├─► 模型层：模型路由、推理服务、流式输出、缓存
  ├─► 工具层：Tool Gateway、权限校验、沙箱、外部 API
  ├─► 状态层：会话、任务、checkpoint、审计日志、长期记忆
  ├─► 异步层：队列、后台任务、Webhook、重试和补偿
  ├─► 观测层：trace、日志、指标、成本、质量评估
  └─► 运维层：部署、灰度、回滚、告警、扩缩容、灾备
```

Agent 的工程化难点来自两个事实：

- LLM 输出具有不确定性，不能像普通函数一样完全依赖固定返回。
- Agent 会调用外部工具并改变世界，因此必须有权限、审计、回滚和人工确认。

所以生产 Agent 的核心不是“更复杂的 prompt”，而是“把不确定的模型行为放进确定性的工程边界里”。

---

# 3. 与其他目录的边界

- [[../Agent-Corepart/README|Agent 核心组件]]讲规划、记忆、工具、状态、执行和评估这些组件本身。
- [[../Agent-Frame/README|Agent 开发框架]]讲 LangChain、LangGraph、AutoGen、CrewAI 等框架如何组织 Agent。
- [[../Multi-Agent/README|Multi-Agent]]讲多个 Agent 之间如何分工、通信、冲突裁决和协作治理。
- Agent Engineering 讲这些能力如何上线：服务边界、部署、状态、沙箱、监控、成本、扩缩容和恢复。
- [[../RAG/README|RAG]]、[[../Prompt-Engineering/README|Prompt Engineering]]、[[../Fine-tuning/README|Fine-tuning]]提供模型能力建设方法，但上线后仍需要工程化治理。

一句话区分：

> Agent-Corepart 讲“Agent 怎么工作”，Agent-Frame 讲“用什么框架组织”，Agent-Engineering 讲“如何把它稳定部署和长期运营”。

---

# 4. 复习思维导图

```text
Agent Engineering
  ├─ 架构
  │   ├─ 接入层
  │   ├─ Agent Runtime
  │   ├─ Model Gateway
  │   ├─ Tool Gateway
  │   ├─ State Store
  │   └─ Queue / Worker
  ├─ 部署
  │   ├─ API 模型
  │   ├─ 自托管模型
  │   ├─ 在线同步
  │   ├─ Streaming
  │   ├─ 异步任务
  │   └─ 批处理
  ├─ 状态
  │   ├─ 会话状态
  │   ├─ 任务状态
  │   ├─ Checkpoint
  │   ├─ 工具结果
  │   └─ 审计日志
  ├─ 安全
  │   ├─ 权限最小化
  │   ├─ Prompt injection 防护
  │   ├─ Tool Gateway
  │   ├─ 沙箱隔离
  │   ├─ 密钥管理
  │   └─ 人工审批
  ├─ 运营
  │   ├─ Trace
  │   ├─ Metrics
  │   ├─ Logs
  │   ├─ Evaluation
  │   ├─ Alerting
  │   └─ Rollback
  └─ 成本可靠性
      ├─ 模型路由
      ├─ 缓存
      ├─ 限流
      ├─ 重试
      ├─ 降级
      └─ 扩缩容
```

---

# 5. 面试复习地图

时间有限时，优先掌握这些问题：

1. Agent 从 demo 到生产系统，必须补哪些工程能力？
2. Agent Runtime、Model Gateway、Tool Gateway、State Store 分别负责什么？
3. 自托管模型和直接调用模型 API 的工程取舍是什么？
4. Streaming、异步任务、批处理和 Webhook 如何选择？
5. 为什么 Agent 必须保存结构化状态，而不能只保存聊天记录？
6. Checkpoint 如何支持长任务恢复和人工介入？
7. Tool calling 为什么必须经过权限校验、参数校验和审计？
8. 代码执行类工具为什么需要沙箱，Docker/gVisor/Firecracker/Wasm 的定位是什么？
9. Agent 可观测性为什么要看 trace，而不只是日志？
10. 如何控制 Agent 的 token 成本、延迟、并发和失败率？

---

# 6. 延伸搜索清单

下面这些先不用背，主线理解后再搜索：

- Agent runtime
- Model gateway
- Tool gateway
- vLLM
- PagedAttention
- Continuous batching
- TGI
- TensorRT-LLM
- SGLang
- llama.cpp
- KV Cache
- Speculative decoding
- Streaming response
- Webhook
- Durable execution
- Checkpointing
- Redis session store
- Temporal workflow
- Celery
- Kafka
- Docker sandbox
- gVisor
- Firecracker
- WebAssembly sandbox
- LangSmith
- Langfuse
- Phoenix
- OpenTelemetry
- LLM evaluation
- Model routing
- Semantic cache
- Circuit breaker
- Rate limiting
