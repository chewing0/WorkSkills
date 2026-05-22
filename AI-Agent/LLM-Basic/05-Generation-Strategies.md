---
tags:
  - LLM
  - Generation
  - Decoding
  - 面试八股
created: 2026-05-21
description: LLM 生成策略详解，从 logits 到 token 的解码链路出发，讲清 Greedy、Beam、Temperature、Top-k、Top-p、约束解码、结构化输出与参数选型
---

> **核心考点**：LLM 每一步并不是直接“写答案”，而是输出词表上每个 token 的分数。生成策略负责把这些分数变成下一个 token。调参的本质是在 **稳定性、多样性、事实性、格式约束、成本** 之间做取舍。

---

# 1. 核心问题：生成策略把概率变成输出

Decoder-only 模型的生成是逐 token 进行的：

```text
prompt
  └─► 生成 token_1
        └─► 再生成 token_2
              └─► 再生成 token_3
```

每一步模型只做一件事：

```text
根据当前上下文，给词表里每个 token 打分
```

这些分数叫 logits。生成策略要决定：

- 是选最高分 token，还是随机采样？
- 随机性应该多大？
- 要不要过滤低概率 token？
- 要不要避免重复？
- 要不要强制输出合法 JSON？
- 生成到哪里停止？

所以生成策略不是模型能力本身，而是 **使用模型输出概率分布的方法**。

---

# 2. 基础机制：从 logits 到 token 的完整链路

模型最后一层 hidden state 经过 LM Head 得到 logits：

$$
z_t = h_t W_{vocab}
$$

其中：

$$
z_t \in \mathbb{R}^{V}
$$

$V$ 是词表大小。一个 token 的 logit 越高，softmax 后概率越大：

$$
P_i = \frac{e^{z_i}}{\sum_j e^{z_j}}
$$

真实推理通常不是直接 softmax，而是一条处理流水线：

```text
raw logits
  │
  ├─► temperature：调分布尖锐程度
  ├─► penalties：惩罚重复或鼓励新主题
  ├─► masks / logit bias：禁用或鼓励某些 token
  ├─► filtering：Top-k / Top-p / Min-p
  ├─► softmax：转成概率
  └─► select：argmax 或 sample
```

这一串处理常叫 logits processor / logits warper。

面试时可以这样说：

> 生成策略不是一个孤立算法，而是一条从 logits 到 token 的处理链路。不同参数会改变分布形状、候选集合、约束边界和停止条件。

---

# 3. 先选目标：你要稳定、创造，还是格式严格

调参前先问任务目标。

| 目标 | 应该偏向 |
|---|---|
| 稳定、可复现 | greedy、低 temperature |
| 创意、多样性 | sampling、高一点 temperature、Top-p |
| 事实问答 | 低 temperature、RAG、引用约束 |
| JSON / 工具调用 | 低 temperature、schema / function calling |
| 代码 | 中低 temperature、多候选 + 测试 |
| 摘要/翻译 | 低温或 beam，控制长度 |

没有“万能参数”。同一个模型在客服、写诗、抽取 JSON、生成代码时，应该使用不同生成策略。

---

# 4. 同族概念分组：选择策略决定怎么选 token

这一组回答的问题是：

> 概率分布已经有了，下一步到底选哪个 token？

## 4.1 Greedy：每一步都选最高概率

Greedy decoding：

$$
y_t = \arg\max_i P(i \mid y_{<t}, x)
$$

优点是快、稳定、便宜、可复现。缺点是每一步局部最优，不保证整段文本最好，也容易保守和重复。

适合：

- 分类。
- 信息抽取。
- 工具参数。
- 严格 JSON。
- 需要稳定复现的业务流程。

常见误区：

> greedy 稳定，不等于一定正确。它只是选择模型当前认为最自然的 token。

## 4.2 Sampling：从概率分布中抽样

Sampling 按概率随机选择 token。

```text
A: 0.50
B: 0.30
C: 0.20
```

Greedy 总选 A；sampling 可能选 A，也可能选 B 或 C。采样带来多样性，也带来不稳定，所以通常要配合 temperature、Top-k、Top-p 限制候选范围。

## 4.3 Beam Search：保留多条高分路径

Beam Search 不是随机采样，而是搜索多个候选序列。

```text
beam_size = 3

第 1 步保留 3 条候选
第 2 步每条继续扩展，再保留总分最高 3 条
重复直到结束
```

序列分数通常来自 log probability 累加：

$$
score(y) = \sum_t \log P(y_t \mid y_{<t}, x)
$$

适合翻译、摘要、语音识别等输出比较确定的任务，不适合开放式聊天、创意写作和长篇自由生成。因为 beam search 倾向高概率、保守、模板化的句子，而且更慢。

Beam Search 还常配合 length penalty，避免模型为了高分过早结束。

## 4.4 Best-of：先生成多个，再选一个

Best-of 思路：

```text
生成 N 个候选
  │
  ├─► 用规则 / 测试 / judge / reward model 打分
  └─► 选择最好的
```

适合：

- 代码：多个候选跑单元测试。
- 数学：多个推理路径投票。
- 创意：给多个版本。
- 复杂问答：采样后 rerank。

代价是 token 成本和延迟上升，而且需要可靠选择器。Best-of 不是单步解码算法，而是一种候选生成和选择框架。

---

# 5. 同族概念分组：分布塑形和候选过滤控制随机性

这一组回答的问题是：

> 如果要采样，应该从多大的候选集合里采，分布应该多尖锐？

## 5.1 Temperature：控制分布形状

Temperature 调整 logits：

$$
P_i = \text{softmax}(z_i / T)
$$

| Temperature | 直觉 | 适合 |
|---:|---|---|
| $T=0$ | 工程上近似 greedy | 分类、抽取、工具 |
| $0<T<1$ | 分布更尖锐，更保守 | 事实问答、代码、摘要 |
| $T=1$ | 原始分布 | 通用聊天 |
| $T>1$ | 分布更平，低概率 token 更容易出现 | 创意写作、头脑风暴 |

高 temperature 不等于更聪明，只是更随机。事实任务、工具调用、JSON、分类通常应低温。

## 5.2 Top-k：固定候选数量

Top-k 只保留概率最高的 $k$ 个 token：

```text
top_k = 5
只在概率最高的 5 个 token 中采样
```

它像“最多给你 k 个备选项”。问题是候选数量固定：有些位置合理候选很多，k 可能太小；有些位置合理候选很少，k 可能太大。

## 5.3 Top-p：按累计概率动态截断

Top-p 又叫 nucleus sampling。它按概率从高到低排序，保留累计概率达到 $p$ 的最小集合。

```text
A: 0.40
B: 0.25
C: 0.15
D: 0.08
E: 0.05
```

如果 `top_p = 0.8`，保留 A、B、C。

Top-p 的好处是候选集合随上下文动态变化：模型很确定时候选少，模型开放时候选多。常见开放生成配置是 `temperature = 0.7 ~ 1.0`、`top_p = 0.8 ~ 0.95`。

## 5.4 Min-p 与 Typical Sampling

Min-p 用最高概率 token 做参考，只保留相对不太低的候选：

```text
保留 P(token) >= min_p * P_max 的 token
```

Typical sampling 希望选择“信息量接近模型预期”的 token，避免过于平庸或过于离谱。

工程里更常见的仍然是：

```text
temperature + top_p
```

---

# 6. 同族概念分组：logits 约束和停止控制让输出可用

这一组回答的问题是：

> 模型不只要会生成，还要避免重复、遵守边界，并且知道在哪里停。

## 6.1 重复控制：为什么模型会啰嗦

模型生成时容易重复：

```text
这个问题非常重要，非常重要，非常重要...
```

原因可能是局部高概率 token 反复被选中、上下文里重复模式被模型延续、stop 条件不清楚或采样策略不合适。

常见手段：

| 方法 | 作用 |
|---|---|
| Repetition Penalty | 对已经出现过的 token 降低分数 |
| Frequency Penalty | 出现次数越多，惩罚越强 |
| Presence Penalty | 只要出现过就惩罚，鼓励新主题 |
| No-repeat N-gram | 禁止重复出现某个 n-gram |

重复惩罚太强会导致模型用奇怪表达绕开惩罚；对代码、合同、固定术语和 JSON 字段要谨慎。

## 6.2 停止条件：生成必须知道在哪里停

常见停止条件：

- EOS token。
- `max_tokens`。
- stop sequences。
- 工具调用结束符。
- JSON schema 完成。
- 服务端超时或用户取消。

`max_tokens` 限制最多生成多少输出 token，它不是上下文窗口。上下文窗口通常是：

```text
输入 token + 输出 token <= context window
```

stop sequences 是外部指定的停止字符串：

```text
stop = ["\n\nUser:", "</tool_call>"]
```

stop 需要和 tokenizer 行为匹配；太宽泛可能过早截断，太窄可能停不住。

## 6.3 Logit Bias 与 Token Mask

Logit bias 直接调整某些 token 的 logit：

```text
禁止某 token: bias = -100
鼓励某 token: bias = +5
```

常见用途：

- 限制只能输出 A/B/C。
- 禁止某些 token。
- 强制某些格式前缀。
- 控制工具调用特殊 token。

这比 prompt 更硬，但也更危险。一个词可能拆成多个 token，中文、空格、大小写也可能导致漏控。

---

# 7. 工程含义：结构化输出不要只靠“请输出 JSON”

结构化输出有四层。

## 7.1 Prompt 约束

```text
请只输出 JSON，不要解释。
```

优点：简单。
缺点：不保证合法，不保证符合 schema。

## 7.2 JSON Mode

通常保证输出是合法 JSON，但可能字段不对。

例如你想要：

```json
{"name": "Alice", "age": 18}
```

模型可能输出：

```json
{"user_name": "Alice", "age": "18"}
```

JSON 合法，但 schema 不一定对。

## 7.3 Schema / Function Calling

通过 schema 或函数定义约束字段、类型和必填项，更适合工具调用、抽取和生产系统。

它本质上是把自然语言输出问题变成结构化参数生成问题。

## 7.4 约束解码

约束解码在生成过程中限制 token，只允许产生符合语法的序列。

边界：

- 能保证格式，不保证事实正确。
- schema 太复杂时可能影响速度。
- 业务语义仍需校验。

---

# 8. 工程含义：工具调用生成

工具调用通常不是让模型直接执行代码，而是让模型生成结构化调用请求：

```json
{
  "name": "search_docs",
  "arguments": {
    "query": "KV Cache 原理"
  }
}
```

关键点：

- 工具名必须可控。
- 参数必须符合 schema。
- 不能让模型自由编造不存在的工具。
- 工具结果返回后，模型还要基于结果继续生成。

常见错误：

- 参数类型错。
- 工具名拼错。
- 把解释文字混入 JSON。
- 工具结果过长导致上下文膨胀。
- 工具失败后不会恢复。

因此工具调用通常使用低 temperature，并配合 schema、校验和重试。

---

# 9. 工程含义：流式生成、Logprobs 与置信度

## 9.1 流式生成

流式生成不是模型一次性更快，而是把 token 边生成边返回。

优点：

- 用户更早看到响应。
- 体感延迟降低。
- 长回答体验更好。

注意：

- 首 token 延迟仍由 prefill 决定。
- 输出解析要处理半截 JSON。
- 用户取消时要停止后端生成。

## 9.2 Logprobs 与置信度

Logprobs 可以观察模型对生成 token 的概率。

用途：

- 分类置信度粗估。
- 比较候选答案。
- 分析模型犹豫位置。
- 做拒答或人工复核阈值。

但 logprob 不是事实置信度。模型可能非常自信地生成错误事实，也可能对正确冷门事实低概率。

---

# 10. 工程含义：按任务目标选择参数

| 场景 | 推荐倾向 | 原因 |
|---|---|---|
| 分类 | greedy / temperature 0 | 稳定、可复现 |
| 信息抽取 | 低温 + schema | 格式比多样性重要 |
| RAG 问答 | 低温 + 引用约束 | 减少编造 |
| 创意写作 | 中高温 + Top-p | 增加多样性 |
| 代码生成 | 中低温 + best-of + 测试 | 需要正确性验证 |
| 工具调用 | 低温 + function calling | 参数要稳定 |
| 摘要 | 低温或 beam | 控制长度和覆盖 |

调参顺序建议：

```text
先确定任务目标
  │
  ├─► 再选 greedy / sampling / beam / best-of
  │
  ├─► 再调 temperature / top_p
  │
  ├─► 再加重复惩罚和 stop
  │
  └─► 最后用评估集验证
```

---

# 11. 常见误区与调参

## 11.1 Temperature 高会让模型更聪明

不会。它只是让低概率 token 更容易被采样，可能更有创意，也可能更离谱。

## 11.2 Top-p 越低越安全

不一定。Top-p 太低会让模型过于保守，甚至在需要开放表达时变差。事实安全主要靠知识来源、引用、工具和评估。

## 11.3 Greedy 一定最准确

不一定。Greedy 是局部最高概率，不等于全局最优，也不等于事实正确。

## 11.4 结构化输出只靠 prompt 就够

生产系统通常不够。需要 JSON mode、schema/function calling、约束解码、后置校验和重试。

## 11.5 Logprob 可以当事实置信度

不能直接当。Logprob 表示模型对文本形式的概率，不代表外部世界真实性。

---

# 12. 面试 Q&A

## Q1：Temperature 和 Top-p 的区别是什么？

Temperature 改变整个概率分布的尖锐程度；Top-p 改变可采样候选集合。Temperature 控制“概率差距”，Top-p 控制“候选范围”。二者常一起使用。

## Q2：为什么 greedy 可能生成质量不好？

因为 greedy 每一步只选局部最高概率 token，不考虑后续全局效果。它稳定但容易保守、重复，也可能在开放任务中错过更好的表达。

## Q3：Beam Search 为什么不常用于聊天？

聊天是开放式生成，单纯追求高概率序列会让回答保守、模板化，而且 beam search 计算更贵。它更适合翻译、摘要等目标更确定的任务。

## Q4：结构化输出怎么保证稳定？

从弱到强可以用 prompt 约束、JSON mode、schema/function calling、约束解码，再加后置校验和重试。格式约束不等于事实正确，业务语义仍要验证。

## Q5：max_tokens 和 context window 是一回事吗？

不是。`max_tokens` 限制最多生成多少输出 token；context window 限制输入 token 加输出 token 的总长度。

## Q6：为什么 logprob 不能当事实置信度？

Logprob 衡量模型认为某段文本在当前上下文下是否自然，不衡量文本是否真实。模型可能高概率生成错误事实，也可能低概率生成正确冷门事实。

## Q7：工具调用为什么要低 temperature？

工具调用更看重参数稳定和 schema 正确，不需要语言多样性。高 temperature 会增加工具名、字段、类型和参数错误概率。

## Q8：RAG 问答怎么调生成参数？

通常用低 temperature、较保守的 Top-p、引用约束和较清晰的“不知道就说不知道”规则。重点不是让模型更随机，而是让它忠实使用检索上下文。

---

# 13. 记忆口诀

```text
先看 logits，再谈采样；
稳定用 greedy，创造用 sampling；
温度调形状，Top-p 管范围；
Beam 保守，Best-of 昂贵；
结构化靠 schema，不靠祈祷；
事实性靠证据，不靠高温。
```

---

# 14. 相关链接

- [[00-LLM-Overview]]
- [[02-Tokenizer-Embedding]]
- [[04-Training-Objectives]]
- [[06-Context-Window-KV-Cache]]
- [[08-Inference-Optimization]]
- [[10-Model-Evaluation]]
