# Agent 八股目录

> AI Agent 应用开发岗的核心面试知识体系。

---

## 1. 大语言模型（LLM）基础
- [[01-Transformer]]：Self-Attention、Multi-Head Attention、Position Encoding（正弦/可学习/RoPE/ALiBi）、FFN、LayerNorm/PreNorm
- 1.2 主流模型差异：GPT 系列（Decoder-only）、BERT（Encoder-only）、T5（Encoder-Decoder）、Llama/Qwen/DeepSeek 架构特点
- 1.3 生成策略：Greedy/Beam Search/Top-k/Top-p（Nucleus Sampling）、Temperature、Repetition Penalty
- 1.4 长度外推：ALiBi、NTK-aware RoPE、YaRN、位置插值
- 1.5 上下文窗口：长上下文建模挑战、滑动窗口注意力、Ring Attention

## 2. 提示词工程（Prompt Engineering）
- 2.1 基础技巧：Zero-shot/Few-shot、System Prompt 设计、角色扮演、输出格式约束（JSON/XML）
- 2.2 高级推理：CoT（Chain-of-Thought）、CoT-SC（Self-Consistency）、ToT（Tree-of-Thoughts）、GoT（Graph-of-Thoughts）
- 2.3 Agent 范式：ReAct（Reasoning + Acting）、Plan-and-Solve、Reflexion（自我反思）、LATS
- 2.4 Prompt 安全：Prompt 注入攻击、防御策略（分隔符/指令优先级/输出过滤）、Prompt 越狱

## 3. RAG（检索增强生成）
- 3.1 架构链路：Indexing → Retrieval → Reranking → Generation
- 3.2 文档处理：PDF/Word 解析、文本切分策略（固定长度/语义/递归/按标题）、元数据提取
- 3.3 Embedding 模型：OpenAI Ada、BGE、M3E、GTE、E5、ColBERT、Late Interaction
- 3.4 向量数据库：Milvus、Pinecone、Weaviate、Chroma、Qdrant、PGVector；选型对比（性能/成本/生态）
- 3.5 检索策略：Dense Retrieval、Sparse Retrieval（BM25）、Hybrid Search、多路召回
- 3.6 重排序：Cross-Encoder、bge-reranker、Cohere Rerank、排序融合（RRF）
- 3.7 幻觉治理：事实性校验、引用溯源、置信度评分、拒绝回答机制
- 3.8 评估指标：Context Precision/Recall、Faithfulness、Answer Relevancy、RAGAS 框架

## 4. 模型微调（Fine-tuning）
- 4.1 全量微调：SFT 数据构造、指令格式（Alpaca/ShareGPT）、过拟合与灾难性遗忘
- 4.2 PEFT 方法：LoRA 原理（低秩分解）、QLoRA（4-bit 量化 + 分页优化器）、Prefix Tuning、P-Tuning、Adapter
- 4.3 对齐技术：RLHF（PPO 算法）、DPO（直接偏好优化）、KTO、ORPO
- 4.4 训练框架：HuggingFace PEFT、DeepSpeed（ZeRO-1/2/3/Offload）、Megatron-LM、Unsloth、Llama-Factory
- 4.5 评估：Perplexity、BLEU/ROUGE、人工评估、LLM-as-a-Judge

## 5. Agent 核心组件
- 5.1 规划（Planning）：任务拆解、目标分解、动态规划、回溯机制
- 5.2 记忆（Memory）：短期记忆（Buffer/滑动窗口）、长期记忆（Vector Store）、实体记忆、摘要记忆、记忆检索与更新
- 5.3 工具使用（Tool Use）：Function Calling 机制、Schema 设计、工具选择策略、工具调用链、错误处理与重试
- 5.4 行动（Action）：API 调用、代码执行、文件操作、多模态输出

## 6. Agent 开发框架
- 6.1 LangChain：Chain、Agent、Memory、Tool、Callback、LCEL（表达式语言）
- 6.2 LangGraph：状态图、节点/边、循环、条件分支、持久化（Persistence）、Human-in-the-Loop
- 6.3 AutoGen：ConversableAgent、GroupChat、UserProxyAgent、代码执行环境
- 6.4 CrewAI：Agent 角色定义、Task、Process（Sequential/Hierarchical）、工具集成
- 6.5 其他：Dify/Coze（低代码）、MetaGPT（多 Agent 软件公司）、AutoGPT、OpenManus、Manus
- 6.6 选型对比：何时用 LangGraph vs AutoGen vs CrewAI？各框架的优劣势与适用场景

## 7. 多智能体系统（Multi-Agent）
- 7.1 协作模式：主从式（Orchestrator-Worker）、扁平协作、层级架构、议会式（Debate）
- 7.2 通信机制：消息传递、共享记忆、黑板系统、发布订阅
- 7.3 任务分配：静态分配 vs 动态分配、负载感知调度
- 7.4 冲突解决：投票机制、优先级覆盖、协商协议
- 7.5 协议：MCP（Model Context Protocol）三层架构、A2A（Agent-to-Agent）协议、Agent Card、任务状态管理

## 8. Agent 工程化与部署
- 8.1 推理引擎：vLLM（PagedAttention/Continuous Batching）、TGI、TensorRT-LLM、SGLang、llama.cpp
- 8.2 部署模式：离线批处理、在线实时、流式（Streaming）、Webhook 回调
- 8.3 状态管理：对话状态持久化、Checkpoint 与恢复、分布式会话（Redis/DB）
- 8.4 安全隔离：Docker、gVisor、Firecracker、Wasm（WebAssembly）沙箱
- 8.5 可观测性：LangSmith、Langfuse、Phoenix、PromptLayer；追踪思考链、工具调用、Token 消耗
- 8.6 成本控制：模型路由（小/大模型分流）、缓存策略、Prompt 压缩、批量请求

## 9. Agent 面试高频场景题
- 9.1 设计一个智能客服 Agent（带订单查询 / 退换货）
- 9.2 设计一个多 Agent 协作的写作 / 编程团队
- 9.3 Agent 陷入死循环 / 工具调用失败 / 幻觉严重时如何处理？
- 9.4 如何实现 Agent 的长期记忆与个性化？
- 9.5 设计一个支持万级并发的 Agent 服务平台

---

> **面试定位**：AI 应用层核心，Agent 开发工程师 / AI 应用工程师必考。建议结合项目实战加深理解。