# 基于 Merlin 预训练 CT 编码器的 PET-CT 三维分割迁移学习

本项目将 Stanford Merlin 的 3D CT 视觉编码器与 Merlin-nnUNet 分割实现合并到一个复现仓库中，用于探索 PET-CT 分割任务中的 CT-only 迁移学习。

## 当前任务定位

- 数据：约 200 例 PET-CT 3D 数据，包含 PET、CT、mask，暂无文本报告或 EHR 标签。
- 首要目标：先使用 CT 通道进行分割，验证 Merlin 预训练 3D CT encoder 是否能提升小样本分割表现。
- 暂不做：CT-文本对齐、报告生成、EHR phenotype 预测、多模态文本融合。

## 建议实验路线

1. CT-only 3D nnU-Net baseline。
2. CT-only Merlin encoder + nnU-Net decoder，先冻结大部分 encoder。
3. Adapter/partial fine-tune：训练 decoder 与轻量 adapter，必要时解冻 layer4。
4. PET+CT 双通道融合，作为后续增强实验。
5. 2.5D 或 2D 仅作为显存/部署成本受限时的对照，不作为主路线。

## 代码结构

- merlin/：StanfordMIMI Merlin 官方代码。
- external/merlin-nnunet/：Merlin-nnUNet 分割 fork，包含 nnUNetTrainerMerlin 与 Merlin encoder + decoder 集成。
- documentation/：Merlin 官方推理和数据说明。

## 初步结论

Merlin 的核心价值在于 3D CT encoder 及其多层 skip features。若输入退化为单张 2D slice，3D 预训练优势会明显损失。对本数据集，主线应优先选择 3D patch-based segmentation；2.5D 可作为显存折中；纯 2D 更适合作为 baseline 或 student distillation，而不是直接复用 Merlin 3D decoder。
