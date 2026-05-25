---
tags:
  - LLM
  - RAG
  - Production
  - Advanced-RAG
created: 2026-05-25
description: 生产级 RAG 与进阶模式，讲清权限、版本、缓存、多租户、GraphRAG、Agentic RAG、多模态 RAG 和常见架构取舍
---

> **核心考点**：生产级 RAG 的难点不只是检索准，而是数据可信、权限正确、版本可控、延迟成本可接受、失败可追踪。进阶 RAG 模式也应该服务于具体失败模式，而不是为了堆概念。

---

# 1. 核心问题：从 demo 到生产差在哪里

RAG demo 往往是：

```text
上传几份 PDF -> 向量化 -> 问答
```

生产系统要面对：

- 文档不断更新。
- 用户权限不同。
- 多租户隔离。
- 文档质量参差。
- 检索延迟和成本。
- 引用必须可追溯。
- 错误答案要能复盘。
- 外部资料可能包含攻击。
- 新旧版本不能混用。

所以生产级 RAG 是知识系统、检索系统、LLM 系统和权限系统的组合。

---

# 2. 权限控制：不能让模型看到不该看的资料

RAG 权限要尽量在检索前和检索中处理，而不是让模型自行判断。

错误做法：

```text
把所有文档都检索出来，再告诉模型“不要使用无权限内容”
```

正确思路：

```text
用户身份
  │
  ├─► 权限解析
  ├─► 元数据过滤
  ├─► 只召回可访问文档
  └─► 生成时只看到授权上下文
```

权限维度可能包括：

- tenant_id。
- department。
- role。
- document_acl。
- project_id。
- data_sensitivity。

高敏感内容不应进入 prompt。模型不是权限边界。

---

# 3. 版本和时效性

知识库常见问题：

- 新政策已发布，旧政策仍被召回。
- FAQ 和正式制度冲突。
- 文档更新时间不同。
- 用户问的是某个历史时间点的规则。

解决思路：

- 每个 chunk 保存版本和更新时间。
- 标记 active / deprecated。
- 检索默认过滤旧版本。
- 冲突时优先权威来源。
- 时间敏感问题保留时间条件。

Prompt 中也要告诉模型：

```text
如果资料包含多个版本，优先使用更新时间最新且状态为 active 的资料。
如果用户询问历史时间，请使用对应时间范围内有效的资料。
```

但最终版本选择最好由系统规则辅助，而不是完全靠模型。

---

# 4. 缓存：降低成本但要小心过期

RAG 中可缓存：

- query embedding。
- 检索结果。
- rerank 结果。
- 常见问题答案。
- 文档解析结果。
- chunk embedding。

缓存适合：

- 高频重复问题。
- 文档变化慢。
- 延迟敏感场景。

风险：

- 文档更新后答案过期。
- 用户权限不同导致缓存泄漏。
- 个性化上下文不同但命中同一缓存。

缓存 key 至少应考虑：

```text
query
user/tenant/permission scope
knowledge base version
prompt version
model version
```

---

# 5. 成本和延迟优化

RAG 延迟包括：

```text
query rewrite
embedding
retrieval
reranking
context compression
LLM generation
post validation
```

优化方向：

- 缓存 query embedding。
- 减少无必要的 query rewriting。
- 控制召回候选数。
- reranker 只处理候选 top-N。
- 并行多路召回。
- 对长文档使用 parent-child。
- 对常见问题用 FAQ 直答。
- 小模型做分类和改写，大模型做最终生成。

不要只优化 LLM。很多 RAG 系统的瓶颈在 rerank、网络 IO、文档存储和上下文过长。

---

# 6. GraphRAG：什么时候需要图

普通 RAG 擅长从文档片段中找证据。GraphRAG 尝试把实体和关系构成图：

```text
实体：人、组织、产品、项目、事件
关系：负责、依赖、属于、影响、发生于
```

适合：

- 多跳关系查询。
- 组织知识。
- 项目依赖。
- 合同主体关系。
- 长文档全局总结。
- 问题需要跨多个实体综合。

不适合一开始就上：

- FAQ 问答。
- 简单政策查询。
- 文档量很小。
- 图抽取质量无法保证。

GraphRAG 的难点：

- 实体抽取错误。
- 关系抽取错误。
- 图更新复杂。
- 查询到图再到证据的链路更长。
- 评估更难。

所以先问：你的失败模式是不是普通 chunk retrieval 无法处理的多跳关系？如果不是，GraphRAG 可能过重。

---

# 7. Agentic RAG：让模型参与检索决策

传统 RAG 通常一次检索：

```text
query -> retrieve -> answer
```

Agentic RAG 会让模型决定：

- 是否需要检索。
- 检索什么。
- 是否需要再次检索。
- 是否换检索策略。
- 是否调用其他工具。
- 是否拒答。

适合：

- 多步问题。
- 需要澄清的问题。
- 需要组合多个知识源。
- 检索后发现证据不足，需要追问或二次检索。

风险：

- 延迟增加。
- 成本增加。
- 检索循环。
- 工具调用错误。
- 行为更难评估。

需要配合：

- 最大检索轮数。
- 工具预算。
- 停止条件。
- trace 记录。
- 每步评估。

---

# 8. Multi-modal RAG

多模态 RAG 处理的不只是文本：

- 图片。
- 表格。
- 图表。
- PPT。
- 扫描 PDF。
- 视频帧。
- 音频转写。

核心难点：

- 图片 OCR 是否准确。
- 表格结构是否保留。
- 图表数据如何提取。
- 多模态 embedding 是否适配任务。
- 引用如何回到原始页面或图像区域。

很多场景可以先转成结构化文本：

```text
图片 -> OCR + layout
表格 -> markdown table / records
图表 -> 数据点 + 描述
```

再进入普通 RAG 链路。

---

# 9. Table RAG 和 SQL RAG

表格数据不一定适合切成普通 chunk。

如果用户问：

```text
2025 年 Q4 华东区销售额比 Q3 增长多少？
```

这不是语义检索问题，而是结构化查询和计算问题。

选择：

- 小表：转成 Markdown 表格放入上下文。
- 大表：用 Text-to-SQL 查询。
- 指标系统：调用 BI/API 工具。
- 混合场景：先检索表说明，再生成 SQL。

不要让模型靠自然语言从长表里“看”出精确计算结果。程序更适合计算。

---

# 10. RAG 和 Agent 的组合

RAG 可以是 Agent 的一个工具：

```text
search_policy(query)
search_codebase(query)
search_ticket(query)
search_web(query)
```

Agent 负责决定什么时候用哪个知识源。RAG 负责给出证据。

关键是工具边界：

- 每个检索工具的知识范围是什么。
- 是否有权限过滤。
- 返回多少证据。
- 是否返回引用。
- 工具失败时怎么处理。

不要把所有知识源混成一个“search everything”。这样召回会变乱，权限也难控制。

---

# 11. 常见误区

## 11.1 进阶 RAG 一定比基础 RAG 好

不一定。很多问题用更好的文档处理、hybrid search、reranker 和评估集就能解决。GraphRAG、Agentic RAG 应该服务于明确失败模式。

## 11.2 权限可以交给 prompt

不可以。权限必须由系统控制，未授权内容不应进入模型上下文。

## 11.3 长上下文模型可以替代 RAG

长上下文能放更多资料，但不能替代检索、权限、版本、引用和评估。把所有文档塞进上下文也会带来成本和噪声。

## 11.4 RAG 只要离线评估高就能上线

不够。上线后文档会变、用户问法会变、权限会变，需要监控和失败样本回流。

---

# 12. 面试 Q&A

## Q1：生产级 RAG 最重要的工程问题是什么？

除了检索质量，还包括权限控制、版本管理、增量更新、引用追踪、延迟成本、日志 trace 和失败样本回归。没有这些，demo 很难变成可信系统。

## Q2：什么时候需要 GraphRAG？

当问题经常需要跨实体、多跳关系、全局总结或复杂依赖推理时，GraphRAG 可能有价值。如果只是 FAQ 或政策条款问答，普通 RAG 加 reranker 往往更合适。

## Q3：Agentic RAG 适合什么场景？

适合多步问题、证据不足时需要二次检索、需要多个知识源或工具组合的场景。它更灵活，但成本、延迟和评估复杂度更高。

## Q4：长上下文模型会取代 RAG 吗？

不会完全取代。长上下文解决“放得下”，但 RAG 还解决“找得准、权限对、版本新、可引用、可评估、成本可控”。

---

# 13. 相关链接

- [[README|RAG 目录]]
- [[00-RAG-Overview]]
- [[04-Grounded-Generation-Citations-And-Hallucination-Control]]
- [[05-RAG-Evaluation-And-Observability]]
- [[../Prompt-Engineering/04-Structured-Output-Tool-Use-And-Workflows|结构化输出、工具调用与工作流]]
- [[../LLM-Basic/06-Context-Window-KV-Cache|上下文窗口与 KV Cache]]

