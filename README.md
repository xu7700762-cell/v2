# VestibularFusion v27

本仓库仅保留 `v27 PolynomialKAN` 主线、参数匹配的 MLP 对照和 `seed=2001` 三数据集汇总结果。未包含 v26/v28、其他训练 seed、旧实验目录、缓存、日志、checkpoint 或原始数据。

## 模型

```text
Raw EEG window [30,1280]
        │
        ▼
Frozen pretrained FEMBA
        │
        ▼
Token mean pooling [525]
        │
        ▼
LayerNorm → Fractional-DoG PolynomialKAN(525,160,degree=2) → LayerNorm
        │
        ├── U3-U6 anchor calibration → state classification
        └── FEMBA token dynamics + bounded KAN residual → severity classification
```

正式 v27 投影名称为 `fractional_dog_polykan`。它保留 degree-2 PolynomialKAN 主映射，并以零初始化的逐特征门控加入 Fractional-DoG 基；初始映射与基础 PolynomialKAN 一致。`mlp` 仅作为截图结果中的参数匹配对照。

## 固定协议

- 数据集：`monifeixing`、`VRQ`、`city`；
- identity-disjoint 五折划分，split seed 固定为 `42`；
- 训练 seed 固定为 `2001`；
- FEMBA encoder 全程冻结并保持 `eval()`；
- 每名被试使用 U3-U6 四个 reference anchors；
- 状态任务使用 anchor-oriented subject KMeans，严重度任务使用 11 个均匀 task windows；
- checkpoint schema：`femba_kan_mtl_v27`；
- ACC 差值定义为 `KAN ACC - MLP ACC`。

## seed=2001 三数据集结果

### 状态分类

| 数据集 | KAN ACC / BACC / AUROC | MLP ACC / BACC / AUROC | ACC 差值 |
|---|---:|---:|---:|
| monifeixing | 81.07 / 81.23 / 86.16 | 79.44 / 79.54 / 84.54 | +1.63 pp |
| VRQ | 79.69 / 80.35 / 84.98 | 81.14 / 81.83 / 88.50 | -1.44 pp |
| city | 80.56 / 81.13 / 85.84 | 80.37 / 80.79 / 86.43 | +0.19 pp |
| 三数据集宏平均 | 80.44 / 80.90 / 85.66 | 80.32 / 80.72 / 86.49 | +0.13 pp |

KAN 在 monifeixing 上更好，在 VRQ 上更差，city 基本持平。状态分类宏平均 ACC 提升 `0.13 pp`，宏平均 AUROC 比 MLP 低 `0.83 pp`。

### 高低眩晕分类

| 数据集 | KAN ACC / BACC / AUROC | MLP ACC / BACC / AUROC | ACC 差值 |
|---|---:|---:|---:|
| monifeixing | 72.22 / 72.22 / 66.67 | 77.78 / 77.78 / 66.67 | -5.56 pp |
| VRQ | 73.91 / 74.24 / 81.06 | 69.57 / 70.08 / 82.58 | +4.35 pp |
| city | 75.97 / 75.97 / 78.16 | 74.66 / 74.68 / 77.97 | +1.30 pp |

以上仅记录已确认截图中可见的汇总值，不补写截图未展示的严重度宏平均。

## 运行

复制并填写本地配置：

```bash
cp configs/paths.example.json configs/paths.local.json
python -m vestibular_fusion preflight --config configs/paths.local.json
```

v27 KAN：

```bash
for dataset in monifeixing vrq city; do
  python -m vestibular_fusion train \
    --config configs/paths.local.json \
    --dataset "$dataset" \
    --training-seed 2001 \
    --projection-variant fractional_dog_polykan \
    --device cuda
done
```

参数匹配 MLP 对照只需将 `--projection-variant` 改为 `mlp`，并使用独立输出目录，避免覆盖 v27 checkpoint。

正式评估示例：

```bash
python -m vestibular_fusion evaluate \
  --config configs/paths.local.json \
  --dataset vrq \
  --checkpoint-root outputs/v27_seed2001/training/vrq \
  --output-root outputs/v27_seed2001/evaluation/vrq \
  --device cuda
```

数据、预训练 FEMBA 权重及本地运行产物由 `.gitignore` 排除，不提交到仓库。
