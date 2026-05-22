---
tags:
  - LLM
  - Transformer
  - 面试八股
created: 2026-05-06
description: Transformer 核心架构详解，覆盖 Attention、位置编码、Norm、FFN、KV Cache、现代 LLM 变体及面试高频考点
---

> **核心考点**：Transformer 的本质是“让 token 彼此通信，再对每个 token 做非线性计算”的可堆叠残差网络。面试和工程中最常考的是 Self-Attention 计算过程、Q/K/V 语义、Mask、Multi-Head、位置编码、PreNorm/PostNorm、RMSNorm、FFN、KV Cache 和复杂度。

---

# 1. 核心问题：Transformer 如何让 token 理解上下文

Transformer 由 Vaswani 等人在 2017 年论文《Attention Is All You Need》中提出。它的核心变化是：**不再用 RNN 的顺序递推，也不依赖 CNN 的局部卷积，而是用 Attention 让序列中任意两个 token 直接建立联系**。

这篇只讲 Transformer Block 的基本工作方式：Attention 负责在 token 之间传信息，FFN 负责对每个位置做非线性加工，Residual 和 Normalization 负责让深层网络稳定训练，位置编码负责补上顺序信息。更深入的 KV Cache、长上下文和推理优化分别放在 [[06-Context-Window-KV-Cache]]、[[07-Length-Extrapolation]]、[[08-Inference-Optimization]]。

一句话理解：

> Transformer = Embedding + Position 信息 + N 层 Block + 输出头
> 每个 Block = Attention（token 之间通信） + FFN（每个 token 独立计算） + Residual + Norm

RNN 需要从左到右一步步传递状态，长距离信息容易衰减，也很难并行。Transformer 的 Self-Attention 一次性计算所有 token 之间的关系，虽然有 $O(n^2)$ 的注意力矩阵，但矩阵乘法非常适合 GPU 并行，因此在中等长度序列上训练效率远高于 RNN。

## 1.1 Transformer Block 的最小心智模型

以现代 Decoder-only LLM 的 PreNorm 结构为例，一个 Block 通常长这样：

```text
x
│
├─► Norm ─► Self-Attention ─► + ──► x'
│                              ▲
└──────────────────────────────┘

x'
│
├─► Norm ─► FFN/MLP ─────────► + ──► y
│                              ▲
└──────────────────────────────┘
```

含义非常直接：

- Attention 回答“当前位置应该从哪些 token 拿信息”。
- FFN 回答“拿到上下文以后，这个位置的表示应该怎样变换”。
- Residual 保留原始信息并提供稳定的梯度通道。
- Norm 控制数值尺度，让深层网络更容易训练。

---

# 2. 整体架构

原始 Transformer 是 Encoder-Decoder 架构，用于机器翻译。后来 BERT 主要使用 Encoder-only，GPT/Llama/Qwen/DeepSeek 等大语言模型主要使用 Decoder-only。

## 2.1 Encoder-Decoder 总览

```text
Source tokens
    │
    ├─► Token Embedding + Position Encoding
    │
    ├─► Encoder Block × N
    │      ├─ Multi-Head Self-Attention (bidirectional)
    │      ├─ Add & Norm
    │      ├─ Feed Forward Network
    │      └─ Add & Norm
    │
    └─► Encoder hidden states ───────────────┐
                                             │
Target tokens                                │
    │                                        │
    ├─► Token Embedding + Position Encoding  │
    │                                        │
    ├─► Decoder Block × N                    │
    │      ├─ Masked Multi-Head Self-Attention
    │      ├─ Add & Norm
    │      ├─ Cross-Attention over Encoder hidden states
    │      ├─ Add & Norm
    │      ├─ Feed Forward Network
    │      └─ Add & Norm
    │
    └─► Linear + Softmax
```

## 2.2 三类主流范式

| 架构 | 注意力方式 | 典型预训练任务 | 适合任务 | 代表模型 |
|---|---|---|---|---|
| Encoder-only | 双向 Self-Attention | MLM，掩码语言模型 | 分类、检索、NER、NLI、句向量 | BERT、RoBERTa |
| Decoder-only | 因果 Self-Attention | Next Token Prediction | 对话、代码、写作、Agent、通用生成 | GPT、Llama、Qwen、DeepSeek |
| Encoder-Decoder | Encoder 双向，Decoder 因果，带 Cross-Attention | Seq2Seq、Span Corruption | 翻译、摘要、结构化改写 | T5、BART、原始 Transformer |

现代 LLM 通常采用 Decoder-only，不是因为 Encoder-Decoder 不强，而是因为 Causal LM 的训练目标极其简单、数据形式统一、推理时可以高效使用 KV Cache，并且生成式接口可以覆盖理解、推理、工具调用和多轮对话。

---

# 3. 基础机制：Self-Attention 如何汇聚上下文

Self-Attention 是 Transformer 的核心。它让每个 token 根据当前上下文，从整段序列中聚合自己需要的信息。

假设输入序列长度为 $n$，模型维度为 $d_{model}$：

$$
X \in \mathbb{R}^{n \times d_{model}}
$$

每一层 Attention 都会把 $X$ 投影成三组向量：

$$
\begin{aligned}
Q &= XW^Q \quad W^Q \in \mathbb{R}^{d_{model} \times d_k} \\
K &= XW^K \quad W^K \in \mathbb{R}^{d_{model} \times d_k} \\
V &= XW^V \quad W^V \in \mathbb{R}^{d_{model} \times d_v}
\end{aligned}
$$

其中：

- **Query**：当前位置想找什么信息。
- **Key**：每个位置能被别人匹配到的索引特征。
- **Value**：每个位置真正贡献出去的内容。

可以用检索系统类比：

- Q 像搜索词。
- K 像文档索引。
- Q 和 K 的相似度决定检索权重。
- V 是最终被加权读出的文档内容。

## 3.1 Scaled Dot-Product Attention

标准公式：

$$
\text{Attention}(Q, K, V) =
\text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V
$$

拆开看有四步：

1. **计算相关性分数**：

$$
S = QK^T \in \mathbb{R}^{n \times n}
$$

$S_{ij}$ 表示第 $i$ 个 token 对第 $j$ 个 token 的关注程度。

2. **缩放**：

$$
\hat{S} = \frac{S}{\sqrt{d_k}}
$$

3. **softmax 归一化**：

$$
A = \text{softmax}(\hat{S})
$$

每一行都是一个概率分布，表示一个 token 对所有 token 的注意力权重。

4. **加权求和 Value**：

$$
O = AV
$$

输出 $O_i$ 是第 $i$ 个 token 从所有 token 的 Value 中加权汇聚得到的新表示。

## 3.2 为什么要除以 $\sqrt{d_k}$

如果 $q$ 和 $k$ 的每个分量独立，均值为 0，方差为 1，那么：

$$
q \cdot k = \sum_{i=1}^{d_k} q_i k_i
$$

其方差约为 $d_k$。当 $d_k$ 很大时，点积结果的绝对值会变大，softmax 容易进入饱和区，输出接近 one-hot，梯度变小，训练不稳定。

除以 $\sqrt{d_k}$ 后，点积分数的方差被拉回到约 1，softmax 更平滑，梯度更稳定。

## 3.3 Attention 矩阵在表达什么

Attention 矩阵：

$$
A \in \mathbb{R}^{n \times n}
$$

可以理解为一张“信息路由表”：

- 第 $i$ 行：第 $i$ 个 token 从其他 token 读取信息的比例。
- 第 $j$ 列：第 $j$ 个 token 被其他位置关注的程度。
- 矩阵越大，显存和计算越贵，这就是长上下文瓶颈的来源。

注意：Attention 权重不一定等价于人类可解释的“重要性”。它只是模型内部的一种信息混合系数，不能过度解读为稳定的解释证据。

## 3.4 Self-Attention 为什么需要位置编码

如果没有位置编码，Self-Attention 对输入顺序本身是不敏感的。因为对 $X$ 做同样的行置换，$Q/K/V$ 和输出也会跟着置换，模型只知道“有哪些 token”，不知道“它们在什么位置”。

所以 Transformer 必须额外注入位置信息，否则它无法区分：

```text
狗 咬 人
人 咬 狗
```

这也是 Position Encoding / Position Embedding 存在的根本原因。

---

# 4. Mask：让模型看该看的内容

Attention 的原始形式会让每个 token 看见所有 token。但不同任务需要不同可见性，所以要加 Mask。

Mask 通常加在 softmax 之前的 Attention Score 上：

$$
\text{Attention}(Q, K, V) =
\text{softmax}\left(\frac{QK^T}{\sqrt{d_k}} + M\right)V
$$

被遮住的位置加一个很大的负数，理论上是 $-\infty$，softmax 后概率约为 0。

## 4.1 Padding Mask

batch 内序列长度不同，需要 padding 到同一长度。Padding token 不应该被真实 token 关注，因此要用 Padding Mask 屏蔽。

```text
真实 token:  我  喜欢  NLP
padding:    PAD PAD PAD
```

没有 Padding Mask 时，模型可能把 PAD 当作有效内容，污染表示。

## 4.2 Causal Mask / Look-ahead Mask

Decoder-only 模型做自回归生成，第 $i$ 个位置只能看见 $0 \sim i$ 的历史 token，不能偷看未来 token。

因果掩码：

$$
M_{ij} =
\begin{cases}
0, & j \le i \\
-\infty, & j > i
\end{cases}
$$

矩阵形状：

```text
        key position
        1  2  3  4
query 1 ✓  ✗  ✗  ✗
query 2 ✓  ✓  ✗  ✗
query 3 ✓  ✓  ✓  ✗
query 4 ✓  ✓  ✓  ✓
```

这保证训练时可以并行计算整段序列，同时每个位置仍然只依赖过去。

## 4.3 训练和推理的差别

训练阶段：

- 输入整段 token。
- 用 Causal Mask 保证每个位置只能预测下一个 token。
- 所有位置的 loss 可以并行计算。

推理阶段：

- prefill 阶段先处理完整 prompt。
- decode 阶段每次生成一个新 token。
- 使用 KV Cache 保存历史 Key/Value，不必重复计算历史 token 的 K/V。

---

# 5. 同族概念分组：Attention 的头、共享和缓存

单头 Attention 只在一个表示子空间里计算相似度。Multi-Head Attention 把表示拆成多个头，让模型从不同角度建立依赖关系。

公式：

$$
\begin{aligned}
\text{head}_i &= \text{Attention}(QW_i^Q, KW_i^K, VW_i^V) \\
\text{MHA}(Q,K,V) &= \text{Concat}(\text{head}_1,\ldots,\text{head}_h)W^O
\end{aligned}
$$

通常：

$$
d_k = d_v = \frac{d_{model}}{h}
$$

如果 $d_{model}=4096$，$h=32$，那么每个 head 的维度是 128。

## 5.1 多头为什么有效

多头的意义不是简单地“参数更多”，而是：

- 不同 head 可以关注不同关系，例如局部邻近、长距离指代、语法依赖、分隔符、代码缩进等。
- 每个 head 在较低维空间里计算，softmax 的匹配模式更灵活。
- 多个 head 的结果拼接后再线性混合，形成更丰富的信息路由。

一个常见误区是：多头数量越多越好。实际上 head_dim 太小会削弱单个 head 的表达能力，head 太多也可能出现冗余。工程上常见 head_dim 为 64、80、96、128 等，具体取决于模型规模和硬件效率。

## 5.2 参数量怎么估算

标准 MHA 中，$W^Q, W^K, W^V, W^O$ 通常都是 $d_{model} \times d_{model}$ 级别。因此单层 Attention 的参数量约为：

$$
4d_{model}^2
$$

注意：在固定 $d_{model}$ 的情况下，拆成多个 head 通常不会显著增加总参数量，因为只是把投影后的维度切分成多个头。

## 5.3 MHA、MQA、GQA 的边界

现代大模型越来越关注推理速度和 KV Cache 显存，因此出现了 MQA 和 GQA。

| 机制 | Query 头数 | Key/Value 头数 | 优点 | 缺点 |
|---|---:|---:|---|---|
| MHA | 多个 | 多个，通常与 Q 相同 | 表达能力强 | KV Cache 大 |
| MQA | 多个 | 1 个 | KV Cache 极小，推理快 | 可能损失效果 |
| GQA | 多个 | 少量分组 | 效果和速度折中 | 比 MQA 稍贵 |

GQA 的直觉：多个 Query head 共享一组 K/V head。这样既保留多头查询能力，又减少缓存的 K/V 数量，是现代 LLM 常见选择。

这里先记住边界：MHA、MQA、GQA 都属于 Attention 的 K/V 共享策略，不改变“用 Q 匹配 K、再聚合 V”的基本机制。它们主要影响 KV Cache 显存和 decode 带宽，细节见 [[06-Context-Window-KV-Cache]]。

---

# 6. 同族概念分组：位置编码负责补上顺序

Attention 本身不感知顺序，所以必须引入位置。位置方案大致分为三类：

- 绝对位置：告诉模型“这是第几个 token”。
- 相对位置：告诉模型“两个 token 相距多远”。
- 位置偏置：直接在 Attention Score 中加入距离相关 bias。

## 6.1 Sinusoidal Position Encoding

原始 Transformer 使用固定正弦/余弦函数：

$$
\begin{aligned}
PE_{(pos, 2i)} &= \sin\left(\frac{pos}{10000^{2i/d_{model}}}\right) \\
PE_{(pos, 2i+1)} &= \cos\left(\frac{pos}{10000^{2i/d_{model}}}\right)
\end{aligned}
$$

然后与 token embedding 相加：

$$
X = E_{token} + PE
$$

优点：

- 不需要学习参数。
- 理论上可以生成任意长度的位置向量。
- 不同维度对应不同频率，能表达多尺度位置。

缺点：

- 位置直接加在输入上，经过多层变换后相对关系不一定保持清晰。
- 长度外推在真实模型中并不总是可靠。

## 6.2 Learned Absolute Position Embedding

为每个位置学习一个向量：

$$
P \in \mathbb{R}^{max\_len \times d_{model}}
$$

输入为：

$$
X_i = E_i + P_i
$$

优点是灵活，缺点是训练时没见过的位置没有可靠表示，因此很难自然外推到更长上下文。

## 6.3 Relative Position Bias

相对位置方案不直接问“这是第几个 token”，而是建模：

$$
\text{distance} = i - j
$$

典型做法是在 Attention Score 上加相对位置 bias：

$$
\text{score}_{ij} = \frac{q_i k_j^T}{\sqrt{d_k}} + b_{i-j}
$$

T5 等模型使用过这类相对位置偏置。它比绝对位置更符合语言中的很多依赖关系：模型通常更关心两个词之间的距离，而不是它们在整段文本中的绝对编号。

## 6.4 RoPE：Rotary Position Embedding

RoPE 是现代 Decoder-only LLM 中非常常见的位置编码。它不把位置向量加到 embedding 上，而是在 Attention 计算前旋转 Query 和 Key。

二维情况下，对向量做旋转：

$$
R_m =
\begin{pmatrix}
\cos m\theta & -\sin m\theta \\
\sin m\theta & \cos m\theta
\end{pmatrix}
$$

位置 $m$ 的 Query 和位置 $n$ 的 Key 分别变成：

$$
\tilde{q}_m = R_m q,\quad \tilde{k}_n = R_n k
$$

它们的内积满足：

$$
\langle \tilde{q}_m, \tilde{k}_n \rangle
= \langle R_m q, R_n k \rangle
= g(q, k, m-n)
$$

也就是说，Attention Score 天然依赖相对位置 $m-n$。

RoPE 的优点：

- 位置信息直接进入 Attention Score，位置关系更明确。
- 具有良好的相对位置建模能力。
- 与自回归生成和 KV Cache 兼容。
- 可通过位置插值、NTK-aware scaling、YaRN 等方法扩展上下文长度。

RoPE 的注意点：

- RoPE 不等于无限长度外推。超过训练长度很多以后，高频维度可能出现分布偏移。
- 长上下文扩展通常还需要训练、微调或特殊 scaling 策略配合。

## 6.5 ALiBi

ALiBi 直接在 Attention Score 中加入与距离相关的线性负偏置：

$$
\text{score}_{ij}
= \frac{q_i k_j^T}{\sqrt{d_k}} - m_h \cdot |i-j|
$$

$m_h$ 是第 $h$ 个 head 的斜率。距离越远，bias 越负，注意力越容易衰减。

优点：

- 不需要位置 embedding。
- 长度外推能力强。
- 实现简单，对 KV Cache 友好。

缺点：

- 强行给远距离 token 加惩罚，不一定适合所有任务。
- 对需要精确绝对位置的任务表达较弱。

## 6.6 位置方案对比

| 方案 | 类型 | 加在哪里 | 是否学习参数 | 长度外推 | 常见场景 |
|---|---|---|---|---|---|
| Sinusoidal | 绝对位置 | 输入 embedding | 否 | 一般 | 原始 Transformer |
| Learned PE | 绝对位置 | 输入 embedding | 是 | 较弱 | BERT、早期 GPT |
| Relative Bias | 相对位置 | Attention score | 可学习或固定 | 较好 | T5 类模型 |
| RoPE | 相对位置 | Q/K 旋转后内积 | 否 | 好，但需 scaling | Llama、Qwen 等 |
| ALiBi | 相对距离偏置 | Attention score | 否 | 强 | 长度外推实验、部分开源模型 |

---

# 7. 基础组件：Feed Forward Network 负责逐位置加工

Attention 负责 token 之间的信息交换，但它本质上是加权求和。真正把每个位置的表示做复杂非线性变换的，是 FFN，也常叫 MLP。

## 7.1 标准 FFN

原始 Transformer 的 FFN：

$$
\text{FFN}(x) = \max(0, xW_1 + b_1)W_2 + b_2
$$

其中：

$$
W_1 \in \mathbb{R}^{d_{model} \times d_{ff}},\quad
W_2 \in \mathbb{R}^{d_{ff} \times d_{model}}
$$

通常 $d_{ff} = 4d_{model}$。

重要特点：

- FFN 对每个 token 独立应用。
- 所有位置共享同一组 FFN 参数。
- 它不会直接混合不同 token 的信息，跨 token 信息混合已经由 Attention 完成。

## 7.2 FFN 为什么重要

FFN 的作用至少有三层：

1. **增加非线性表达能力**
   没有 FFN，Attention 主要是在做线性投影和加权平均，表达能力会受限。

2. **对上下文后的 token 表示做变换**
   Attention 把上下文信息汇聚到每个位置，FFN 再对这个位置进行“思考”和重编码。

3. **存储知识和模式**
   很多研究把 FFN 视为近似的键值记忆网络。事实知识、短语模式、代码模式等都可能被存储在 FFN 参数中。

## 7.3 参数量占比

如果 $d_{ff}=4d_{model}$，标准 FFN 参数量约为：

$$
2d_{model}d_{ff} = 8d_{model}^2
$$

标准 Attention 约为：

$$
4d_{model}^2
$$

所以在经典配置下，FFN 的参数量通常比 Attention 更大。大模型里大量“知识”存在 MLP/FFN 中，这也是 MoE 通常替换 FFN 而不是替换 Attention 的原因之一。

## 7.4 激活函数和门控变体

| 结构 | 形式 | 特点 | 常见模型 |
|---|---|---|---|
| ReLU FFN | $\text{ReLU}(xW_1)W_2$ | 简单高效 | 原始 Transformer |
| GELU FFN | $\text{GELU}(xW_1)W_2$ | 平滑，训练稳定 | BERT、GPT-2/3 类 |
| GEGLU | $\text{GELU}(xW_g) \odot xW_u$ | 门控 MLP | T5 变体、PaLM 相关 |
| SwiGLU | $\text{SiLU}(xW_g) \odot xW_u$ | 效果好，现代 LLM 常用 | Llama、Qwen 等 |

SwiGLU 常写作：

$$
\text{SwiGLU}(x) =
(\text{SiLU}(xW_g) \odot xW_u)W_d
$$

其中 $\odot$ 是逐元素乘法。门控结构可以让模型动态控制哪些特征通过，通常比普通 FFN 表现更好。

## 7.5 MoE 与 FFN

MoE（Mixture of Experts）通常把密集 FFN 替换成多个专家 FFN，并通过 Router 为每个 token 选择少量专家：

```text
token hidden state
    │
    ├─► Router: 选择 top-k experts
    │
    ├─► Expert 1 / Expert 7 / ...
    │
    └─► 加权合并
```

这样可以显著增加总参数量，但每个 token 只激活少量参数，计算量不按总参数量线性增长。DeepSeek、Mixtral 等模型都使用了类似思想。

---

# 8. 基础组件：Residual 与 Normalization 负责稳定训练

Transformer 能堆得很深，离不开 Residual 和 Normalization。

## 8.1 Residual Connection

残差连接：

$$
y = x + F(x)
$$

作用：

- 保留原始信息，避免每层都强制覆盖表示。
- 提供梯度直通路径，缓解深层网络训练困难。
- 让每一层学习“增量修正”，而不是从零重建表示。

## 8.2 LayerNorm

LayerNorm 对单个样本的特征维度归一化：

$$
\text{LayerNorm}(x) =
\gamma \odot \frac{x-\mu}{\sqrt{\sigma^2+\epsilon}} + \beta
$$

其中 $\mu,\sigma^2$ 在 hidden dimension 上计算。

为什么 NLP/LLM 常用 LayerNorm 而不是 BatchNorm：

- 序列长度可变，padding 多，BatchNorm 的 batch 统计不稳定。
- 自回归推理时 batch 和时间步变化大，BatchNorm 不方便。
- LayerNorm 对每个 token 的 hidden state 独立归一化，更适合 Transformer。

## 8.3 PostNorm vs PreNorm

这是高频考点，而且很容易写反。

原始 Transformer 使用 PostNorm：

$$
\text{PostNorm:}\quad y = \text{LN}(x + \text{Sublayer}(x))
$$

现代 LLM 多使用 PreNorm：

$$
\text{PreNorm:}\quad y = x + \text{Sublayer}(\text{LN}(x))
$$

对比：

| 方案 | 公式 | 训练稳定性 | 特点 |
|---|---|---|---|
| PostNorm | $\text{LN}(x + F(x))$ | 较难，常需要 warmup 和精细初始化 | 原始 Transformer 使用，深层时梯度更难传 |
| PreNorm | $x + F(\text{LN}(x))$ | 更稳定，适合深层大模型 | 现代 LLM 主流，通常最后再接一个 final norm |

PreNorm 更稳定的原因：残差主路径没有被 Norm 阻断，梯度可以更直接地沿着 $x$ 回传。PostNorm 中每层输出都经过 LN，深层训练时梯度更容易受归一化影响。

但 PreNorm 也不是完美的。非常深时，残差路径可能过强，导致某些层的有效贡献变小。因此现代模型还会配合合理初始化、学习率调度、残差缩放、final norm 等技巧。

## 8.4 RMSNorm

RMSNorm 去掉均值中心化，只按均方根缩放：

$$
\text{RMSNorm}(x) =
\frac{x}{\sqrt{\frac{1}{d}\sum_i x_i^2 + \epsilon}} \odot \gamma
$$

优点：

- 计算更简单。
- 在 LLM 中效果通常与 LayerNorm 接近或更好。
- 与 PreNorm、SwiGLU、RoPE 一起成为很多现代 LLM 的常见组合。

## 8.5 一个现代 Decoder Block 的伪代码

```python
def decoder_block(x):
    x = x + self_attention(rms_norm(x), causal_mask=True)
    x = x + swiglu_mlp(rms_norm(x))
    return x
```

最终模型通常还有：

```python
x = final_norm(x)
logits = x @ embedding_table.T
```

很多语言模型会使用 tied embedding，即输入 embedding 和输出 lm head 共享权重，以减少参数并提升一致性。

---

# 9. 训练目标：模型到底在学什么

Transformer 是架构，不等于 LLM。LLM = Transformer 架构 + tokenizer + 预训练目标 + 大规模数据 + 对齐/后训练 + 推理系统。

## 9.1 Decoder-only 的 Next Token Prediction

给定 token 序列：

```text
x1, x2, x3, ..., xt
```

训练目标是预测下一个 token：

$$
P(x_t \mid x_{<t})
$$

损失函数通常是交叉熵：

$$
\mathcal{L} =
-\sum_t \log P(x_t \mid x_{<t})
$$

通过 Causal Mask，模型在训练时可以并行预测每个位置的下一个 token。

## 9.2 Encoder-only 的 Masked Language Modeling

BERT 类模型随机 mask 掉一部分 token，让模型根据双向上下文恢复它们：

```text
我 喜欢 [MASK] 学习
```

这种训练让模型很擅长理解任务，但天然不适合逐 token 生成长文本。

## 9.3 Encoder-Decoder 的 Seq2Seq

Encoder 读入源序列，Decoder 自回归生成目标序列：

```text
source: English sentence
target: Chinese sentence
```

Decoder 通过 Cross-Attention 读取 Encoder hidden states，因此很适合翻译、摘要、改写等输入输出结构明确的任务。

---

# 10. 工程含义：推理时为什么需要 KV Cache

训练时，我们通常一次性处理完整序列。推理时，Decoder-only 模型逐 token 生成，如果每一步都重新计算整个历史，会非常浪费。

KV Cache 的核心思想：历史 token 的 Key 和 Value 一旦算出来，就缓存起来，后续生成只计算新 token 的 Query、Key、Value。本节只建立直觉：KV Cache 是 Transformer 自回归推理的工程缓存，不是 Transformer Block 的新结构。

## 10.1 没有 KV Cache 会怎样

生成第 $t$ 个 token 时，如果重新把 $1 \sim t$ 的所有 token 喂进模型，每一步都重复计算历史 K/V。

总开销会非常高：

```text
step 1: 计算 1 个 token
step 2: 重新计算 2 个 token
step 3: 重新计算 3 个 token
...
```

## 10.2 有 KV Cache 后

每层缓存历史：

$$
K_{cache}, V_{cache}
$$

生成新 token 时：

1. 只对新 token 计算 $q_t, k_t, v_t$。
2. 把 $k_t, v_t$ append 到 cache。
3. 用 $q_t$ attend 到所有历史 $K_{cache}$。
4. 输出下一个 token 的 logits。

这样避免了重复计算历史 token 的 K/V，大幅提高 decode 阶段速度。

## 10.3 Prefill 和 Decode

LLM 推理通常分两个阶段：

| 阶段 | 输入 | 特点 | 主要瓶颈 |
|---|---|---|---|
| Prefill | 完整 prompt | 并行度高，计算 prompt 所有 hidden states 并建立 KV Cache | 算力 |
| Decode | 每次 1 个新 token | 串行生成，依赖上一步输出 | 显存带宽和 KV Cache 读取 |

这解释了为什么长 prompt 的首 token 延迟高，而长回答的生成速度更受 KV Cache 和显存带宽影响。

## 10.4 KV Cache 显存估算

近似公式：

$$
\text{KV Cache bytes}
= B \times L \times 2 \times n_{layers} \times n_{kv\_heads} \times d_{head} \times bytes
$$

其中：

- $B$：batch size。
- $L$：上下文长度。
- $2$：K 和 V 两份缓存。
- $n_{layers}$：层数。
- $n_{kv\_heads}$：KV head 数，MHA 中通常等于 attention heads，GQA/MQA 中更少。
- $d_{head}$：每个 head 的维度。
- $bytes$：每个元素字节数，例如 FP16/BF16 是 2。

这就是长上下文和高并发推理昂贵的根本原因之一。

---

# 11. 复杂度分析

设：

- 序列长度：$n$
- 模型维度：$d$
- FFN 中间维度：$d_{ff}$

单层 Transformer 大致复杂度：

| 模块 | 时间复杂度 | 空间复杂度 | 说明 |
|---|---|---|---|
| Q/K/V/O 投影 | $O(nd^2)$ | $O(nd)$ | 线性层 |
| Attention Score | $O(n^2d)$ | $O(n^2)$ | 计算 $QK^T$ 和注意力矩阵 |
| Attention 加权 V | $O(n^2d)$ | $O(n^2)$ | $AV$ |
| FFN | $O(ndd_{ff})$ | $O(nd_{ff})$ | 若 $d_{ff}=4d$，计算量很大 |

当 $n$ 很大时，$n^2$ Attention 成为瓶颈。当 $d$ 很大但 $n$ 中等时，线性层和 FFN 也会占大量计算。

## 11.1 Transformer 与 RNN/CNN 对比

| 模型 | 单层复杂度 | 最大路径长度 | 并行能力 | 长距离建模 |
|---|---|---:|---|---|
| RNN | $O(nd^2)$ | $O(n)$ | 差 | 容易衰减 |
| CNN | $O(knd^2)$ | $O(\log_k n)$ 或多层堆叠 | 好 | 依赖感受野 |
| Transformer | $O(n^2d + nd^2)$ | $O(1)$ | 很好 | 任意 token 直接交互 |

最大路径长度指两个远距离 token 发生信息交互需要经过多少步。Transformer 中任意两个 token 在一层 Attention 内就能直接交互，所以路径长度是 $O(1)$。

## 11.2 FlashAttention 解决了什么

普通 Attention 会显式存储 $n \times n$ 注意力矩阵，长序列时显存压力很大。FlashAttention 的核心是 IO-aware exact attention：

- 不改变 Attention 数学结果。
- 分块计算 softmax 和 $AV$。
- 避免把完整 Attention 矩阵写入显存。
- 显著降低显存占用并提升速度。

它解决的是 Attention 的显存和带宽问题，不是把理论复杂度从 $O(n^2)$ 变成 $O(n)$。

---

# 12. 同族概念分组：现代 LLM 中常见的 Transformer 配方

现代 Decoder-only LLM 通常不是原始 Transformer 的完全复刻，而是一组经过工程验证的组合。

常见配方：

- Decoder-only Causal LM。
- Token embedding + RoPE。
- PreNorm，常用 RMSNorm。
- Multi-Head Attention / GQA / MQA。
- SwiGLU MLP。
- Causal Mask。
- KV Cache。
- FlashAttention 或类似高效 attention kernel。
- 最后一层 final norm。

## 12.1 典型组件对比

| 模型家族 | 架构 | 位置方案 | Norm | MLP | 备注 |
|---|---|---|---|---|---|
| BERT | Encoder-only | Learned absolute PE | LayerNorm | GELU FFN | 双向理解模型 |
| GPT-2/3 类 | Decoder-only | Learned absolute PE 或变体 | Pre-LN 常见 | GELU/MLP | 早期生成模型代表 |
| T5 | Encoder-Decoder | Relative position bias | RMSNorm 风格 | FFN/GEGLU 变体 | 文本到文本框架 |
| Llama 类 | Decoder-only | RoPE | RMSNorm | SwiGLU | 现代开源 LLM 常见范式 |
| Qwen 类 | Decoder-only | RoPE 及长上下文扩展 | RMSNorm | SwiGLU | 多语言、长上下文优化 |
| MoE 模型 | 多为 Decoder-only | RoPE 常见 | RMSNorm 常见 | Sparse Experts | 总参数大，激活参数较少 |

注：具体模型实现会随版本变化，上表是公开模型中常见的架构模式，不应当当作所有版本的精确配置表。

---

# 13. 工程含义：一个 token 怎么走完模型

以 Decoder-only LLM 为例：

1. **Tokenization**
   文本被 tokenizer 切成 token id。

2. **Embedding**
   token id 查表变成向量。

3. **Position 注入**
   RoPE 通常在每层 Attention 中作用于 Q/K，而不是直接加到 embedding。

4. **第 1 层 Attention**
   当前 token 根据 causal mask 只能读取历史 token 信息。

5. **第 1 层 FFN**
   对每个位置独立做非线性变换。

6. **重复 N 层**
   浅层常学局部和词法模式，中高层逐步形成更抽象的语义、推理和任务表示。

7. **Final Norm**
   稳定输出尺度。

8. **LM Head**
   将 hidden state 投影到词表大小，得到每个候选 token 的 logits。

9. **Sampling / Decoding**
   通过 greedy、temperature、top-p、top-k 等策略选出下一个 token。

---

# 14. 常见误区

## 14.1 Attention 不是全部

论文名叫《Attention Is All You Need》，但实际 Transformer 离不开：

- 残差连接。
- Normalization。
- FFN/MLP。
- 位置编码。
- 合适的初始化、优化器、学习率调度。

只有 Attention 的模型表达能力和训练稳定性都不够。

## 14.2 Attention 权重不等于解释

Attention 权重能提供一些直觉，但不能简单地说“权重最高的 token 就是模型推理原因”。多层、多头、残差、FFN 会不断重写信息流，最终行为不由某一层注意力矩阵单独决定。

## 14.3 Transformer 不天然理解顺序

顺序感来自位置编码或位置偏置。没有位置机制，Self-Attention 对顺序是置换等变的。

## 14.4 长上下文不只是改位置编码

长上下文能力同时受以下因素影响：

- 位置编码是否能外推。
- 训练时是否见过长序列。
- Attention kernel 是否能承受长序列显存。
- KV Cache 显存是否可接受。
- 数据中是否有需要远距离依赖的样本。
- 模型是否学会在长文本中检索和利用信息。

---

# 15. 面试 Q&A

## Q1：Self-Attention 的计算过程是什么？

**答**：输入 $X$ 先通过三个线性层得到 $Q,K,V$；然后计算 $QK^T$ 得到 token 两两相似度；除以 $\sqrt{d_k}$ 稳定数值；加 mask 屏蔽不可见位置；softmax 得到注意力权重；最后乘以 $V$ 得到每个 token 汇聚上下文后的表示。

## Q2：Q/K/V 分别代表什么？

**答**：Q 是当前 token 发出的查询，K 是每个 token 用来被匹配的索引，V 是每个 token 真正贡献的信息。Attention 先用 Q 和 K 决定“看谁”，再用权重对 V 加权求和决定“拿什么内容”。

## Q3：为什么要除以 $\sqrt{d_k}$？

**答**：点积的方差会随维度 $d_k$ 增大而增大，导致 softmax 饱和、梯度变小。除以 $\sqrt{d_k}$ 可以把分数尺度归一化，让 softmax 更平滑，训练更稳定。

## Q4：为什么 Decoder 要用 Causal Mask？

**答**：自回归生成中，第 $t$ 个位置只能依赖 $t$ 之前的 token，不能看到未来答案。Causal Mask 在训练时屏蔽未来位置，使模型并行训练时仍满足生成约束。

## Q5：Multi-Head Attention 为什么比单头好？

**答**：多头让模型在多个子空间中学习不同依赖模式，例如局部关系、长距离指代、语法关系等。固定总维度时，多头通常不是简单增加参数，而是改变表示分解方式，提高信息路由能力。

## Q6：FFN 在 Transformer 中有什么作用？

**答**：Attention 负责跨 token 聚合信息，FFN 负责对每个 token 的上下文表示做非线性变换。FFN 提供主要的非线性表达能力，也被认为存储了大量模式和事实知识。

## Q7：为什么 Transformer 需要位置编码？

**答**：Self-Attention 本身对 token 顺序不敏感。没有位置编码，模型无法区分“人咬狗”和“狗咬人”。位置编码把顺序信息注入模型，使注意力计算能够感知 token 的相对或绝对位置。

## Q8：RoPE 和绝对位置编码的区别是什么？

**答**：绝对位置编码通常把位置向量加到输入 embedding 上；RoPE 则旋转 Q/K，让 Attention 内积直接包含相对位置信息。RoPE 更适合自回归模型和长度扩展，但长上下文仍需要 scaling、训练和工程优化配合。

## Q9：PreNorm 为什么比 PostNorm 更稳定？

**答**：PreNorm 的残差主路径是 $x + F(\text{LN}(x))$，梯度可以沿残差连接更直接回传；PostNorm 是 $\text{LN}(x+F(x))$，深层时梯度更容易受归一化影响，需要更精细的学习率 warmup 和初始化。

## Q10：KV Cache 解决了什么问题？

**答**：KV Cache 缓存历史 token 在每一层的 Key 和 Value。生成新 token 时只计算新 token 的 Q/K/V，并用新 Q attend 到历史 K/V，避免重复计算整个历史序列，大幅提升自回归推理速度。

## Q11：Transformer 的主要瓶颈是什么？

**答**：训练长序列时，Attention 的 $O(n^2)$ 时间和空间复杂度是主要瓶颈；推理长上下文和高并发时，KV Cache 显存和带宽是主要瓶颈；大模型整体训练中，FFN 和线性投影也会占据大量 FLOPs。

## Q12：为什么现在主流 LLM 多用 Decoder-only？

**答**：Decoder-only 的 next token prediction 目标简单统一，能利用几乎所有文本数据；自回归接口天然适合生成；推理时 KV Cache 高效；规模扩大后同一生成式模型可以覆盖理解、推理、对话、代码和工具调用等任务。

---

# 16. 手写 Attention 伪代码

```python
import math

def scaled_dot_product_attention(q, k, v, mask=None):
    # q: [batch, heads, query_len, head_dim]
    # k: [batch, heads, key_len, head_dim]
    # v: [batch, heads, key_len, head_dim]
    scores = q @ k.transpose(-2, -1)
    scores = scores / math.sqrt(q.size(-1))

    if mask is not None:
        scores = scores.masked_fill(mask == 0, float("-inf"))

    attn = scores.softmax(dim=-1)
    out = attn @ v
    return out
```

现代实现中通常不会真的显式保存完整 `attn` 矩阵，而是使用 FlashAttention 等 kernel 做分块计算，节省显存并提升速度。

---

# 17. 学习路线建议

如果要真正吃透 Transformer，可以按这个顺序学：

1. 先手算一遍 $QK^T$、softmax、乘 $V$ 的小例子。
2. 理解 mask 为什么加在 softmax 前。
3. 明白多头只是把表示拆成多个子空间，不是魔法。
4. 把 PreNorm Block 的伪代码背下来。
5. 搞清楚 RoPE 是作用在 Q/K 上，不是加在 token embedding 上。
6. 搞清楚训练并行和推理串行的区别。
7. 用 KV Cache 解释为什么 LLM 首 token 慢、后续 token 逐个吐。
8. 用复杂度公式解释长上下文为什么贵。

---

# 18. 相关链接

- [[README|LLM 基础目录]]
- [[03-LLM-Variants|主流模型差异（GPT/BERT/T5/Llama/DeepSeek）]]
- [[05-Generation-Strategies|生成策略（Greedy/Beam/Top-p/Temperature）]]
- [[07-Length-Extrapolation|长度外推（NTK/YaRN/位置插值）]]
- [[06-Context-Window-KV-Cache|上下文窗口与 KV Cache]]

---

> **参考**
> - Vaswani et al. "Attention Is All You Need" (NeurIPS 2017)
> - Devlin et al. "BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding" (2018)
> - Raffel et al. "Exploring the Limits of Transfer Learning with a Unified Text-to-Text Transformer" (2020)
> - Su et al. "RoFormer: Enhanced Transformer with Rotary Position Embedding" (2021)
> - Press et al. "Train Short, Test Long: Attention with Linear Biases" (2021)
> - Shazeer. "Fast Transformer Decoding: One Write-Head is All You Need" (2019)
> - Dao et al. "FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness" (2022)
