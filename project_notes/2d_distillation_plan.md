# 2D / 2.5D 蒸馏规划：如何让 2D 模型学习 3D Merlin teacher

## 背景问题

如果最终目标是 2D 分割，直接把 Merlin 3D encoder/decoder 改成 2D 并不理想。Merlin 的预训练来自 3D CT volume，强行输入单张 slice 会损失层间上下文。

更合理的路线是：先训练或使用一个 3D teacher，再训练一个轻量 2D 或 2.5D student，让 student 学 teacher 的输出分布、边界不确定性或中间特征。

## 3D teacher 和 2D student 输出不一样，怎么学？

3D teacher 的输出是一个体数据：

- teacher logits: [C, D, H, W]

2D student 的输出是单张切片：

- student logits: [C, H, W]

它们可以按 z 轴切片对齐。对于第 z 层：

- teacher_logits[:, z, :, :] 是 [C, H, W]
- student_logits(slice_z) 也是 [C, H, W]

所以蒸馏不是让 2D student 一次学习整个 3D volume，而是让它逐 slice 学习 teacher 在对应层的 soft prediction。

## 基础 2D 蒸馏形式

输入：

- teacher 输入完整 3D volume 或 3D patch。
- student 输入单张 CT slice，或 CT+PET slice。

损失函数：

- hard loss：student prediction vs ground-truth mask slice，例如 Dice + CE。
- soft loss：student logits vs teacher logits slice，例如 KL divergence / MSE。

总损失：

loss = DiceCE(student, mask_slice) + lambda * KL(student_logits / T, teacher_logits_slice / T)

其中：

- T 是 temperature，用来软化概率分布。
- lambda 控制 teacher 知识的权重。

## 2.5D 蒸馏形式

2.5D student 输入不是单张 slice，而是一个小 slab：

- student input: CT[z-k : z+k]
- student output: mask[z]

teacher 仍输出 3D patch prediction，取中心 slice 监督 student：

- teacher center logits: teacher_logits[:, center_z, :, :]
- student logits: student_logits[:, :, :]

这样 student 保留部分 3D 上下文，但输出仍是 2D。

## 可以蒸馏什么知识

### 1. Logit distillation

最简单可靠。让 student 学 teacher 的 soft logits。

优点：实现简单，适合类别数一致的情况。

### 2. Probability distillation

让 student 学 teacher 的 softmax 概率图。

优点：直观。
缺点：概率可能过于平滑，通常 logits 更好。

### 3. Boundary / uncertainty distillation

teacher 在边界附近往往不是 0/1，而是有不确定性。student 学这个 soft boundary，可以比只学 hard mask 更稳定。

### 4. Feature distillation

让 student 的中间 feature map 学 teacher 的某层 feature。因为 3D feature 和 2D feature 形状不同，需要取 teacher 的第 z 层 feature 或做投影：

- teacher feature: [C3D, D, H, W]
- teacher feature slice: [C3D, H, W]
- projection: 1x1 conv 把 C3D 映射到 student channel
- MSE / cosine loss 对齐 feature

这个更复杂，不建议第一版就做。

## 200 人样本少时的蒸馏价值

200 人样本下，2D slice 数量看似很多，但 slice 之间强相关，不能把它们当成真正独立样本。划分必须按 patient 级别做，不能按 slice 随机划分，否则数据泄漏。

蒸馏的价值在于：

- teacher 使用 3D 上下文，减少单 slice 歧义。
- teacher soft logits 提供比 hard mask 更丰富的边界/不确定性信息。
- student 最终部署成本低。

但前提是 teacher 足够强。如果 3D teacher 本身不稳定，蒸馏会把错误也教给 student。

## 推荐蒸馏路线

### Step 1：先训练 3D teacher

使用 CT-only 或 PET+CT 的 3D Merlin/nnU-Net 模型，得到稳定的 3D segmentation teacher。

### Step 2：离线保存 teacher logits

对训练集和验证集生成 teacher logits 或 probability maps。

注意：如果是 cross-validation，teacher 给某个训练样本生成 soft label 时，最好使用没有见过该样本的 fold teacher，避免 teacher 泄漏。

### Step 3：训练 2D student

student 输入单张 slice 或 2.5D slab。监督来自：

- ground-truth mask slice。
- teacher logits slice。

### Step 4：评估 patient-level 3D mask

student 虽然逐 slice 输出，但评估时需要把所有 slices 拼回 3D volume，再按 3D Dice / HD95 / lesion recall 评估。

## 何时选择 2D、2.5D、3D

- 3D：最适合利用 Merlin，主实验。
- 2.5D：显存或部署折中，仍保留上下文。
- 2D：轻量部署、快速 baseline、蒸馏 student。

## 第一版不建议做的事

- 不建议直接把 Merlin 3D decoder 改成 2D decoder。
- 不建议 slice 级随机划分训练/验证/测试。
- 不建议在 teacher 还没训练好的情况下立刻蒸馏。
- 不建议一开始做 feature distillation，先做 logits distillation。

## 一句话总结

3D teacher 和 2D student 输出维度不同不是障碍，因为 3D 输出可以沿 z 轴切成 2D logits，与 2D student 的每张 slice 输出逐层对齐。蒸馏的关键不是复制 3D 结构，而是把 3D teacher 利用上下文形成的 soft decision 教给 2D student。
