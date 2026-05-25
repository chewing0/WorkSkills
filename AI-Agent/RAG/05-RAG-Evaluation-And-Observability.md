---
tags:
  - LLM
  - RAG
  - Evaluation
  - Observability
created: 2026-05-25
description: RAG 评估与可观测性基础，讲清检索指标、上下文指标、生成忠实性、引用准确率、端到端评估和线上监控
---

> **核心考点**：RAG 评估不是只看最终答案好不好。必须把失败拆到检索、上下文、生成、引用和线上行为各层，否则不知道该改 embedding、chunk、reranker、prompt 还是知识库。

---

# 1. 核心问题：RAG 评估要定位失败环节

同一个错误答案，可能有不同原因。

用户问：

```text
跨城市打车能否报销？
```

系统答错，可能是：

- 文档没入库。
- chunk 切断了审批条件。
- 检索没召回正确条款。
- reranker 把 FAQ 排到政策前面。
- prompt 没要求基于证据。
- 模型忽略了“提前审批”条件。
- 引用指向了错误片段。

如果只看最终答案，就不知道该改哪里。

RAG 评估要拆层：

```text
检索是否找到了证据？
  │
  ├─► 上下文是否包含可用证据？
  ├─► 生成是否忠实于证据？
  ├─► 引用是否支持结论？
  └─► 用户是否真的解决问题？
```

---

# 2. 构建 RAG 评估集

评估集最基本结构：

```text
query
expected_answer 或 scoring rubric
golden_docs / golden_chunks
metadata filters
should_refuse
```

样本来源：

- 真实用户日志。
- 高频搜索词。
- 客服工单。
- 历史失败案例。
- 专家设计问题。
- 权限边界问题。
- 多版本冲突问题。
- 注入攻击样本。

RAG 评估集不应只放简单问答。真正有价值的是边界样本：

- 文档中没有答案。
- 多个文档冲突。
- 需要多个证据合并。
- 问题包含精确数字或型号。
- 需要根据时间过滤。
- 用户无权限访问答案。

---

# 3. 检索评估：正确证据有没有被找回

常见指标：

| 指标 | 含义 |
|---|---|
| Recall@k | 标准相关文档是否出现在前 k 个结果里 |
| Precision@k | 前 k 个结果中有多少相关 |
| MRR | 第一个相关结果排在多前 |
| nDCG | 排名质量，越相关越靠前越好 |
| Hit Rate | 是否至少命中一个相关文档 |

RAG 中最常先看 Recall@k。

原因：

> 如果正确证据没被召回，后面的 reranker 和 LLM 都救不回来。

但 Recall@k 高不代表最终答案好。因为 top-k 里可能包含大量噪声，也可能相关但不支持结论。

---

# 4. 上下文评估：放进 prompt 的材料是否有用

检索候选不等于最终上下文。经过 rerank、过滤、压缩后，放进 prompt 的材料可能变化很大。

要评估：

- 上下文是否包含答案证据。
- 是否包含无关噪声。
- 是否包含冲突资料。
- 是否包含过期版本。
- 是否保留标题和来源。
- 是否超出权限。

常见指标：

| 指标 | 关注点 |
|---|---|
| Context Recall | 答案所需证据是否在上下文里 |
| Context Precision | 上下文中有多少是相关证据 |
| Noise Ratio | 无关片段比例 |
| Conflict Rate | 上下文中冲突材料比例 |

上下文质量差时，应该优先改 rerank、过滤、去重、chunk 和压缩，而不是直接换模型。

---

# 5. 生成评估：答案是否正确且忠实

生成层关注：

| 维度 | 问题 |
|---|---|
| Answer Correctness | 最终答案是否正确 |
| Faithfulness | 答案是否被上下文支持 |
| Completeness | 是否覆盖用户所有约束 |
| Refusal Quality | 资料不足时是否正确拒答 |
| Format | 输出格式是否符合要求 |

Faithfulness 很关键。一个答案可能“常识上正确”，但没有被当前资料支持，这在 RAG 中仍然有问题。

例如：

```text
资料只说“市内交通可报销”
答案却说“跨城市打车也可报销”
```

这就是不忠实。

---

# 6. 引用评估：引用是否支撑结论

引用评估要看：

- 答案中的每个关键 claim 是否有引用。
- 引用片段是否真的支持 claim。
- 引用是否来自用户有权限的文档。
- 引用是否是最新版本。
- 多证据 claim 是否引用完整。

可以把答案拆成 claim：

```text
claim 1：跨城市打车可以报销
claim 2：需要提前审批
claim 3：审批人为部门负责人
```

然后检查每个 claim 的 evidence。

引用不是“答案末尾放几个来源”，而是 claim-evidence 对齐。

---

# 7. 端到端评估：用户看到的结果是否有用

最终还要看端到端效果：

- 是否解决用户问题。
- 是否简洁可读。
- 是否正确拒答。
- 是否带可追溯引用。
- 是否给出下一步行动。
- 延迟是否可接受。
- 成本是否可接受。

端到端评估可以用：

- 人工评分。
- LLM-as-a-Judge。
- 用户反馈。
- A/B 测试。
- 任务成功率。

但端到端分数不能替代分层指标。端到端告诉你“坏了”，分层指标告诉你“坏在哪里”。

---

# 8. LLM-as-a-Judge 怎么用

Judge 可以评估开放问答，但要给 rubric。

示例：

```text
请根据用户问题、上下文和答案评分：
1. 答案是否被上下文支持。
2. 是否遗漏关键条件。
3. 引用是否支持结论。
4. 如果上下文不足，是否正确拒答。
```

注意：

- Judge 也会错。
- Judge 可能偏好更长答案。
- Judge 可能被漂亮格式影响。
- 高风险场景要人工抽检。

最好用人工标注样本校准 judge，一旦规则改了，要重新验证一致性。

---

# 9. 可观测性：记录完整 trace

RAG 系统上线后，必须记录：

```text
query
query rewrite
metadata filters
retrieved candidates
scores
reranked results
final context
prompt version
model version
generation params
answer
citations
latency
token cost
user feedback
```

没有 trace，就很难定位线上问题。

例如用户说“答案错了”，你需要知道：

- 是不是检索到了错文档？
- 是不是用了旧版本？
- 是不是模型没引用正确资料？
- 是不是用户没有权限却看到了内容？

可观测性不是锦上添花，而是 RAG 调试和治理的基础。

---

# 10. 线上监控

常见线上指标：

| 类别 | 指标 |
|---|---|
| 检索 | 空召回率、平均相似度、召回来源分布 |
| 生成 | 拒答率、引用覆盖率、格式失败率 |
| 质量 | 点赞率、差评率、人工质检通过率 |
| 性能 | 检索延迟、rerank 延迟、总延迟 |
| 成本 | embedding 成本、rerank 成本、生成 token |
| 安全 | 越权召回、敏感泄露、注入命中 |
| 漂移 | 新 query 类型、文档更新、失败样本变化 |

线上最有价值的资产是失败样本池。每次人工确认的坏例子，都应该进入离线回归集。

---

# 11. 常见误区

## 11.1 最终答案正确就说明 RAG 好

不一定。模型可能靠参数知识答对，但没有使用证据。RAG 要看答案是否可溯源、可复现、可评估。

## 11.2 Recall@k 高就够了

不够。Recall 高说明候选里有证据，但最终上下文可能噪声大，生成也可能不忠实。

## 11.3 LLM judge 可以完全替代人工

不能。Judge 要校准，高风险和边界样本要人工复核。

## 11.4 没有线上日志也能调好 RAG

很难。RAG 的真实失败往往来自用户问法、文档变化和业务边界，没有日志就无法持续改进。

---

# 12. 面试 Q&A

## Q1：RAG 评估为什么要分层？

因为同一个错误答案可能来自索引、召回、排序、上下文、生成或引用。分层评估能定位失败原因，指导具体改进。

## Q2：Recall@k 和 Context Recall 有什么区别？

Recall@k 看检索候选中是否有正确证据；Context Recall 看最终放进 prompt 的上下文中是否包含答案所需证据。中间可能经过 rerank、过滤和压缩，所以两者不同。

## Q3：Faithfulness 和 Answer Correctness 有什么区别？

Answer Correctness 看答案是否正确；Faithfulness 看答案是否被给定上下文支持。RAG 中即使答案现实中正确，但上下文不支持，也属于不忠实。

## Q4：RAG 上线后为什么要记录 trace？

trace 能帮助定位线上坏例子是检索问题、排序问题、生成问题还是权限问题。没有 trace，就只能猜。

---

# 13. 相关链接

- [[README|RAG 目录]]
- [[03-Hybrid-Retrieval-Reranking-And-Query-Rewriting]]
- [[04-Grounded-Generation-Citations-And-Hallucination-Control]]
- [[06-Production-RAG-And-Advanced-Patterns]]
- [[../LLM-Basic/10-Model-Evaluation|模型评估]]

