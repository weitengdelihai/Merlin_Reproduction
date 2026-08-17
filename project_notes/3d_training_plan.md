# 3D 训练规划：基于 Merlin 预训练 CT 编码器的 PET-CT 三维分割迁移学习

## 任务定位

当前数据约 200 名病人，每例包含 PET、CT、mask，暂无文本报告或 EHR 标签。因此不做 Merlin 的多模态文本路线，首要目标是利用 Merlin 的 3D CT encoder 做小样本 CT/PET-CT 分割迁移学习。

主任务建议定义为：基于 Merlin 预训练 CT 编码器的 PET-CT 三维分割迁移学习。

## 为什么优先 3D

Merlin 的预训练优势来自完整 3D CT volume 的编码能力。其 encoder 输出的不只是最后 2048 维全局 embedding，还可以返回多层 skip features，这些 feature map 保留空间结构，适合接 3D U-Net decoder 做 dense segmentation。

如果退化为纯 2D slice 输入，3D 卷积的层间上下文会被削弱，Merlin 的预训练优势会明显损失。2.5D 可以作为显存/部署折中，但主线应先跑 3D。

## 推荐实验阶段

### Stage 0：数据准备与 baseline

- 将 CT、PET、mask 整理为 nnU-Net 或 MONAI 可读的 3D NIfTI 格式。
- 先做 CT-only 3D nnU-Net baseline。
- 如果显存允许，再做 PET+CT two-channel nnU-Net baseline。

### Stage 1：冻结 Merlin encoder，训练随机初始化 decoder

- Merlin encoder 加载官方预训练权重。
- 冻结 encoder。
- decoder 和 segmentation head 随机初始化。
- 输出头改为本任务类别数，例如 background + tumor = 2 类。

目的：验证 Merlin 预训练 CT encoder 是否对肿瘤分割有帮助。

### Stage 2：adapter + decoder 微调

- 继续加载 Stage 1 权重。
- encoder 主干仍冻结。
- 在 skip features 或 bottleneck feature 后加入轻量 adapter，例如 1x1x1 conv / bottleneck adapter。
- 训练 adapter + decoder + segmentation head。

目的：在不破坏 Merlin 主干预训练权重的情况下，让 CT 表征适配病灶分割。

### Stage 3：解冻 encoder layer4

- 继续加载 Stage 2 权重。
- 解冻 layer4。
- 训练 layer4 + adapter + decoder + head。
- conv1、layer1、layer2、layer3 继续冻结。

目的：只调整最高层语义特征，让模型适配肿瘤/病灶类别。

### Stage 4：可选解冻 layer3

- 继续加载 Stage 3 权重。
- 解冻 layer3 + layer4。
- 训练 layer3、layer4、adapter、decoder、head。

风险：200 病人样本较少，过拟合风险明显增加。只有在验证集显示 Stage 3 欠拟合时再考虑。

### Stage 5：PET 融合

- CT branch 使用 Merlin encoder。
- PET branch 使用轻量 3D encoder。
- 融合方式可从简单 concat / 1x1x1 conv 开始，再考虑 cross-attention。

建议不要一开始加 PET，因为 PET 信号强，可能掩盖 Merlin CT encoder 的真实贡献。

## 参数量粗略估计

- Merlin 3D encoder 主干 conv1-layer4：约 117M。
- layer3：约 83M。
- layer4：约 29M。
- Merlin 原任务头：约 4.5M。
- 当前 Merlin-nnUNet 3D decoder：约 134M。
- adapter：取决于设计，常见 bottleneck adapter 约 2-5M。

输出头从 23 类改成 2 类只减少约几万参数，不是主要显存来源。3D 分割主要显存消耗来自 activation 和 patch size。

## 数据划分建议

总样本约 200 人，固定测试集不宜太小，但训练集也不能太少。建议两种策略：

1. 如果目标是严谨评估：保留 20% 独立测试集，剩余 80% 做 5-fold cross-validation。
   - test：约 40 人，只在最终使用。
   - train/val：约 160 人，做 5 折。

2. 如果目标是先跑通方法：先用 160/20/20 或 140/30/30。
   - 不要频繁看 test。
   - 所有模型选择和早停只看 validation。

## 服务器建议

- 跑通代码、小 patch、冻结 encoder：RTX 4090 24GB。
- 正式 3D 实验、layer4 微调：L40S / RTX 6000 Ada / A6000，48GB 显存档。
- 大 patch、PET+CT、layer3+layer4 微调、多实验并行：A100 80GB。

## 推荐主线

1. CT-only 3D nnU-Net from scratch。
2. CT-only Merlin encoder frozen + random decoder。
3. CT-only Merlin encoder frozen + adapter + decoder。
4. CT-only Merlin layer4 unfrozen + adapter + decoder。
5. PET+CT fusion。
6. 2.5D / 2D 蒸馏作为轻量化后续。
