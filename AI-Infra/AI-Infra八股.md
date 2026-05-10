# AI Infra 八股目录

> AI 基础设施 / 平台架构岗的核心面试知识体系，占面试权重的 **50%–60%**，是传统计算机八股的升级版。

---

## 1. GPU 体系结构与算力基础
- 1.1 GPU 硬件：SM/CUDA Core/Tensor Core、显存层次（L1/L2/HBM）、NVLink/NVSwitch、多卡互联拓扑
- 1.2 CUDA 编程模型：Grid/Block/Thread、Warp、共享内存、Bank Conflict、内存合并访问
- 1.3 算子优化：GEMM（cuBLAS）、卷积（cuDNN）、FlashAttention（IO-Awareness）、Triton 自定义算子
- 1.4 混合精度：FP16/BF16/FP8/INT8、Loss Scaling、GradScaler
- 1.5 国产芯片适配：华为昇腾（CANN/NPU）、寒武纪、海光 DCU、性能调优差异

## 2. 分布式训练框架
- 2.1 并行策略：数据并行（DP/DDP/FSDP）、模型并行（Tensor/Pipeline/Sequence）、3D 并行、ZeRO（1/2/3/Offload）
- 2.2 通信原语：AllReduce/AllGather/ReduceScatter/Broadcast、Ring AllReduce、NCCL/RCCL/Gloo
- 2.3 训练框架：Megatron-LM（GPT 系列标配）、DeepSpeed（微软生态）、Colossal-AI、HuggingFace Accelerate
- 2.4 稳定性：Checkpoint 高频保存、故障恢复（Elastic Training）、确定性训练（Deterministic）
- 2.5 超大规模：MoE（Mixture of Experts）、EP（Expert Parallelism）、TP/DP/EP 混合并行

## 3. 模型推理优化与服务化
- 3.1 推理引擎：vLLM（PagedAttention/Prefix Caching）、TensorRT-LLM（Plugin/量化）、TGI、SGLang、llama.cpp
- 3.2 批处理策略：Static Batching vs Continuous Batching（Inflight Batching）、Split-Fuse
- 3.3 量化与压缩：PTQ（Post-Training Quantization）、GPTQ/AWQ、SmoothQuant、KV Cache 量化、模型剪枝/蒸馏
- 3.4 服务化架构：API Gateway、模型路由（Router）、A/B 测试、Canary 发布、多租户隔离
- 3.5 投机解码：Draft Model、Medusa、Lookahead Decoding、SD（Speculative Decoding）
- 3.6 长上下文优化：KV Cache 压缩（H2O、StreamingLLM）、上下文分流、Chunked Prefill

## 4. AI 训练与推理平台
- 4.1 资源调度：Kubernetes + Volcano/Yunikorn、Slurm、Kubeflow、gang-scheduling / coscheduling
- 4.2 GPU 虚拟化：MIG（NVIDIA）、vGPU、OrionX、Time-slicing vs MPS vs MIG
- 4.3 显存与计算优化：显存碎片整理、内存池、计算图优化（Graph Compilation）、算子融合
- 4.4 存储系统：高性能 Checkpoint 存储（CPFS/Lustre/WEKA）、模型权重加载优化（并行加载、延迟加载）
- 4.5 网络优化：RDMA（RoCE v2 / InfiniBand）、集合通信优化、网络拓扑感知调度

## 5. MLOps / LLMOps / AgentOps
- 5.1 模型生命周期：MLflow、WandB、实验管理、模型注册中心（Model Registry）、版本回滚
- 5.2 数据流水线：数据标注、数据版本（DVC）、特征存储（Feature Store）、增量更新
- 5.3 监控与告警：模型漂移（Data/Concept Drift）、性能退化、幻觉检测、Token 消耗监控
- 5.4 AgentOps：编排调度（Airflow/Celery for Agents）、自愈机制（超时/重试/熔断）、状态持久化、上下文管理
- 5.5 可观测性：分布式链路追踪（OpenTelemetry）、日志聚合（ELK/Loki）、指标采集（Prometheus/Grafana）

## 6. 云原生与基础设施
- 6.1 容器运行时：Docker、containerd、Kata Containers、gVisor（用户态内核）、Firecracker（MicroVM）
- 6.2 K8s for AI：Device Plugin（GPU 调度）、Operator 模式、Custom Scheduler、Sidecar 注入
- 6.3 Serverless 与弹性：Knative/OpenFunction、冷启动优化、自动扩缩容（HPA/VPA/KEDA）、Spot/抢占式实例
- 6.4 网络与存储：CNI 插件（Calico/Cilium）、CSI 存储、高性能网络（SR-IOV/DPDK）
- 6.5 安全与隔离：多租户网络隔离、RBAC、镜像安全扫描、机密计算（TEE/SGX）、沙箱执行

## 7. 性能分析与调优
- 7.1 性能剖析：Nsight Systems、PyTorch Profiler、TensorBoard、Roofline Model
- 7.2 瓶颈定位：计算瓶颈 vs 内存瓶颈 vs 通信瓶颈、算子耗时分析、Kernel Launch 开销
- 7.3 优化手段：算子融合、图优化（TorchDynamo/TorchInductor）、编译加速（XLA/TVM/MLIR）
- 7.4 成本优化：GPU 利用率提升、动态批大小、模型蒸馏（小模型替代大模型）、离线推理 vs 在线服务

## 8. AI Infra 系统设计题
- 8.1 设计一个支持千卡训练的 LLM 训练平台（考虑故障恢复、Checkpoint、存储）
- 8.2 设计一个高吞吐低延迟的大模型推理服务（考虑 Continuous Batching、Prefix Caching、多模型路由）
- 8.3 设计一个多租户的 Agent 执行平台（考虑 gVisor 隔离、状态持久化、工具市场、计费）
- 8.4 设计一个模型即服务（MaaS）平台（考虑版本管理、A/B 测试、灰度发布、自动扩缩容）
- 8.5 设计一个面向企业的 RAG 服务平台（考虑文档解析、向量索引、多租户数据隔离、权限）

---

> **面试定位**：AI 平台层核心，AI Infra 工程师 / 平台架构师必考。建议结合系统设计与工程实战加深理解。