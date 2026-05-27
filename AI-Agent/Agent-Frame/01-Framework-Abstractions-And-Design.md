---
tags:
  - LLM
  - AI-Agent
  - Agent-Framework
created: 2026-05-27
description: Agent 框架通用抽象与设计基础，讲清 Model、Prompt、Chain、Tool、Memory、State、Graph、Callback、Parser、Retriever 等抽象的本质
---

> **核心考点**：不同 Agent 框架名字不同，但底层抽象高度相似：模型调用、输入模板、工具、状态、流程图、输出解析、回调追踪和评估。理解这些抽象，比死背某个框架 API 更重要。

---

# 1. 核心问题：框架到底抽象了什么

Agent 框架通常把一堆胶水代码抽象成标准组件：

```text
Input
  -> Prompt
  -> Model
  -> Parser
  -> Tool / Retriever
  -> State
  -> Workflow
  -> Output
```

框架的价值是让这些组件可组合、可替换、可观测。

真正要理解的是每个抽象背后的职责边界。

---

# 2. Model 抽象

Model 抽象统一不同模型供应商：

- OpenAI。
- Anthropic。
- Gemini。
- 本地 vLLM。
- Ollama。
- Azure。

统一能力：

- chat completion。
- streaming。
- tool calling。
- structured output。
- token usage。
- retry。

注意：不同模型的 tool calling、system prompt、JSON mode、上下文长度和安全策略都可能不同。框架统一接口不代表能力完全等价。

---

# 3. Prompt / Template 抽象

Prompt 模板负责：

- 注入变量。
- 管理 system/user/assistant 格式。
- 复用任务模板。
- 组合上下文和检索结果。

风险：

- 模板过深，调试困难。
- 变量注入不安全。
- prompt 和代码逻辑分散。
- 不同模型 chat template 不一致。

Prompt 模板应该版本化，并与评估结果绑定。

---

# 4. Chain / Runnable 抽象

Chain 表示一段可执行流程：

```text
输入 -> 模型 -> 输出解析
```

更复杂时：

```text
输入
  ├─► 检索
  ├─► Prompt
  ├─► Model
  └─► Parser
```

Runnable 更强调统一执行接口：

- invoke。
- batch。
- stream。
- async。
- compose。

Chain 适合无复杂状态的流水线。如果有循环、分支、状态恢复，就需要 graph/workflow。

---

# 5. Tool 抽象

Tool 通常包含：

- name。
- description。
- args schema。
- function handler。
- permission metadata。
- error handling。

框架可以帮你把函数包装成工具，但工具安全仍要自己设计。

工具设计重点：

- 描述清楚边界。
- 参数可校验。
- 失败可恢复。
- 高风险动作可确认。
- 执行有审计。

---

# 6. Memory 和 State 抽象

Memory 常被框架包装成：

- conversation buffer。
- summary memory。
- vector memory。
- entity memory。

State 更偏当前任务的结构化状态：

```json
{
  "step": "awaiting_confirmation",
  "order_id": "A123",
  "tool_results": {},
  "attempts": 1
}
```

区别：

- Memory 偏“模型需要记住什么”。
- State 偏“系统当前处于什么状态”。

复杂 Agent 应优先显式 State，而不是只依赖 Memory。

---

# 7. Graph / Workflow 抽象

Graph 把 Agent 流程表示成节点和边：

```text
node: retrieve
node: call_model
node: execute_tool
edge: if tool_needed -> execute_tool
edge: else -> final
```

优势：

- 分支清楚。
- 循环可控。
- 状态显式。
- 易于 checkpoint。
- 便于可视化和调试。

适合多步 Agent、工具调用、多轮流程和 Human-in-the-loop。

---

# 8. Parser / Structured Output 抽象

Parser 负责把模型输出转成程序可用结构：

- JSON parser。
- Pydantic parser。
- XML parser。
- function call parser。
- retry parser。

Parser 不是万能的。模型输出业务语义仍要校验：

- 字段合法。
- 权限合法。
- 金额范围。
- 状态可转移。

结构化输出要结合 schema、后置校验和重试。

---

# 9. Callback / Trace 抽象

Callback 记录过程：

- prompt。
- model response。
- token usage。
- tool call。
- retriever result。
- latency。
- error。
- state transition。

Trace 是 Agent 调试和评估基础。没有 trace，很难回答：

```text
为什么这个 Agent 失败了？
是检索、工具、模型还是状态问题？
```

生产系统要把 trace 接到观测平台，而不是只在本地 print。

---

# 10. 常见误区

## 10.1 抽象越多越好

不一定。抽象过深会让调试困难。关键链路要保持可见。

## 10.2 统一接口等于统一能力

不对。不同模型和工具底层能力差异仍然存在。

## 10.3 Memory 可以替代 State

不能。Memory 给模型看，State 给系统控制。生产流程需要结构化 State。

## 10.4 Parser 通过就代表安全

不够。Parser 只保证结构，业务安全靠权限和校验。

---

# 11. 面试 Q&A

## Q1：Agent 框架常见抽象有哪些？

Model、Prompt、Chain/Runnable、Tool、Memory、State、Graph/Workflow、Parser、Callback/Trace、Retriever 和 Eval。

## Q2：Chain 和 Graph 的区别是什么？

Chain 更像线性流水线，适合简单流程；Graph 支持分支、循环、状态和 checkpoint，更适合复杂 Agent。

## Q3：Memory 和 State 有什么区别？

Memory 是模型可用的历史和知识，State 是系统控制流程的结构化状态。生产 Agent 不能只靠 Memory。

## Q4：为什么 trace 很重要？

Trace 能记录模型输入输出、工具调用、检索结果、状态变化和错误，是调试、评估和审计 Agent 的基础。

---

# 12. 相关链接

- [[README|Agent 框架目录]]
- [[00-Agent-Framework-Overview]]
- [[03-LangGraph-Stateful-Agents]]
- [[../Agent-Corepart/README|Agent 核心组件]]

