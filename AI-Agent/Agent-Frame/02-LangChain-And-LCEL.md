---
tags:
  - LLM
  - AI-Agent
  - LangChain
  - LCEL
created: 2026-05-27
description: LangChain 与 LCEL 基础，讲清 LangChain 的定位、核心模块、LCEL Runnable 编排、工具和 RAG 集成、适用边界与常见问题
---

> **核心考点**：LangChain 的价值不是“替你写 Agent”，而是提供一套模型、Prompt、Retriever、Tool、Parser、Runnable 的集成和编排抽象。LCEL 让链路组合更显式，但复杂状态 Agent 仍更适合 LangGraph 或工作流。

---

# 1. LangChain 解决什么问题

LangChain 早期主要解决：

- 不同 LLM API 统一。
- Prompt 模板。
- Chain 编排。
- Retriever 和向量库集成。
- Tool 包装。
- Output parser。
- Memory。
- Callback 和 LangSmith 观测。

它适合快速搭建：

- RAG 问答。
- 简单工具调用。
- 文档处理流水线。
- Prompt + Model + Parser 链路。

---

# 2. LangChain 的核心模块

可以按职责理解：

| 模块 | 作用 |
|---|---|
| ChatModel/LLM | 模型调用 |
| PromptTemplate | Prompt 模板 |
| Retriever | 检索器 |
| Tool | 工具包装 |
| OutputParser | 输出解析 |
| Runnable | 可组合执行单元 |
| Callback | 中间过程观测 |
| Memory | 对话历史管理 |

不要把这些当“必须全用”。简单任务只需要其中几个。

---

# 3. LCEL：Runnable 编排

LCEL（LangChain Expression Language）核心是把每个步骤看成 Runnable。

直观形式：

```text
input
  -> prompt
  -> model
  -> parser
```

可以组合：

- pipe。
- parallel。
- assign。
- branch。
- retry。
- stream。
- batch。

价值：

- 链路更清晰。
- 每步可替换。
- 支持流式和批处理。
- 便于接 callback。

LCEL 适合 DAG/流水线式任务。若需要复杂循环和持久状态，应该考虑 LangGraph。

---

# 4. LangChain 做 RAG

常见 RAG 链路：

```text
Loader -> Splitter -> Embedding -> VectorStore -> Retriever
User Query -> Retriever -> Prompt -> LLM -> Parser
```

LangChain 优点：

- 集成丰富。
- 原型速度快。
- 生态资料多。

需要注意：

- 默认 splitter 未必适合你的文档。
- Retriever 配置要评估。
- Prompt 和引用规则要自己设计。
- RAG eval 不能只靠 chain 跑通。

---

# 5. LangChain Agent

LangChain Agent 通常围绕：

- 工具列表。
- 模型决策。
- 工具调用。
- observation。
- 最终回答。

早期 Agent 抽象容易让人把控制权交给模型自由循环。生产系统中更推荐：

- 工具数量收敛。
- 设置迭代上限。
- 工具调用有权限和校验。
- 关键流程用 LangGraph 或状态机控制。

---

# 6. LangChain 的优势

- 上手快。
- 集成多。
- 文档和社区丰富。
- 适合原型和 RAG 链路。
- LCEL 组合简洁。
- 可接 LangSmith 做观测。

---

# 7. LangChain 的常见问题

## 7.1 抽象变化快

版本迭代快，API 可能变化。生产项目要锁版本。

## 7.2 调试深层链路困难

链路层层封装后，出错时要看 trace，而不是只看最终输出。

## 7.3 默认组件不等于最佳实践

默认 splitter、retriever、memory 不一定适合业务。

## 7.4 复杂 Agent 状态不够显式

多分支、多循环、人工确认任务更适合 LangGraph。

---

# 8. 适用边界

适合：

- 快速 RAG 原型。
- Prompt/Model/Parser 流水线。
- 轻量工具调用。
- 文档加载和向量库集成。
- 教学和实验。

不适合单独承担：

- 高风险流程状态机。
- 大规模复杂多 Agent 协作。
- 需要强事务和恢复的工作流。
- 严格权限控制。

---

# 9. 常见误区

## 9.1 LangChain 等于 Agent

不对。LangChain 是开发框架，Agent 设计仍要自己完成。

## 9.2 用默认 RAG chain 就能生产

不够。文档处理、检索、重排、引用、权限和评估都要按业务设计。

## 9.3 LCEL 只能串行

不对。LCEL 支持 parallel、branch、assign、stream 等组合，但复杂状态仍需要图。

## 9.4 Memory 开了就解决上下文管理

不对。Memory 只是历史管理方式，不能替代结构化 state 和长期记忆策略。

---

# 10. 面试 Q&A

## Q1：LangChain 适合什么场景？

适合快速构建 LLM 应用、RAG、Prompt/Model/Parser 链路和轻量工具集成，尤其适合原型和中等复杂度应用。

## Q2：LCEL 的核心思想是什么？

把 Prompt、Model、Parser、Retriever 等都抽象成 Runnable，通过组合形成可执行链路，并支持流式、批量、并行和回调。

## Q3：LangChain 和 LangGraph 的关系是什么？

LangChain 更偏组件和链路组合；LangGraph 在其生态上强化状态图、循环、分支、checkpoint 和复杂 Agent 控制。

## Q4：LangChain 上生产要注意什么？

锁版本、保留 trace、不要迷信默认组件、显式处理权限和状态、建立评估集，并对关键流程做可恢复设计。

---

# 11. 相关链接

- [[README|Agent 框架目录]]
- [[01-Framework-Abstractions-And-Design]]
- [[03-LangGraph-Stateful-Agents]]
- [[../RAG/README|RAG]]

