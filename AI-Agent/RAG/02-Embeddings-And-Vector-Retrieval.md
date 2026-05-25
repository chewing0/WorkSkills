---
tags:
  - LLM
  - RAG
  - Embedding
  - Vector-Search
created: 2026-05-25
description: Embedding 与向量检索基础，讲清语义向量、相似度、ANN 索引、向量数据库、embedding 模型选择和 dense retrieval 的边界
---

> **核心考点**：Embedding 把文本映射到向量空间，让语义相近的文本距离更近。向量检索擅长找“意思相近”，但不擅长精确匹配数字、符号、专有名词和权限条件。

---

# 1. 核心问题：Embedding 到底表示什么

Embedding 模型把文本变成一个向量：

```text
"跨城市打车报销条件" -> [0.13, -0.02, 0.87, ...]
```

向量不是人工定义的标签，而是模型从大量文本中学到的语义表示。理想情况下，意思相近的文本在向量空间中更接近：

```text
"跨城市打车能报销吗"
"异地出行出租车费用是否可以报销"
```

这两个句子字面不同，但语义接近，dense retrieval 可以把它们匹配起来。

RAG 里 embedding 常用于：

- 文档 chunk 向量化。
- 用户 query 向量化。
- 计算 query 和 chunk 的相似度。
- 召回 top-k 候选片段。

---

# 2. 向量相似度：如何判断接近

常见相似度：

## 2.1 Cosine Similarity

衡量向量方向是否接近：

$$
\cos(\theta)=\frac{a \cdot b}{\|a\|\|b\|}
$$

很多文本 embedding 检索使用 cosine。

## 2.2 Dot Product

直接点积：

$$
score = a \cdot b
$$

如果向量已归一化，dot product 和 cosine 排序接近。

## 2.3 Euclidean Distance

计算距离：

$$
\|a-b\|
$$

向量库会根据索引类型和模型建议选择不同度量。

关键不是背公式，而是理解：

> embedding retrieval 本质是在向量空间里找和 query 表示最接近的 chunk。

---

# 3. Dense Retrieval 的优势

相比关键词检索，向量检索的优势是语义泛化。

用户问：

```text
员工离职后还能访问系统多久？
```

文档写：

```text
账号将在劳动关系终止后的 24 小时内停用。
```

关键词重合不多，但语义相关。dense retrieval 可能召回。

适合：

- 同义改写。
- 模糊问法。
- 自然语言 FAQ。
- 概念性问题。
- 跨语言或多表达方式场景。

---

# 4. Dense Retrieval 的边界

向量检索容易漏掉这些：

## 4.1 精确标识符

```text
ERR_CONN_1045
SKU-8A391
订单号 A20260525001
```

这些不是语义问题，而是精确匹配问题。

## 4.2 数字和范围

```text
报销上限 300 元
质保期 180 天
版本 2.13.4
```

embedding 可能知道“报销上限”相关，但不一定对数字精确敏感。

## 4.3 否定和细微差别

```text
可以报销
不可以报销
仅审批后可以报销
```

这些句子语义非常接近，但业务含义差别巨大。

## 4.4 长 chunk 混合语义

一个 chunk 同时包含多个主题时，它的向量会变成混合表示，检索结果可能看似相关但不精确。

因此 dense retrieval 常常要和关键词检索、元数据过滤、reranker 配合。

---

# 5. 向量检索的基本流程

```text
离线：
chunk -> embedding -> vector index

在线：
query -> embedding -> nearest neighbors -> top-k chunks
```

伪代码：

```text
query_vec = embed(user_query)
candidates = vector_index.search(query_vec, top_k=50)
```

注意：向量检索返回的 top-k 只是候选，不等于最终上下文。通常还要重排、过滤、去重和压缩。

---

# 6. ANN：为什么需要近似最近邻

如果有 1 亿个 chunk，逐个计算相似度太慢。向量数据库通常用 ANN（Approximate Nearest Neighbor，近似最近邻）索引。

常见思路：

- HNSW：图索引，通过邻近图快速搜索。
- IVF：先聚类，再只搜索相关簇。
- PQ：向量压缩，降低内存和计算。

ANN 的本质是用一点召回损失换速度和内存：

```text
更快查询
  ↔
可能漏掉真正最近的向量
```

所以向量库参数也要评估。比如 HNSW 的搜索深度调高，召回更好但更慢。

---

# 7. 向量数据库到底做什么

向量数据库通常提供：

- 向量写入。
- ANN 索引。
- top-k 查询。
- 元数据过滤。
- 分区或 collection。
- 删除和更新。
- 混合检索或全文检索。
- 水平扩展。

它不负责：

- 判断文档是否可信。
- 自动选择最佳 chunk。
- 保证答案正确。
- 替你处理权限语义。
- 替模型生成忠实引用。

选型时不要只看 QPS，还要看：

- 数据规模。
- 更新频率。
- 元数据过滤能力。
- 混合检索支持。
- 多租户隔离。
- 备份和恢复。
- 运维复杂度。
- 成本。

---

# 8. Embedding 模型怎么选

看这些维度：

| 维度 | 问题 |
|---|---|
| 语言 | 中文、英文、多语言是否都好 |
| 领域 | 是否适合代码、法律、医疗、客服 |
| 维度 | 向量维度影响存储和检索成本 |
| 上下文长度 | 单次能 embed 多长文本 |
| 查询文档匹配 | 是否区分 query encoder 和 doc encoder |
| 归一化 | 相似度计算方式是否匹配 |
| 延迟成本 | 在线 query embedding 是否够快 |
| 部署 | API 还是本地模型 |

不要只看公开榜单。最好构建自己的检索评估集：

```text
query -> 应该召回的 document/chunk
```

然后比较 Recall@k、MRR、nDCG。

---

# 9. Query 和 Document 不一定同分布

用户 query 通常短、口语化、不完整：

```text
离职账号多久关？
```

文档 chunk 通常正式、完整：

```text
员工离职后，IT 系统账号将在劳动关系终止后的 24 小时内停用。
```

这叫 query-document mismatch。

解决方式：

- 使用专门面向检索训练的 embedding 模型。
- 对 query 做改写或扩展。
- 用 HyDE 生成假设答案再检索。
- 多路召回。
- 用 reranker 重排。

---

# 10. 常见误区

## 10.1 向量相似度高就一定能回答

不一定。相似片段可能只是主题相关，但不包含答案证据。RAG 需要 evidence，不只是 relevance。

## 10.2 embedding 模型越大越好

不一定。还要看领域、语言、延迟、成本和部署约束。小模型在垂直数据上可能更实用。

## 10.3 换向量库能解决检索质量

通常不能。向量库主要影响性能和索引能力。质量问题更多来自文档处理、embedding、query、重排和评估。

## 10.4 top-1 分数可以当置信度

不能直接当。相似度分数受模型、归一化、chunk 长度和索引影响，不等于答案正确概率。

---

# 11. 面试 Q&A

## Q1：Embedding 在 RAG 中的作用是什么？

Embedding 把 query 和文档 chunk 映射到向量空间，通过相似度找语义相关内容。它解决了字面不匹配的问题，但不保证精确事实和引用正确。

## Q2：Dense retrieval 和 BM25 的差异是什么？

Dense retrieval 看语义相似，适合同义表达和模糊问法；BM25 看词项匹配，适合关键词、编号、错误码、专有名词和精确短语。生产系统常用 hybrid search。

## Q3：ANN 为什么是近似的？

大规模向量逐个精确比较太慢，ANN 用图、聚类或压缩加速搜索。它牺牲少量召回换查询速度和内存效率。

## Q4：如何评估 embedding 模型是否适合业务？

构建业务 query 和标准相关文档集合，评估 Recall@k、MRR、nDCG，并观察失败样本。不要只看通用榜单。

---

# 12. 相关链接

- [[README|RAG 目录]]
- [[01-Document-Processing-And-Indexing]]
- [[03-Hybrid-Retrieval-Reranking-And-Query-Rewriting]]
- [[05-RAG-Evaluation-And-Observability]]
- [[../LLM-Basic/02-Tokenizer-Embedding|Tokenizer 与 Embedding]]

