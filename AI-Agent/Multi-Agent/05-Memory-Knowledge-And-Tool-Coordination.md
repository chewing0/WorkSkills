---
tags:
  - LLM
  - AI-Agent
  - Multi-Agent
created: 2026-05-27
description: Multi-Agent 记忆、知识与工具协同，解释共享记忆、私有记忆、知识检索、工具权限、资源锁、幂等和行动协调
---

# 记忆、知识与工具协同

多 Agent 系统不仅要让 Agent 说话，还要让它们围绕同一批知识、状态和工具行动。很多系统在 demo 中看起来顺畅，一到真实任务就出问题，原因往往是共享记忆混乱、工具权限过宽、多个 Agent 同时修改同一资源。

记忆和工具是多 Agent 从“讨论系统”变成“行动系统”的关键边界。

---

# 1. 私有记忆与共享记忆

多 Agent 中的记忆不能简单理解为“大家都能看到的聊天记录”。更合理的划分是：

| 类型 | 内容 | 用途 |
|---|---|---|
| 私有记忆 | 单个 Agent 的工作草稿、局部尝试、短期上下文 | 保持角色连续性，避免污染全局 |
| 任务记忆 | 当前子任务的输入、输出、依赖、状态 | 支持任务转交和恢复 |
| 共享记忆 | 已确认事实、关键决策、公共产物、风险清单 | 形成共同事实来源 |
| 长期记忆 | 历史偏好、项目规范、组织知识、过往案例 | 跨任务复用经验 |
| 审计记忆 | 完整 trace、工具调用、消息记录、审批日志 | 调试、合规、评估 |

共享记忆越多，不一定越好。未经确认的猜测如果进入共享记忆，会被后续 Agent 当成事实反复引用。更好的方式是给记忆加状态，例如 candidate、verified、rejected、deprecated。

---

# 2. 共同事实来源

多 Agent 必须有一个“共同事实来源”。它不一定是单一数据库，也可以是结构化状态、文档仓库、知识库和任务系统的组合。

共同事实来源解决的问题是：当 Agent 对同一件事有不同说法时，以哪里为准？

例如在代码任务中，共同事实来源可以是当前 git 工作区、测试结果、issue 描述和设计文档；在客服任务中，可以是订单系统、用户记录、政策条款和操作日志；在研究任务中，可以是已确认引用、原始文档和事实表。

LLM 的对话记忆不应该成为最终事实来源。它适合承载上下文，不适合承载权威状态。

---

# 3. 多 Agent RAG

RAG 在多 Agent 系统里有几种形态。

第一种是共享检索。所有 Agent 使用同一个知识库和检索策略。这简单一致，适合组织知识问答、内部文档助手。

第二种是专用检索。不同 Agent 使用不同知识源，例如法律 Agent 查政策，技术 Agent 查代码库，客服 Agent 查订单。这能提高专业性，也能做权限隔离。

第三种是检索 Agent。专门由一个 Agent 负责查询、筛选、引用和证据整理，其他 Agent 只消费它提供的证据包。这样可以减少重复检索，并让证据质量更容易评估。

多 Agent RAG 的关键不是“检索更多”，而是“证据如何被确认和共享”。建议把检索结果转成结构化证据：

```text
evidence
  ├─ claim：支持哪个结论
  ├─ source：来源
  ├─ quote：关键片段
  ├─ relevance：相关性
  ├─ confidence：置信度
  └─ limitations：限制或反例
```

---

# 4. 工具权限隔离

工具是多 Agent 的行动能力，也是一切风险的入口。

不要让所有 Agent 都拥有所有工具。更安全的方式是按角色分配权限：

- Research Agent：可检索外部资料，只读。
- Coding Agent：可编辑代码，但不能直接部署。
- Test Agent：可运行测试和读取日志。
- Ops Agent：可查看运行状态，但高风险操作需要审批。
- Approval Agent 或人工：负责确认危险动作。

权限隔离的价值在于，即使某个 Agent 被 prompt injection 影响，也不至于立刻执行高风险动作。

工具 schema 也要收窄。与其给一个通用 shell 工具，不如给更具体的 `read_file`、`run_tests`、`query_order`、`create_refund_request`。工具越具体，参数越容易校验，行为越容易审计。

---

# 5. 资源锁与并发写入

多个 Agent 同时行动时，最怕同时修改同一资源。

例如两个编码 Agent 同时改同一个文件，一个测试 Agent 基于旧版本运行测试，一个审查 Agent 又读取了中间状态。最后系统可能不知道哪个结果有效。

常见治理方式包括：

- 给任务分配明确的资源范围，例如文件、模块、数据库记录。
- 写操作前获取锁，避免并发覆盖。
- 所有写操作记录版本号或 revision。
- 合并前做冲突检测。
- 对外部 API 使用幂等键。
- 高风险写操作进入人工审批。

很多多 Agent demo 忽略并发写入，是因为它们只生成文本。一旦系统开始改代码、调 API、写数据库，资源协调就是核心问题。

---

# 6. 工具结果如何进入上下文

工具结果不应该原样塞回所有 Agent。系统需要把工具结果整理成适合下游使用的形式。

例如测试日志可能很长，Coding Agent 需要的是失败用例、错误堆栈和相关文件；Manager Agent 只需要知道测试是否通过、失败是否阻塞；审计系统则需要保存完整日志。

这说明工具结果也要分层：

- 原始结果：完整保存，用于审计。
- 工作摘要：给负责修复的 Agent。
- 状态更新：写入任务系统。
- 关键证据：写入共享记忆。

这样既减少 token 成本，也避免无关 Agent 被噪声影响。

---

# 7. 行动协调

当多个 Agent 都能调用工具时，需要明确行动协议。

一个简单的行动协议可以是：

```text
propose -> validate -> execute -> observe -> record
```

Agent 先提出要执行的动作；系统检查权限、参数和风险；通过后执行工具；执行结果返回；最后把结果写入状态和 trace。

高风险动作可以改成：

```text
propose -> risk_review -> human_approve -> execute -> audit
```

这比让 Agent 直接自由调用工具更可靠。生产 Agent 的核心不是“模型想做什么就做什么”，而是“模型提出意图，系统受控执行”。

---

# 8. 延伸搜索清单

- Shared memory
- Private memory
- Long-term memory
- Vector memory
- Memory consolidation
- Multi-agent RAG
- Evidence table
- Tool permissioning
- Least privilege
- Capability-based security
- Resource locking
- Optimistic concurrency control
- Idempotency key
- Tool result summarization
- Audit log
- Prompt injection defense
- Human approval workflow
