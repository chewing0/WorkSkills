---
tags:
  - LLM
  - Fine-tuning
  - Training
  - Optimization
created: 2026-05-25
description: LLM 微调训练工程与优化基础，讲清显存构成、混合精度、优化器、梯度累积、checkpoint、ZeRO/FSDP、训练稳定性和常见故障
---

> **核心考点**：LLM 微调工程的核心是显存、吞吐和稳定性。理解参数、梯度、优化器状态、激活值和 KV/attention 成本，才能判断为什么 OOM、为什么慢、为什么 loss 不稳定。

---

# 1. 核心问题：训练为什么这么吃显存

训练时显存不只是存模型权重。

主要组成：

```text
模型参数
梯度
优化器状态
前向激活值
临时 buffer
batch 数据
```

对于 AdamW，优化器还要保存一阶和二阶动量，显存可能远大于模型权重本身。

推理只需要前向；训练需要前向、反向和参数更新，所以训练显存压力大得多。

---

# 2. Full Fine-tune vs LoRA 的显存差异

全量微调：

```text
所有参数都要梯度
所有参数都有优化器状态
checkpoint 保存完整模型
```

LoRA：

```text
基座模型冻结
只有 LoRA 参数有梯度和优化器状态
checkpoint 只保存 adapter
```

所以 LoRA 显存和存储成本低很多。

但注意：前向和反向仍要经过基座模型，激活值仍然占显存。LoRA 不是免费训练。

---

# 3. 混合精度

常见精度：

- FP32。
- FP16。
- BF16。
- INT8 / INT4 量化。

训练中常用 BF16 或 FP16 降低显存和提升速度。

BF16：

- 动态范围更大。
- 通常比 FP16 更稳定。
- 需要硬件支持。

FP16：

- 显存低。
- 可能需要 loss scaling。
- 更容易数值溢出或 underflow。

量化训练如 QLoRA 主要把基座权重量化存储，但训练的小参数仍通常用较高精度。

---

# 4. Batch、梯度累积和序列长度

显存和吞吐受三个量影响很大：

```text
micro batch size
gradient accumulation steps
sequence length
```

实际全局 batch：

```text
global batch = micro_batch * grad_accum * num_gpus
```

sequence length 很关键。长序列会显著增加 attention 和激活值成本。

如果 OOM：

- 降低 micro batch。
- 降低 max sequence length。
- 开启 gradient checkpointing。
- 使用 LoRA/QLoRA。
- 使用 ZeRO/FSDP。

不要只盯 batch size，长上下文训练常常是序列长度导致 OOM。

---

# 5. Gradient Checkpointing

反向传播需要前向激活值。Gradient checkpointing 的思路：

```text
不保存所有激活
反向时重新计算部分前向
```

优点：

- 降低显存。

代价：

- 训练变慢。

适合显存不足但可以接受更长训练时间的场景。

---

# 6. 优化器和学习率

常见优化器：

- AdamW。
- Adafactor。
- 8-bit Adam。
- Paged AdamW。

学习率对微调非常敏感。

学习率过高：

- loss 抖动。
- 模型能力被破坏。
- 输出格式崩。

学习率过低：

- 学不动。
- 收敛慢。

常见策略：

```text
warmup -> decay
```

warmup 让训练初期逐步增大学习率，避免刚开始更新过猛。

---

# 7. ZeRO、FSDP 和并行训练

当模型和优化器状态太大，需要分布式训练。

## 7.1 ZeRO

ZeRO 思路是把训练状态切分到多张 GPU。

- ZeRO-1：切 optimizer states。
- ZeRO-2：再切 gradients。
- ZeRO-3：连 parameters 也切。

越高阶段节省显存越多，但通信和工程复杂度更高。

## 7.2 FSDP

FSDP（Fully Sharded Data Parallel）也会 shard 参数、梯度和优化器状态。它和 ZeRO-3 思路接近，常用于 PyTorch 生态。

## 7.3 Tensor / Pipeline Parallel

更大规模训练可能需要：

- Tensor Parallel：把单层矩阵计算切到多卡。
- Pipeline Parallel：把不同层放到不同 GPU。

应用微调通常先用 LoRA、QLoRA、ZeRO/FSDP，除非模型很大才进入更复杂并行。

---

# 8. Checkpoint 和恢复

训练必须保存：

- 模型或 adapter 权重。
- optimizer state。
- scheduler state。
- random seed。
- tokenizer / chat template。
- 训练配置。

只保存权重不够。中断恢复需要 optimizer 和 scheduler 状态，否则训练曲线会变化。

部署时则可能只需要：

- base model id。
- adapter 权重。
- tokenizer。
- 推理配置。

训练 checkpoint 和部署 artifact 要区分。

---

# 9. 训练稳定性排查

## 9.1 loss 不下降

可能原因：

- 数据格式错。
- assistant loss mask 错。
- 学习率太低。
- LoRA target module 不合适。
- 标签全是空或被截断。

## 9.2 loss 突然爆炸

可能原因：

- 学习率太高。
- FP16 数值不稳定。
- 数据异常。
- 梯度爆炸。

处理：

- 降学习率。
- 用 BF16。
- gradient clipping。
- 检查异常样本。

## 9.3 eval 变差但 train 变好

过拟合或数据泄漏。需要减少 epoch、去重、调低 rank、增加验证集覆盖。

## 9.4 输出格式崩坏

检查：

- chat template 是否一致。
- 是否只对 assistant 算 loss。
- 训练样本格式是否混乱。
- stop token 是否正确。

---

# 10. 常见误区

## 10.1 OOM 只靠换大 GPU

不一定。可以先降序列长度、batch、开启 checkpointing、LoRA/QLoRA、ZeRO/FSDP。

## 10.2 训练速度慢一定是模型太大

也可能是数据加载、tokenization、padding 浪费、长序列、checkpointing、通信开销或存储 IO。

## 10.3 checkpoint 越多越好

checkpoint 太多会占大量磁盘。需要保存关键里程碑和最优验证点。

## 10.4 分布式训练越复杂越好

复杂并行会增加通信和故障概率。能用简单 LoRA 解决，就不必一开始上复杂训练栈。

---

# 11. 面试 Q&A

## Q1：训练显存主要由哪些部分组成？

模型参数、梯度、优化器状态、激活值、临时 buffer 和 batch 数据。全量微调还要为所有参数保存梯度和优化器状态。

## Q2：gradient checkpointing 的作用是什么？

它通过少保存激活值、反向时重新计算部分前向来节省显存，代价是训练速度变慢。

## Q3：ZeRO-1/2/3 的区别是什么？

ZeRO-1 切分优化器状态，ZeRO-2 进一步切分梯度，ZeRO-3 连模型参数也切分。阶段越高越省显存，但通信和复杂度更高。

## Q4：训练 loss 不下降应该怎么排查？

先查数据和模板：chat template、loss mask、标签是否为空、样本是否被截断；再查学习率、LoRA target modules 和优化器配置。

---

# 12. 相关链接

- [[README|Fine-tuning 目录]]
- [[03-PEFT-LoRA-QLoRA-And-Adapters]]
- [[06-Evaluation-Deployment-And-Iteration]]
- [[../LLM-Basic/08-Inference-Optimization|推理优化]]

