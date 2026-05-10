---
tags:
  - LLM
  - Transformer
  - 面试八股
created: 2026-05-06
description: Transformer 核心架构详解，覆盖 Attention、位置编码、Norm、FFN 及面试高频考点
---
> **核心考点**：Self-Attention 计算过程、多头注意力意义、位置编码方案演进、PreNorm vs PostNorm、复杂度分析。

---

# 1. 整体架构概览

Transformer 由 Vaswani 等人在 2017 年提出（《Attention Is All You Need》），完全基于 **Attention 机制**，摒弃了 RNN/CNN 的序列依赖，实现了高度并行化。

```text
Input Embedding
    │
    ├─► [Encoder Stack] × N          ├─► [Decoder Stack] × N
    │      ├─ Multi-Head Attention      │      ├─ Masked Multi-Head Attention
    │      ├─ Add & Norm                │      ├─ Add & Norm
    │      ├─ Feed Forward              │      ├─ Multi-Head Attention (Cross)
    │      └─ Add & Norm                │      ├─ Add & Norm
    │                                   │      ├─ Feed Forward
    │                                   │      └─ Add & Norm
    │                                   │
    └───────────────────────────────────┘
```

| 组件              | Encoder | Decoder   | 作用             |
| --------------- | ------- | --------- | -------------- |
| Self-Attention  | ✓ (双向)  | ✓ (因果/单向) | 计算 token 间关联权重 |
| Cross-Attention | ✗       | ✓         | 解码器关注编码器输出     |
| FFN             | ✓       | ✓         | 逐 token 非线性变换  |
| LayerNorm       | ✓       | ✓         | 稳定训练、加速收敛      |

---

# 2. Self-Attention 机制

## 2.1 核心思想

Self-Attention（自注意力）让序列中的每个 token 都能**直接**与其他所有 token 交互，捕获长距离依赖，而不像 RNN 那样逐步传递。

## 2.2 Q/K/V 计算

对每个输入 token 的嵌入向量 $x \in \mathbb{R}^{d_{model}}$，通过三个可学习的权重矩阵投影为：

$$
\begin{aligned}
Q &= X W^Q \quad (W^Q \in \mathbb{R}^{d_{model} \times d_k}) \\
K &= X W^K \quad (W^K \in \mathbb{R}^{d_{model} \times d_k}) \\
V &= X W^V \quad (W^V \in \mathbb{R}^{d_{model} \times d_v})
\end{aligned}
$$

> **面试点**：Q/K/V 的含义
> - **Q（Query）**：当前 token "我要查什么"
> - **K（Key）**：每个 token "我是什么"
> - **V（Value）**：每个 token "我携带的信息"

## 2.3 Scaled Dot-Product Attention

$$
\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V
$$

**为什么要除以 $\sqrt{d_k}$？**

当 $d_k$ 较大时，$QK^T$ 的点积值方差会随维度增大而增大，导致 softmax 进入梯度极小的饱和区（接近 one-hot）。缩放后数值更平滑，梯度更稳定。

> 数学上：假设 $q, k$ 各分量独立，均值为 0，方差为 1，则 $q \cdot k$ 的均值为 0，**方差为 $d_k$**。除以 $\sqrt{d_k}$ 将方差归一化为 1。

## 2.4 Masked Self-Attention（Decoder）

 Decoder 在训练时需要避免看到未来 token，通过对 Attention Score 施加**上三角掩码**实现：

$$
\text{Mask}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}} + M\right)V, \quad M_{ij} = \begin{cases} 0 & i \ge j \\ -\infty & i < j \end{cases}
$$

---

# 3. Multi-Head Attention（MHA）

## 3.1 机制

将 Q/K/V 投影到 $h$ 个低维子空间，分别做 Attention，再拼接线性变换：

$$
\begin{aligned}
\text{MultiHead}(Q, K, V) &= \text{Concat}(\text{head}_1, ..., \text{head}_h)W^O \\
\text{head}_i &= \text{Attention}(QW_i^Q, KW_i^K, VW_i^V)
\end{aligned}
$$

其中 $d_k = d_v = d_{model} / h$。原始论文中 $d_{model}=512, h=8$，即每头维度 64。

## 3.2 多头的意义

1. **多子空间表示**：不同头关注不同的语义关系（句法、指代、语义等）。
2. **扩展表达能力**：单个 Attention 易受噪声主导，多头提供ensemble效果。
3. **并行计算**：各 head 可并行计算，不增加时间复杂度（增加参数量）。

> **面试高频**：为什么不用一个大的 Attention 而要分多头？
> 答：直接增大 $d_k$ 虽然能增加容量，但计算复杂度和显存开销更大；多头机制在相同计算量下，让模型在不同表示子空间中独立学习多种依赖模式，更具表达效率。

## 3.3 复杂度分析

| 操作 | 时间复杂度 | 空间复杂度 |
|------|-----------|-----------|
| Self-Attention | $O(n^2 \cdot d)$ | $O(n^2)$（注意力矩阵） |
| RNN（每步） | $O(n \cdot d^2)$ | $O(n)$ |
| CNN（kernel=k） | $O(k \cdot n \cdot d^2)$ | $O(k \cdot d)$ |

> $n$：序列长度；$d$：模型维度
> 
> **关键结论**：Attention 的 $O(n^2)$ 复杂度是长上下文的主要瓶颈，后续有各种优化（稀疏注意力、线性注意力、Ring Attention 等，见 [[01-Transformer#4.5 上下文窗口优化|上下文窗口]]）。

---

# 4. Position Encoding（位置编码）

Transformer 本身**不具备序列顺序感知能力**（Self-Attention 是位置无关的），需要显式注入位置信息。

## 4.1 正弦/余弦位置编码（Sinusoidal）

原始 Transformer 使用固定函数：

$$
\begin{aligned}
PE_{(pos, 2i)} &= \sin\left(\frac{pos}{10000^{2i/d_{model}}}\right) \\
PE_{(pos, 2i+1)} &= \cos\left(\frac{pos}{10000^{2i/d_{model}}}\right)
\end{aligned}
$$

**优点**：
- 无需学习参数，可外推到更长序列（理论上任意长度）
- 具有相对位置信息：$PE_{pos+k}$ 可表示为 $PE_{pos}$ 的线性变换

**缺点**：
- 外推效果有限，超出训练长度后性能下降明显
- 不是真正的平移不变性

## 4.2 可学习位置编码（Learned PE）

BERT/GPT 采用：为每个位置 $pos \in [0, max\_len)$ 学习一个向量，与词嵌入相加。

**优点**：灵活，模型可自主学习最适合的位置表示。
**缺点**：
- 无法外推到训练长度之外（max_len 固定）
- 参数量随 max_len 增加（通常可忽略）

## 4.3 Rotary Position Embedding（RoPE，旋转位置编码）

苏剑林提出，现被 Llama、Qwen、Baichuan 等主流模型采用。

**核心思想**：通过**旋转矩阵**将相对位置信息编码到 Attention 的内积中，而非直接加在输入上。

对二维情况，将向量 $(x_q^{(1)}, x_q^{(2)})$ 旋转角度 $m\theta$：

$$
R_{\Theta, m} = \begin{pmatrix} \cos m\theta & -\sin m\theta \\ \sin m\theta & \cos m\theta \end{pmatrix}
$$

高维通过分块旋转实现。最终使得：

$$
\langle R_{\Theta, m} q, R_{\Theta, n} k \rangle = g(q, k, m - n)
$$

即 Attention Score **仅依赖于相对位置 $m-n$**。

**优点**：
- 真正的相对位置编码
- 支持一定程度的长度外推（通过调整 base frequency $\theta$）
- 与 Attention 内积天然兼容

**NTK-aware 扩展**：通过非线性插值调整旋转频率，改善长文本外推能力。

## 4.4 ALiBi（Attention with Linear Biases）

直接向 Attention Score 添加与距离成**线性负相关**的偏置：

$$
\text{softmax}\left(\frac{QK^T}{\sqrt{d_k}} - m \cdot \text{distance}\right)
$$

其中 $m$ 是预设的斜率（不同头用不同斜率）。

**优点**：
- **无需位置编码向量**，直接通过偏置实现位置感知
- **极强的长度外推能力**：训练时短序列，测试时可直接推广到长序列
- 被 BLOOM、MPT 等模型采用

**缺点**：
- 对绝对位置信息建模较弱
- 长距离依赖的权重被强制压低，可能影响某些任务

## 4.5 方案对比总结

| 方案 | 类型 | 是否需要学习参数 | 外推能力 | 代表模型 |
|------|------|----------------|---------|---------|
| Sinusoidal | 绝对 | 否 | 一般 | 原始 Transformer |
| Learned PE | 绝对 | 是 | 差 | BERT、GPT-2 |
| RoPE | 相对 | 否 | 好（+NTK/YaRN 更强） | Llama、Qwen、Baichuan |
| ALiBi | 相对偏置 | 否 | **强** | BLOOM、MPT |

> **面试高频**：为什么 LLaMA 用 RoPE 而不是绝对位置编码？
> 答：RoPE 将相对位置信息内嵌到 Attention 内积中，具有更好的外推性和平移不变性；而绝对位置编码在超出训练长度时性能急剧下降。

---

# 5. Feed Forward Network（FFN）

## 5.1 标准结构

$$
\text{FFN}(x) = \max(0, xW_1 + b_1)W_2 + b_2
$$

即两层全连接 + ReLU 激活，中间维度通常为 $4 \times d_{model}$（如 512 → 2048）。

## 5.2 现代变种

| 变种 | 公式 | 特点 | 使用模型 |
|------|------|------|---------|
| **GELU** | $x \Phi(x)$，$\Phi$ 为标准正态 CDF | 平滑、更接近随机正则化 | BERT、GPT-3 |
| **SwiGLU** | $\text{Swish}_1(xW) \odot (xV)$ | 门控机制，效果更稳定 | PaLM、LLaMA、Qwen |
| **GEGLU** | $\text{GELU}(xW) \odot (xV)$ | GLU 变体 | PaLM |

> **GLU 系列**将单个隐藏层拆分为两个并行投影再逐元素乘，参数量和计算量略有增加，但收敛更快、效果更优。

## 5.3 FFN 的作用

虽然 Attention 做全局信息聚合，但 FFN 负责：
1. **增加非线性**：提升模型表达能力（Attention 本质是加权平均，线性变换）
2. **逐 token 变换**：在统一语义空间后，对每个位置独立做复杂映射
3. **存储知识**：有研究表明 FFN 近似键值记忆网络，存储了大量事实知识

---

# 6. Normalization：LayerNorm / PreNorm / PostNorm

## 6.1 LayerNorm

对**单个样本的所有特征**做归一化（与 BatchNorm 不同，不依赖 batch 维度）：

$$
\text{LayerNorm}(x) = \gamma \odot \frac{x - \mu}{\sqrt{\sigma^2 + \epsilon}} + \beta
$$

其中 $\mu, \sigma^2$ 是对 $d_{model}$ 维度计算的均值和方差。$\gamma, \beta$ 为可学习参数。

**为什么用 LayerNorm 不用 BatchNorm？**
- NLP 中序列长度不一，batch 内 padding 较多，BatchNorm 统计不稳定
- LayerNorm 对样本独立，适合变长序列和自回归生成

## 6.2 PostNorm vs PreNorm

原始 Transformer 使用 **PostNorm**：

```
PostNorm:  x + Sublayer(LayerNorm(x))   ❌ 原始论文
PreNorm:   x + Sublayer(x)  → LayerNorm   ✅ 现代主流
```

| 方案 | 结构 | 训练稳定性 | 收敛速度 | 最终效果 | 代表 |
|------|------|-----------|---------|---------|------|
| PostNorm | `LN(x + SubLayer(x))` | 差，需配合 Warmup | 慢 | 略好 | 原始 Transformer |
| PreNorm | `x + SubLayer(LN(x))` | **好**，可大学习率 | **快** | 接近 | GPT-3、Llama、主流模型 |

**PostNorm 的问题**：残差路径经过 LN，梯度在深层会被抑制，需要精心设计学习率预热。

**PreNorm 的问题**：深层时残差连接主导，实质层数变浅，可能略损失容量（实际中可通过增加层数补偿）。

> 现代 LLM（GPT、Llama、Qwen、DeepSeek）**全部使用 PreNorm**。

## 6.3 RMSNorm（Root Mean Square LayerNorm）

LLaMA 等模型采用，去除均值偏移计算，只保留 RMS：

$$
\text{RMSNorm}(x) = \frac{x}{\sqrt{\frac{1}{d}\sum_i x_i^2 + \epsilon}} \cdot \gamma
$$

**优点**：
- 计算量略低（无需计算均值）
- 在 LLM 中效果与 LayerNorm 相当甚至略优
- 与 SwiGLU 等模块搭配成为现代 LLM 标配

---

# 7. 主流架构范式对比

| 特性 | Encoder-only (BERT) | Decoder-only (GPT/Llama) | Encoder-Decoder (T5/BART) |
|------|---------------------|-------------------------|--------------------------|
| **注意力** | 双向 | 因果（单向） | Encoder 双向 + Decoder 因果 |
| **预训练任务** | MLM（掩码语言模型） | Next Token Prediction | Span Corruption |
| **典型应用** | 理解任务（分类、NER、NLI） | 生成任务（对话、续写、代码） | 翻译、摘要、多任务 |
| **代表模型** | BERT、RoBERTa、ALBERT | GPT-4、Llama、Qwen、DeepSeek | T5、BART、GLM |
| **参数效率** | 中等 | **高**（统一架构，Causal LM 一统江湖） | 中等 |

> **趋势**：2023 年后，**Decoder-only 架构占据绝对主导**。原因：
> 1. 生成式预训练（GPT）可扩展性极强，数据获取成本低
> 2. Causal Attention 实现简单，推理时可高效 KV-Cache
> 3. 强大的涌现能力随规模出现，统一了理解与生成

## 7.1 现代 Decoder-only 模型特点

| 模型 | 位置编码 | Norm | 激活函数 | 特色 |
|------|---------|------|---------|------|
| **LLaMA** | RoPE | Pre-RMSNorm | SwiGLU | 开源标杆，高效实现 |
| **Qwen** | RoPE | Pre-RMSNorm | SwiGLU | 中英优化，长文本 |
| **DeepSeek** | RoPE (Yarn) | Pre-RMSNorm | SwiGLU | MLA 注意力、专家混合 |
| **GPT-4** | Learned PE | Pre-LayerNorm | GELU | 规模+数据+RLHF |

---

# 8. 面试高频 Q&A

## Q1：Self-Attention 的时间复杂度是多少？为什么比 RNN 更适合长序列并行计算？

**答**：$O(n^2 \cdot d)$。虽然序列复杂度是 $O(n^2)$，但**没有时序依赖**，矩阵乘法可高度并行（GPU 友好）；RNN 是 $O(n \cdot d^2)$ 但需逐步计算 $n$ 步，无法并行。实际中 $n$ 适中时（< 10k），Transformer 远快于 RNN。

## Q2：Transformer 中 Attention 层和 FFN 层各起什么作用？

**答**：
- **Attention**：**全局信息聚合**，让每个 token 获取全序列上下文（"找谁重要"）
- **FFN**：**非线性变换与知识存储**，对每个 token 独立做复杂映射（"具体怎么变"）
两者缺一不可：没有 Attention，模型看不到上下文；没有 FFN，模型只是加权平均，表达能力不足。

## Q3：为什么 Decoder 需要 Masked Attention？训练时和推理时有什么区别？

**答**：Decoder 是自回归生成，只能依赖已生成的 token。训练时通过上三角掩码一次性对整个序列计算；推理时由于 KV-Cache，每次只输入一个新 token，注意力自然只关注到过去（无需显式掩码，或说 causal mask 自动满足）。

## Q4：RoPE 和绝对位置编码的本质区别是什么？

**答**：绝对位置编码将位置信息**加到输入嵌入上**（$x + PE$），进入网络后可能被多层变换模糊；RoPE 将位置信息**嵌入 Attention 的内积计算中**，直接决定两个 token 的关联权重，具有更好的相对位置感知和长度外推能力。

## Q5：PreNorm 为什么比 PostNorm 更稳定？

**答**：PostNorm 中残差经过 LayerNorm，深层梯度被逐层缩放，容易消失；PreNorm 中主路径是干净的残差连接 $x + f(LN(x))$，梯度可直接回传，允许使用更大学习率、跳过 Warmup，训练更稳定。

---

# 9. 相关链接

- [[../README|返回目录]]
- [[02-LLM-Variants|主流模型差异（GPT/BERT/T5/Llama/DeepSeek）]]
- [[03-Generation-Strategies|生成策略（Greedy/Beam/Top-p/Temperature）]]
- [[04-Length-Extrapolation|长度外推（NTK/YaRN/位置插值）]]
- [[05-Context-Window|上下文窗口与 Ring Attention]]

---

> **参考**
> - Vaswani et al. "Attention Is All You Need" (NeurIPS 2017)
> - Su et al. "RoFormer: Enhanced Transformer with Rotary Position Embedding" (2021)
> - Press et al. "Train Short, Test Long: Attention with Linear Biases" (2021)
> - Xiong et al. "On Layer Normalization in the Transformer Architecture" (2020)
