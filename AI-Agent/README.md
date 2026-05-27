# Agent 八股目录

> AI Agent 应用开发岗的核心面试知识体系。

---

## 1. 大语言模型（LLM）基础
- [[LLM-Basic/README|LLM 基础目录]]：学习顺序、面试复习地图、与其他目录的边界
- [[LLM-Basic/00-LLM-Overview|LLM 总览]]：从 token 到 logits 的完整链路、训练与推理区别、能力边界
- [[LLM-Basic/01-Transformer|Transformer]]：Self-Attention、Multi-Head Attention、位置编码、FFN、Norm、KV Cache
- [[LLM-Basic/02-Tokenizer-Embedding|Tokenizer 与 Embedding]]：BPE、WordPiece、SentencePiece、特殊 token、词表与 token 成本
- [[LLM-Basic/03-LLM-Variants|主流模型架构差异]]：GPT、BERT、T5、Llama、Qwen、DeepSeek、Dense vs MoE
- [[LLM-Basic/04-Training-Objectives|训练目标与对齐]]：预训练、MLM、Causal LM、SFT、RLHF、DPO、数据质量
- [[LLM-Basic/05-Generation-Strategies|生成策略]]：Greedy、Beam Search、Top-k、Top-p、Temperature、惩罚项、结构化输出
- [[LLM-Basic/06-Context-Window-KV-Cache|上下文窗口与 KV Cache]]：prefill、decode、长上下文成本、上下文管理
- [[LLM-Basic/07-Length-Extrapolation|长度外推]]：RoPE、ALiBi、NTK-aware、YaRN、位置插值、滑动窗口注意力
- [[LLM-Basic/08-Inference-Optimization|推理优化]]：FlashAttention、PagedAttention、连续批处理、量化、投机解码
- [[LLM-Basic/09-Scaling-Laws|Scaling Laws]]：参数量、数据量、算力、Chinchilla、涌现能力、模型路由
- [[LLM-Basic/10-Model-Evaluation|模型评估]]：Perplexity、Benchmark、RAG/Agent 评估、LLM-as-a-Judge、线上监控

## 2. 提示词工程（Prompt Engineering）
- [[Prompt-Engineering/README|Prompt Engineering 目录]]：学习顺序、核心主线、面试复习地图、延伸搜索清单
- [[Prompt-Engineering/00-Prompt-Engineering-Overview|Prompt Engineering 总览]]：prompt 为什么有效、工程位置、能力边界、最小迭代闭环
- [[Prompt-Engineering/01-Prompt-Structure-And-Instructions|Prompt 结构与指令设计]]：任务说明、角色职责、上下文分区、规则约束、输出契约
- [[Prompt-Engineering/02-Context-Examples-And-Task-Framing|上下文、示例与任务表述]]：Zero-shot、Few-shot、示例选择、上下文组织、上下文污染
- [[Prompt-Engineering/03-Reasoning-Decomposition-And-Planning|推理、拆解与规划]]：CoT、任务拆解、Plan-and-Solve、自检、Self-Consistency、ToT/GoT
- [[Prompt-Engineering/04-Structured-Output-Tool-Use-And-Workflows|结构化输出、工具调用与工作流]]：JSON/Schema、Function Calling、校验重试、ReAct、Agent 工作流
- [[Prompt-Engineering/05-Prompt-Safety-Evaluation-And-Iteration|Prompt 安全、评估与迭代]]：Prompt 注入、指令层级、权限控制、黄金集、版本管理

## 3. RAG（检索增强生成）
- [[RAG/README|RAG 目录]]：学习顺序、核心主线、面试复习地图、延伸搜索清单
- [[RAG/00-RAG-Overview|RAG 总览]]：检索增强生成的完整链路、能力边界、失败模式和工程定位
- [[RAG/01-Document-Processing-And-Indexing|文档处理与索引构建]]：解析清洗、chunk、overlap、元数据、多索引和增量更新
- [[RAG/02-Embeddings-And-Vector-Retrieval|Embedding 与向量检索]]：语义向量、相似度、ANN、向量数据库、embedding 模型选择
- [[RAG/03-Hybrid-Retrieval-Reranking-And-Query-Rewriting|混合检索、重排与查询改写]]：BM25、Hybrid Search、RRF、Reranker、Multi-query、HyDE
- [[RAG/04-Grounded-Generation-Citations-And-Hallucination-Control|基于证据生成、引用与幻觉治理]]：上下文构造、忠实性、引用准确、拒答、RAG 注入防护
- [[RAG/05-RAG-Evaluation-And-Observability|RAG 评估与可观测性]]：检索指标、上下文指标、忠实性、引用评估、trace 和线上监控
- [[RAG/06-Production-RAG-And-Advanced-Patterns|生产级 RAG 与进阶模式]]：权限、版本、缓存、GraphRAG、Agentic RAG、多模态和表格 RAG

## 4. 模型微调（Fine-tuning）
- [[Fine-tuning/README|Fine-tuning 目录]]：学习顺序、核心主线、面试复习地图、延伸搜索清单
- [[Fine-tuning/00-Fine-Tuning-Overview|Fine-tuning 总览]]：微调定位、适用边界、主要类型、完整流程、常见失败模式
- [[Fine-tuning/01-Data-Design-And-Formatting|数据设计与格式]]：数据质量、指令格式、chat template、loss mask、数据划分、合成数据
- [[Fine-tuning/02-SFT-And-Instruction-Tuning|SFT 与指令微调]]：监督微调目标、assistant loss、训练参数、过拟合、灾难性遗忘
- [[Fine-tuning/03-PEFT-LoRA-QLoRA-And-Adapters|PEFT、LoRA、QLoRA 与 Adapter]]：低秩适配、关键超参、量化训练、合并部署和适用边界
- [[Fine-tuning/04-Preference-Alignment-RLHF-DPO-And-RM|偏好对齐、RLHF、DPO 与 Reward Model]]：偏好数据、奖励模型、PPO、DPO、KTO/ORPO 和对齐风险
- [[Fine-tuning/05-Training-Systems-And-Optimization|训练工程与优化]]：显存构成、混合精度、优化器、ZeRO/FSDP、checkpoint、稳定性排查
- [[Fine-tuning/06-Evaluation-Deployment-And-Iteration|评估、部署与迭代]]：离线评估、回归测试、安全评估、灰度上线、监控和版本管理

## 5. Agent 核心组件
- [[Agent-Corepart/README|Agent 核心组件目录]]：学习顺序、核心主线、面试复习地图、延伸搜索清单
- [[Agent-Corepart/00-Agent-Core-Overview|Agent 核心总览]]：Agent 最小闭环、核心组件、适用边界、常见失败模式
- [[Agent-Corepart/01-Planning-And-Task-Decomposition|规划与任务拆解]]：目标理解、任务分解、ReAct、Plan-and-Execute、重规划、停止条件
- [[Agent-Corepart/02-Memory-And-Context-Management|记忆与上下文管理]]：短期记忆、工作状态、长期记忆、摘要、实体记忆、记忆检索
- [[Agent-Corepart/03-Tool-Use-And-Function-Calling|工具使用与 Function Calling]]：工具选择、schema 设计、参数校验、工具结果处理、错误恢复
- [[Agent-Corepart/04-Action-Execution-And-Sandbox|行动执行与安全边界]]：API 调用、代码执行、文件操作、权限、幂等、沙箱、审计
- [[Agent-Corepart/05-State-Workflow-And-Human-In-The-Loop|状态、工作流与 Human-in-the-Loop]]：状态管理、checkpoint、状态机、人工确认、异步任务恢复
- [[Agent-Corepart/06-Agent-Evaluation-Safety-And-Observability|Agent 评估、安全与可观测性]]：任务成功率、过程指标、trace、循环检测、工具安全、成本监控

## 6. Agent 开发框架
- [[Agent-Frame/README|Agent 开发框架目录]]：学习顺序、核心主线、面试复习地图、延伸搜索清单
- [[Agent-Frame/00-Agent-Framework-Overview|Agent 框架总览]]：框架定位、常见类型、适用边界、选型主线
- [[Agent-Frame/01-Framework-Abstractions-And-Design|框架通用抽象与设计]]：Model、Prompt、Chain/Runnable、Tool、Memory/State、Graph、Callback、Parser
- [[Agent-Frame/02-LangChain-And-LCEL|LangChain 与 LCEL]]：LangChain 模块、LCEL Runnable 编排、RAG/Tool 集成、适用边界
- [[Agent-Frame/03-LangGraph-Stateful-Agents|LangGraph 与状态图 Agent]]：StateGraph、节点/边、条件分支、循环、checkpoint、Human-in-the-loop
- [[Agent-Frame/04-Multi-Agent-Frameworks-AutoGen-CrewAI-MetaGPT|多 Agent 框架：AutoGen、CrewAI、MetaGPT]]：多角色协作、对话协议、任务流程、协作风险
- [[Agent-Frame/05-Low-Code-Agent-Platforms-And-Ecosystem|低代码 Agent 平台与生态]]：Dify、Coze、Flowise、平台优势、限制和适用场景
- [[Agent-Frame/06-Framework-Selection-Engineering-And-Migration|框架选型、工程落地与迁移]]：生产能力清单、框架锁定、迁移策略、手写 workflow 取舍

## 7. 多智能体系统（Multi-Agent）
- [[Multi-Agent/README|Multi-Agent 目录]]：学习顺序、核心主线、复习思维导图、面试地图、延伸搜索清单
- [[Multi-Agent/00-Multi-Agent-Overview|Multi-Agent 总览]]：多智能体价值、适用边界、最小构成、失败模式和设计原则
- [[Multi-Agent/01-Collaboration-Patterns-And-Architectures|协作模式与系统架构]]：Orchestrator-Worker、层级式、扁平协作、Debate/Critic、Blackboard、Handoff
- [[Multi-Agent/02-Communication-Protocols-And-Shared-State|通信协议、消息与共享状态]]：消息格式、上下文边界、共享状态、发布订阅、任务状态、MCP/A2A
- [[Multi-Agent/03-Task-Decomposition-Allocation-And-Orchestration|任务拆解、分配与调度]]：任务图、静态分配、动态路由、并行合并、预算调度、停止条件
- [[Multi-Agent/04-Conflict-Consensus-And-Quality-Control|冲突、共识与质量控制]]：投票、Judge、Generator-Critic、证据约束、群体幻觉、质量门禁
- [[Multi-Agent/05-Memory-Knowledge-And-Tool-Coordination|记忆、知识与工具协同]]：私有/共享记忆、多 Agent RAG、工具权限、资源锁、行动协议
- [[Multi-Agent/06-Production-Multi-Agent-Safety-Cost-And-Evaluation|生产治理、安全、成本与评估]]：安全边界、成本延迟、trace、回归评估、渐进落地

## 8. Agent 工程化与部署
- [[Agent-Engineering/README|Agent Engineering 目录]]：学习顺序、核心主线、复习思维导图、面试地图、延伸搜索清单
- [[Agent-Engineering/00-Agent-Engineering-Overview|Agent 工程化总览]]：从 demo 到生产系统需要补齐的架构、状态、安全、评估、部署和运营能力
- [[Agent-Engineering/01-Runtime-Architecture-And-Service-Boundaries|运行时架构与服务边界]]：接入层、Agent Runtime、Model Gateway、Tool Gateway、状态服务、队列和业务系统边界
- [[Agent-Engineering/02-Model-Serving-And-Inference-Optimization|模型服务与推理优化]]：API 模型、自托管模型、推理引擎、KV Cache、批处理、量化和模型路由
- [[Agent-Engineering/03-Deployment-Patterns-Streaming-And-Async-Workflows|部署模式、流式响应与异步工作流]]：在线同步、Streaming、异步任务、批处理、Webhook、灰度和回滚
- [[Agent-Engineering/04-State-Persistence-Checkpoint-And-Recovery|状态持久化、Checkpoint 与恢复]]：会话状态、任务状态、工作流状态、工具结果、长期记忆、幂等和恢复
- [[Agent-Engineering/05-Security-Isolation-Permissions-And-Sandboxing|安全隔离、权限与沙箱]]：Prompt injection、工具权限、参数校验、代码执行沙箱、密钥管理、审批和审计
- [[Agent-Engineering/06-Observability-Evaluation-And-Operations|可观测性、评估与运营]]：trace、日志、指标、评估集、线上监控、告警、回归测试和版本管理
- [[Agent-Engineering/07-Cost-Latency-Scaling-And-Reliability|成本、延迟、扩缩容与可靠性]]：token 预算、缓存、模型路由、限流、重试、熔断、降级、容量规划和 SLA

## 9. Agent 面试高频场景题
- 9.1 设计一个智能客服 Agent（带订单查询 / 退换货）
- 9.2 设计一个多 Agent 协作的写作 / 编程团队
- 9.3 Agent 陷入死循环 / 工具调用失败 / 幻觉严重时如何处理？
- 9.4 如何实现 Agent 的长期记忆与个性化？
- 9.5 设计一个支持万级并发的 Agent 服务平台

---

> **面试定位**：AI 应用层核心，Agent 开发工程师 / AI 应用工程师必考。建议结合项目实战加深理解。
