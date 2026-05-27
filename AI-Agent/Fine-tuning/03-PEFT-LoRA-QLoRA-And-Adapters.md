---
tags:
  - LLM
  - Fine-tuning
  - LoRA
  - QLoRA
  - PEFT
created: 2026-05-25
description: PEFT、LoRA、QLoRA 与 Adapter 基础，讲清参数高效微调、低秩适配、量化训练、关键超参、合并部署和适用边界
---

> **核心考点**：PEFT 的目标是在尽量不更新原模型大部分参数的情况下，让模型学到目标任务变化。LoRA 通过低秩增量学习权重变化，QLoRA 通过量化基座模型进一步降低显存。

---

# 1. 核心问题：为什么需要 PEFT

全量微调会更新模型所有参数。对于 7B、14B、70B 模型，这意味着：

- 显存需求高。
- 训练成本高。
- checkpoint 很大。
- 多任务版本管理困难。
- 更容易破坏原模型能力。

PEFT（Parameter-Efficient Fine-Tuning）只训练少量新增或局部参数。

直观效果：

```text
冻结大模型主体
  │
  └─► 只训练少量适配参数
```

常见 PEFT：

- LoRA。
- QLoRA。
- Adapter。
- Prefix Tuning。
- P-Tuning。

应用中最常见的是 LoRA 和 QLoRA。

---

# 2. LoRA 的核心思想

神经网络里很多层有权重矩阵 $W$。全量微调直接更新 $W$。

LoRA 假设目标任务需要的权重变化 $\Delta W$ 可以用低秩矩阵近似：

$$
W' = W + \Delta W
$$

$$
\Delta W = B A
$$

其中：

- $W$ 冻结。
- $A$ 和 $B$ 是可训练小矩阵。
- rank $r$ 远小于原矩阵维度。

直观理解：

> 不直接改大模型原始权重，而是在旁边学一个小的“增量补丁”。

这样训练参数少很多，显存和存储成本也低很多。

---

# 3. LoRA 放在哪些层

Transformer 中常见 target modules：

- q_proj。
- k_proj。
- v_proj。
- o_proj。
- gate_proj。
- up_proj。
- down_proj。

只放 attention 层：

- 成本更低。
- 对很多任务够用。

同时放 attention + MLP：

- 表达能力更强。
- 成本更高。
- 更容易过拟合。

选择 target modules 要看任务：

- 格式和对话风格：attention LoRA 可能够。
- 领域知识和复杂行为：可能需要更多模块。
- 数据很少：不要盲目放太多模块。

---

# 4. LoRA 关键超参

| 超参 | 含义 | 影响 |
|---|---|---|
| rank r | 低秩维度 | 越大表达越强，参数越多 |
| alpha | 缩放系数 | 控制 LoRA 增量强度 |
| dropout | LoRA dropout | 防过拟合 |
| target modules | 插入哪些层 | 决定影响范围 |
| learning rate | LoRA 参数学习率 | 过高会不稳定 |

rank 不是越大越好。小任务、小数据用过大 rank 容易过拟合。

常见调参思路：

```text
先用较小 rank 跑通
  │
  ├─► 如果欠拟合，再增大 rank 或 target modules
  └─► 如果过拟合，降低 rank、epoch 或加 dropout
```

---

# 5. LoRA 合并和部署

LoRA 训练后有两种部署方式。

## 5.1 保留 adapter

```text
base model + LoRA adapter
```

优点：

- 多个任务 adapter 可切换。
- 存储成本低。
- 可组合管理。

缺点：

- 推理框架需要支持 adapter。
- 多 adapter 路由复杂。

## 5.2 Merge 到基座权重

```text
W' = W + BA
```

优点：

- 推理部署简单。
- 不需要 adapter 加载逻辑。

缺点：

- 合并后不易切换。
- 多 adapter 管理不灵活。
- 量化模型 merge 要谨慎。

---

# 6. QLoRA：量化基座 + LoRA 训练

QLoRA 的核心：

```text
基座模型权重量化存储
  │
  └─► 训练 LoRA 小参数
```

也就是说：

- 基座模型以 4-bit 等低精度加载，节省显存。
- LoRA 参数用较高精度训练。
- 反向传播只更新 LoRA 参数。

QLoRA 解决的是显存问题，让较大模型可以在较小 GPU 上微调。

但要注意：

- 训练更依赖实现细节。
- 速度不一定更快。
- 量化可能带来精度损失。
- 合并和部署要考虑量化格式。

QLoRA 适合资源有限下的 SFT，不等于所有场景最优。

---

# 7. Adapter、Prefix Tuning、P-Tuning

## 7.1 Adapter

在 Transformer 层中插入小的可训练模块。主体参数冻结。

优点：

- 模块化清晰。
- 多任务适配方便。

缺点：

- 可能增加推理延迟。
- 架构侵入比 LoRA 更明显。

## 7.2 Prefix Tuning

为每层 attention 加可训练前缀向量，让模型像看到额外上下文。

适合生成任务，但在现代大模型应用中常见度低于 LoRA。

## 7.3 P-Tuning

学习连续 prompt embedding，而不是手写离散 prompt。

优点是参数少，缺点是可解释性和通用性较弱。

---

# 8. PEFT 的适用边界

适合：

- 指令格式微调。
- 领域风格适配。
- 多任务 adapter 管理。
- 资源有限训练。
- 快速实验。

不适合：

- 需要大幅改变模型基础能力。
- 数据量极大且预算充足的深度领域训练。
- 需要完全重新塑造模型语言分布。

PEFT 不是魔法。它减少训练参数，但仍依赖数据质量和评估。

---

# 9. 常见误区

## 9.1 LoRA 只是省显存，效果一定差

不一定。很多 SFT 任务上 LoRA 效果足够好，甚至更稳，因为它减少了破坏原模型的风险。

## 9.2 rank 越大越好

不一定。rank 大表达能力强，但更容易过拟合，训练和存储成本也更高。

## 9.3 QLoRA 等于模型全程 4-bit 训练

不准确。QLoRA 通常是基座权重量化存储，训练 LoRA 参数。量化和 LoRA 分别解决不同问题。

## 9.4 adapter 可以随便叠加

多 adapter 组合可能互相干扰。需要明确路由、合并策略和评估。

---

# 10. 面试 Q&A

## Q1：LoRA 为什么能降低训练成本？

LoRA 冻结原模型权重，只训练低秩矩阵表示的权重增量。训练参数、优化器状态和 checkpoint 都大幅减少。

## Q2：LoRA 的 rank 影响什么？

rank 决定低秩增量的表达能力。rank 越大，能表示的变化越复杂，但参数更多、成本更高，也更容易过拟合。

## Q3：QLoRA 和 LoRA 的区别是什么？

LoRA 冻结基座并训练低秩增量；QLoRA 在此基础上把基座模型量化加载，进一步降低显存占用。QLoRA 主要解决资源受限下训练大模型的问题。

## Q4：什么时候用全量微调，什么时候用 LoRA？

多数应用任务先用 LoRA/QLoRA。只有当数据量大、预算足、需要深度改变模型能力或 PEFT 达不到目标时，再考虑全量微调。

---

# 11. 相关链接

- [[README|Fine-tuning 目录]]
- [[02-SFT-And-Instruction-Tuning]]
- [[05-Training-Systems-And-Optimization]]
- [[06-Evaluation-Deployment-And-Iteration]]

