---
tags:
  - LLM
  - Prompt-Engineering
  - Structured-Output
  - Tool-Use
created: 2026-05-24
description: 结构化输出、工具调用与工作流基础，讲清 JSON、Schema、Function Calling、校验重试、ReAct 和 Agent 工作流 prompt
---

> **核心考点**：LLM 应用进入工程系统后，输出必须能被程序消费。结构化输出、工具调用和工作流 prompt 的关键不是让模型“说得像 JSON”，而是让模型、schema、校验器和工具权限形成闭环。

---

# 1. 核心问题：为什么结构化输出很重要

自然语言适合人读，但程序需要稳定结构。

例如用户输入：

```text
我昨天买了 2 个键盘，一个 399，帮我记一下。
```

如果模型输出：

```text
好的，已记录：昨天购买了两个键盘，每个 399 元。
```

人能懂，程序不一定能直接入库。更好的输出是：

```json
{
  "date": "昨天",
  "item": "键盘",
  "quantity": 2,
  "unit_price": 399
}
```

结构化输出把“生成文本”变成“生成可验证数据”。这对抽取、分类、工具调用、Agent 状态更新都很关键。

---

# 2. 从弱约束到强约束

结构化输出有多个层次。

| 层次 | 做法 | 稳定性 |
|---|---|---|
| 自然语言提示 | “请输出 JSON” | 弱 |
| 模板约束 | 给出字段和示例 | 中 |
| JSON mode | 保证 JSON 语法 | 中高 |
| Schema / Function Calling | 限定字段、类型、必填项 | 高 |
| 约束解码 | 生成过程中限制 token | 很高 |
| 后置校验与重试 | 程序检查并反馈错误 | 必需 |

只写“请输出 JSON”通常不够，因为模型可能：

- 输出解释文字。
- 字段名变化。
- 类型错误。
- 缺少必填字段。
- 自己补不存在的信息。
- 在 JSON 里放入不符合业务规则的值。

工程上要用 schema 和校验器兜底。

---

# 3. Schema 设计：把模糊输出变成契约

好的 schema 会减少模型自由发挥。

例子：

```json
{
  "type": "object",
  "properties": {
    "intent": {
      "type": "string",
      "enum": ["refund", "exchange", "consult", "complaint", "other"]
    },
    "order_id": {
      "type": ["string", "null"]
    },
    "needs_human": {
      "type": "boolean"
    }
  },
  "required": ["intent", "order_id", "needs_human"]
}
```

设计原则：

- 能枚举就枚举。
- 能用数字就不要让模型输出带单位的字符串。
- 缺失值用 `null`，不要让模型猜。
- 布尔值不要写成“是/否/可能”混合。
- 字段名保持稳定。
- 高风险字段加证据字段。

例如：

```json
{
  "answer": "...",
  "evidence_ids": ["doc-3", "doc-7"],
  "unsupported_claims": []
}
```

这能迫使模型把结论和证据连接起来。

---

# 4. 后置校验和重试：不要相信第一次输出

结构化输出链路：

```text
模型输出
  │
  ├─► JSON parse
  ├─► schema validate
  ├─► business validate
  ├─► fail 时带错误信息重试
  └─► pass 后进入下游系统
```

重试 prompt 可以这样写：

```text
你的上一次输出无法通过校验。
错误：
{validation_error}

请只修正格式和字段，不要改变可确定的业务含义。
重新输出符合 schema 的 JSON。
```

注意：重试不应该无限进行。通常设置最大重试次数，并在失败后转人工或返回明确错误。

---

# 5. Function Calling：让模型生成工具调用参数

工具调用不是让模型直接执行动作，而是让模型生成结构化请求：

```json
{
  "tool": "query_order",
  "arguments": {
    "order_id": "A12345"
  }
}
```

系统再决定是否真的执行。

工具调用 prompt 要写清：

- 可用工具有哪些。
- 每个工具能做什么，不能做什么。
- 参数 schema 是什么。
- 什么时候不该调用工具。
- 工具失败后如何处理。
- 哪些工具需要用户确认。

不要让模型自由编造工具。工具集合应由系统提供，模型只能在集合内选择。

---

# 6. 工具 schema 要面向模型理解

工具定义不是给程序员看的注释，而是模型选择工具的依据。

差的工具描述：

```text
get_data: 获取数据
```

更好的工具描述：

```text
query_order_status:
用于根据订单号查询订单状态、物流状态和可退换货状态。
仅当用户提供订单号时调用。
不能用于修改订单或发起退款。
```

参数也要清楚：

```json
{
  "order_id": {
    "type": "string",
    "description": "用户提供的完整订单号，不要猜测或补全"
  }
}
```

工具描述越含糊，模型越容易乱调用。

---

# 7. ReAct：推理和行动交替

ReAct 的核心是让模型在推理和工具行动之间交替：

```text
观察用户问题
  │
  ├─► 判断需要什么信息
  ├─► 调用工具
  ├─► 观察工具结果
  ├─► 决定是否继续
  └─► 给出最终答案
```

在工程系统中，不一定要把“思考”全文暴露给用户。更常见的是维护内部 trace：

```text
model decision -> tool call -> tool result -> next decision -> final answer
```

ReAct 适合：

- 查询外部信息。
- 多步 API 操作。
- 需要根据工具结果继续判断的任务。

风险：

- 工具循环调用。
- 工具参数错误。
- 把工具返回的恶意文本当指令。
- 没有停止条件。

因此 ReAct prompt 必须配合工具权限、调用预算和终止规则。

---

# 8. 工作流 prompt：让模型只负责合适的环节

不是所有步骤都应该交给模型。

例如退款流程：

```text
模型适合：
  识别用户意图
  提取订单号
  解释政策
  判断是否需要转人工

程序适合：
  查询订单
  判断是否超过退款期限
  计算金额
  写数据库
  发起真实退款
```

工作流 prompt 应该说明模型只负责哪些决策，不负责哪些确定性动作。

例子：

```text
你只负责判断用户意图和需要调用的工具。
不要承诺退款成功。
不要自行计算最终退款金额。
金额以 refund_quote 工具结果为准。
```

这能减少模型越权和事实编造。

---

# 9. 常见误区

## 9.1 合法 JSON 等于业务正确

不等于。JSON 只保证语法。字段值是否真实、是否有权限、是否符合业务规则，还要程序校验。

## 9.2 Function Calling 等于工具安全

不等于。Function Calling 只是结构化工具请求。是否执行、能执行什么、需不需要确认，仍然由系统权限控制。

## 9.3 工具结果一定可信

不一定。工具可能失败、超时、返回旧数据，也可能返回来自网页或文档的不可信文本。工具结果也要标记来源和可信级别。

## 9.4 Agent prompt 可以替代状态机

不应该。高价值业务流程应该由状态机或工作流引擎控制关键状态，模型只做语言理解和模糊决策。

---

# 10. 面试 Q&A

## Q1：为什么结构化输出不能只靠“请输出 JSON”？

因为模型可能输出非法 JSON、字段名不一致、类型错误或混入解释文字。生产系统需要 schema、解析校验、业务校验和失败重试。

## Q2：JSON mode 和 Function Calling 有什么区别？

JSON mode 通常只保证输出是合法 JSON，不一定符合业务 schema。Function Calling 会围绕给定工具和参数 schema 生成调用请求，更适合工具执行和受控动作。

## Q3：工具调用链路中模型负责什么，系统负责什么？

模型负责理解意图、选择工具、生成参数和解释工具结果；系统负责提供工具集合、校验参数、控制权限、执行工具、记录审计和处理失败。

## Q4：ReAct 的主要风险是什么？

主要风险是循环调用、参数错误、工具结果被当成指令、调用成本失控和越权执行。需要调用预算、停止条件、权限控制和工具结果隔离。

---

# 11. 相关链接

- [[README|Prompt Engineering 目录]]
- [[01-Prompt-Structure-And-Instructions]]
- [[03-Reasoning-Decomposition-And-Planning]]
- [[05-Prompt-Safety-Evaluation-And-Iteration]]
- [[../LLM-Basic/05-Generation-Strategies|生成策略]]
- [[../LLM-Basic/10-Model-Evaluation|模型评估]]

