<div align="center">

# How Far Can Aerial Detection Go on 8 GB?

**A Systematic Ablation Study of Aerial Small-Object Detection on Consumer Hardware**

*7-class tiny-target detection from drone imagery on an RTX 5060 Laptop GPU*

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Python 3.11](https://img.shields.io/badge/python-3.11-blue.svg)](https://www.python.org/downloads/release/python-3110/)
[![PyTorch](https://img.shields.io/badge/PyTorch-%3E%3D2.1-ee4c2c.svg)](https://pytorch.org/)
[![Ultralytics](https://img.shields.io/badge/Ultralytics-8.4.12-9ACD32.svg)](https://github.com/ultralytics/ultralytics)
[![CUDA](https://img.shields.io/badge/CUDA-RTX%205060-76B900.svg)](https://developer.nvidia.com/cuda-zone/)

[Key Results](#-key-results) •
[Architecture](#-architecture) •
[Dataset](#-dataset) •
[Ablation](#-ablation-study) •
[Reproduce](#-reproducibility)

</div>

---

## 🔭 Overview

Most computer vision research assumes access to high-end GPUs (A100, 4090) and large-scale cloud
compute. This project starts from the opposite end: **a single consumer laptop (RTX 5060, 8 GB
VRAM), no cloud, no hardware upgrade.**

The initial constraint was practical — a laptop was the only compute available. But working under
it revealed something more interesting: **which design decisions actually matter when memory is
the binding constraint?** Many published detection architectures are tuned for a memory budget
that a real deployment rarely has. So the project's guiding question became:

> **How good can aerial detection get under a hard 8 GB VRAM budget, and which architectural
> decisions matter most under that constraint?**

Every design decision in this project is evaluated by both **accuracy and memory cost**, documented
through **20+ versioned ablation experiments**.

**Scope of this work.** This repository covers the full research and training pipeline, plus
**offline analysis of drone footage** on a consumer laptop — the scenario that runs today. Onboard
real-time inference on embedded UAV hardware (Jetson-class) is **future work**; the memory-conscious
design decisions documented here are made with that migration path in mind.

---

## 📊 Key Results

### Best Model — V16.0

<div align="center">

| | |
|:---|:---|
| **mAP@0.5** | **74.00%** |
| **mAP@0.5:0.95** | **52.11%** |
| **Precision** | **81.3%** |
| **Recall** | **67.6%** |
| **Parameters** | **7.03 M** |
| **Inference** | **4.0 ms** |
| **VRAM** | **6.8 GB** (batch=4, imgsz=800) |

<sub>P3+P4 dual-head + per-scale CoordAtt · RTX 5060 8 GB · FP16 · avg over 2,224 val images</sub>

</div>

### Per-Class Performance (V16.0)

| Class | AP@0.5 | AP@0.5:0.95 | Precision | Recall |
|:------|:------:|:-----------:|:---------:|:------:|
| person | 60.2% | 24.7% | 77.7% | 50.9% |
| cycle | 43.0% | 17.6% | 67.0% | 36.3% |
| bus | 91.8% | 74.5% | 91.3% | 86.3% |
| small-bus | 99.2% | 83.3% | 92.3% | 99.4% |
| car | 67.2% | 47.3% | 87.2% | 51.4% |
| truck | 60.0% | 41.0% | 63.8% | 53.6% |
| freight | 96.6% | 76.6% | 89.8% | 95.0% |
| **Overall** | **74.0%** | **52.1%** | **81.3%** | **67.6%** |

### Detection Examples

<p align="center">
  <img src="results/detection_example_0.jpg" width="32%" alt="Detection example 1" />
  <img src="results/detection_example_1.jpg" width="32%" alt="Detection example 2" />
  <img src="results/detection_example_2.jpg" width="32%" alt="Detection example 3" />
</p>

### Training & Evaluation Plots

<p align="center">
  <img src="results/training_curves.png" width="48%" alt="Training curves" />
  <img src="results/confusion_matrix_normalized.png" width="44%" alt="Confusion matrix" />
</p>

---

## 🧠 Architecture

The core improvements target three bottlenecks at once — **positional information loss**,
**small-object gradient starvation**, and **VRAM overcommitment** — applied on top of a
customized Ultralytics 8.4.12 fork.

| # | Modification | Targets | Impact |
|:-:|:-------------|:--------|:-------|
| 1 | **Coordinate Attention (CoordAtt)** | Positional info loss | +2.4% Precision, negligible params |
| 2 | **P3+P4 dual-head** | VRAM overcommitment | −13% params, +0.71% mAP vs triple-head (controlled) |
| 3 | **WIoU v3 loss** | Small-object gradient | +1.78% mAP@0.5, zero VRAM cost |
| 4 | **TAL 4 px threshold** | Small-object assignment | +6–7% on person/cycle |
| 5 | **Class weighting** | Long-tail imbalance | person ×2.0, cycle ×3.0, bus ×1.8, small-bus ×1.5 |

> 📄 All custom modifications are documented in [`patches/README.md`](patches/README.md);
> full CoordAtt source in [`patches/coordatt.py`](patches/coordatt.py).

<details>
<summary><b>Why dual-head beats triple-head on 8 GB</b> (click to expand)</summary>

<br>

A P2 high-resolution head (25,600 cells) would help small objects, but its alignment matrix
in the TaskAlignedAssigner exceeds 8 GB VRAM at batch=4. Removing the P5 head cuts parameters
from 8.06 M to 7.03 M and frees detection-head VRAM. This memory is reallocated to larger
P3/P4 channels, which improves detection of the majority of targets (person, cycle, car, truck
— over 80% of instances). The trade-off: car detection at the P4→P5 boundary degrades slightly
(67.2% vs ~71% on triple-head), which remains an open problem.

**On the size of this effect.** Our cleanest head-count comparison is **V16 (dual-head, 74.00%)
vs V20 (triple-head, 73.29%)** — a **+0.71%** mAP@0.5 gain, with the only changed variable being
the number of detection heads. An earlier version pair (V12 → V14) shows a larger +2.1% gap, but
that comparison changed five variables at once (head count, TAL threshold, class weighting,
augmentation scale, and imgsz). We report the controlled +0.71% as the honest isolated effect.

</details>

---

## 📁 Dataset

### aerial_v9 (Primary)

| Split | Images | Classes |
|:------|:------:|:-------:|
| Train | 8,075 | 7 |
| Val | 2,224 | 7 |
| Test | 738 | 7 |

**Classes:** `person · cycle · bus · small-bus · car · truck · freight`

**Class distribution (train instances, counted from labels across 8,075 images):**

```
person    69,061  ████████████████████████████████████████
car       68,280  ███████████████████████████████████████
cycle     14,191  ████████
truck     13,278  ███████
bus        6,106  ███
freight    1,180  █
small-bus    987  █
```

### Dataset History

| Dataset | Images | Classes | Source |
|:--------|:------:|:-------:|:-------|
| **aerial_v9** (current) | 8,075 / 2,224 / 738 | 7 | Assembled from 3 public sources, unified class schema |
| aerial_v8 (early) | 3,800 | 7 | Public aerial datasets, superseded |
| aerial_merged (early) | 7,148 | 5 | aerial.v1i (CC BY 4.0) + VisDrone2019 |
| aerial (early) | 2,090 | 6 | Roboflow aerial.v1i (CC BY 4.0) |

> **⚠️ Dataset availability**: aerial_v9 is curated from public aerial datasets
> (aerial.v1i CC BY 4.0, Aerial Vehicle Detection MIT, aerial.v3i MIT). Due to license terms and
> total size (~10 GB), the full dataset is **not open-sourced**. Contact the owner for access, or
> prepare a compatible dataset using [`data/data.yaml`](data/data.yaml).
>
> A **50-image CC BY 4.0 subset** is included in [`data/val_samples/`](data/val_samples/) for
> verification. See [`docs/data_cleaning_report.md`](docs/data_cleaning_report.md) for the
> curation process.

---

## 🔬 Ablation Study

Most versions isolate exactly one variable. All training on RTX 5060 8 GB, SGD lr₀=0.01 (batch=8
through V11, batch=4 from V12 onward, where the larger imgsz required it).

| Version | Key Change | mAP@0.5 | mAP@0.5:0.95 | Δ | Status |
|:-------:|:-----------|:-------:|:------------:|:--:|:------:|
| V8.0 | Baseline (clean aerial_v8, 7 classes) | 67.81% | 44.31% | — | Baseline |
| V8.1 | + WIoU v3 loss | 69.59% | 47.00% | +1.78% | ✅ |
| V10.0 | + CoordAtt + WIoU + aerial_v9 (8,075 img) | 70.51% | 49.33% | +0.92% | Milestone |
| V12.0 | imgsz 640 → 800 | 71.58% | 50.27% | +1.07% | ✅ |
| V12.1 | + copy_paste / multi-scale fine-tune | 70.89% | 49.29% | −0.69% | ❌ Failed |
| V12.2 | Continue V12.0 (cls 0.5 → 0.7) | 72.10% | 50.43% | +0.52% | ✅ |
| V14.0 | P3+P4 dual-head — *5 changes bundled* † | 73.71% | **52.65%** | +1.61% | Peak mAP₅₀₋₉₅ |
| V14.1 | car weight 1.0 → 1.3 (fine-tune) | 73.69% | 52.62% | −0.02% | Neutral |
| **V16.0** | **+ per-scale CoordAtt, imgsz=800** | **74.00%** | 52.11% | +0.29% | **🏆 Best** |
| V17.0 | + P4 RepNCSPELAN4 | 73.97% | 51.56% | −0.03% | Neutral |
| V18.0 | + ECA / ASFF attention | 73.46% | 51.16% | −0.51% | Neutral |
| V19.0 | P4 wide channel 256 → 384 | 73.29% | 50.89% | −0.17% | Neutral |
| V20.0 | P3+P4+P5 triple-head | 73.29% | 50.41% | −0.71% (vs V16) | Neutral |

> 📄 **Complete 20+ version log** with per-class breakdowns, failed experiments, and full training
> configuration → [`docs/ablation_table.md`](docs/ablation_table.md)
>
> **† On the V14 jump.** V14's +2.13% bundles five simultaneous changes (head removal, TAL
> threshold, class weighting, augmentation scale, imgsz 800→960) — it is not a single-variable
> ablation. The *isolated* effect of the head-count change is measured by the V16-vs-V20 pair:
> **+0.71%**. See [Why dual-head beats triple-head](#-architecture).
>
> **On version numbering.** This table starts at V8.0. Versions V1–V6 used the 6-class `aerial`
> dataset and V7 used noisy 5-class `aerial_merged`, so their numbers are **not comparable** to the
> V8+ 7-class benchmark and are excluded to keep the ablation controlled. A short summary of that
> exploratory phase — including two early failures that shaped later design choices — is in
> [`docs/early_experiments.md`](docs/early_experiments.md).

<details>
<summary><b>Failed experiments (honest record)</b> (click to expand)</summary>

<br>

| Version | Attempt | Result | Lesson |
|:-------:|:--------|:------:|:-------|
| V4.0 | Freeze + unfreeze 2-stage training | 46.82% | COCO pretrain ≠ aerial features; train from scratch |
| V7.0 | VisDrone noisy data | 55.25% (R=50%) | Data quality > architecture |
| V8.2 | Transfer-learning relay (V8.1 → v9) | 69.71% | Marginal overall (+0.12%), but records disagree on per-class behavior — relay lr likely too low for the new data |
| V12.1 | copy_paste + multi-scale fine-tune | 70.89% | copy_paste adds person/cycle false positives; −0.69% vs V12.0 |
| V13.0 | P2 high-res head | 66.68% | 8 GB OOM; forces batch=2 + imgsz=640, cancelling P2's benefit |

</details>

---

## ⚖️ Reference Benchmark

To give context for V16.0, we report our own VisDrone2019 result (V11.0) against a published
reference on the same public dataset.

| Model | Dataset | Classes | mAP@0.5 | Note |
|:------|:--------|:-------:|:-------:|:-----|
| **YOLO26s + aerial tuning (V11)** | VisDrone2019-DET official | 10 | **32.65%** | Our result, 6,471 train / 548 val |
| YOLOv8s-p2 (published reference) | VisDrone2019-DET | 10 | ~43.7% | External SOTA reference on the same benchmark |

**Reading this correctly.** This is a **secondary** comparison on a *public* dataset, not our
primary benchmark. Our V11 was trained with aerial-oriented hyperparameters (tuned for the
aerial_v9 class schema) and applied to VisDrone's 10-class setting without re-tuning, so the
~11 pp gap largely reflects domain mismatch rather than an architectural limit. The primary
result of this project is **74.00% mAP@0.5 on our own aerial_v9 dataset** (7 classes) — see
[Key Results](#-key-results).

> No other cross-model comparison is reported, because we have not trained YOLOv8/YOLO11
> baselines on aerial_v9 ourselves. We would rather report fewer numbers than publish
> ones we cannot reproduce.

---

## 🚀 Reproducibility

<details open>
<summary><b>⚡ Quick verification (no GPU, 50-image sample)</b></summary>

<br>

A 50-image CC BY 4.0 subset is included in [`data/val_samples/`](data/val_samples/).

```bash
# 1. Check dependencies
python -c "import torch, ultralytics; print('OK')"

# 2. Validate model configs
python -c "import yaml, os; [yaml.safe_load(open(f'configs/{c}')) for c in os.listdir('configs') if c.endswith('.yaml')]; print('All configs valid')"

# 3. Run inference on the sample set (needs best.pt from Releases)
python scripts/eval.py --weights best.pt --data data/val_samples/val_data.yaml --name sample_verify
```

</details>

<details>
<summary><b>🔧 Full training (V16, ~16.6 h on RTX 5060)</b></summary>

<br>

```bash
# Environment
conda create -n yolo_new python=3.11
conda activate yolo_new
pip install -r requirements.txt          # base ultralytics==8.4.12
# → then apply patches/ to add CoordAtt, WIoU v3, TAL 4px

# Prepare data — edit data/data.yaml to point at your aerial_v9 directory

# Train
python scripts/train_v16.py

# Evaluate
python scripts/eval.py --weights runs/detect/runs/aerial_train/yolo26s_v16/weights/best.pt \
                       --data data/data.yaml
```

</details>

> **Model weights** — download `best.pt` from
> [**GitHub Releases**](https://github.com/ZHY9981/aerial-yolo26s/releases) (~15 MB).

### Hardware Constraints

| Constraint | Effect | Solution |
|:-----------|:-------|:---------|
| RTX 5060 Laptop · 8 GB | Caps batch at 8; batch=4 at imgsz=800; blocks P2 head | P3+P4 dual-head architecture |
| Fixed (not purchasable) | Cannot relax via hardware | Every design evaluated by accuracy **and** memory cost |

---

## 💡 Key Findings

1. **Data > Architecture** — switching from noisy VisDrone to curated aerial_v9 gave **+12.56%**
   mAP@0.5, more than all architectural changes combined.
2. **Attention is cheap but effective** — CoordAtt adds negligible params but **+2.4%** Precision.
3. **Dual-head beats triple-head on 8 GB** — the controlled head-count comparison (V16 vs V20,
   both with CoordAtt at imgsz=800) gives **+0.71%** mAP. Counterintuitive but reproducible
   under the memory constraint.
4. **Assignment threshold matters for small objects** — TAL 4 px (vs default 8 px) directly boosted
   person/cycle by **+6–7%**.

---

## 🛣️ Future Work

| Direction | Motivation | Status |
|:----------|:-----------|:-------|
| **Onboard UAV inference** | Migrate the trained model to embedded hardware (Jetson-class) for real-time detection during flight. The memory-conscious architecture (7.03 M params, 6.8 GB train / ~1.5 GB inference) is designed with this path in mind. | Planned |
| **Car detection recovery** | Car AP lags at 67.3% due to the removed P5 head. Explore SimOTA center-prior and SGLoss-style adaptive grid selection to recover P2-level benefit without OOM. | Exploring |
| **TensorRT quantization** | Quantize for further latency reduction on lower-power edge devices. | Planned |
| **Rare-class generalization** | Freight / small-bus (< 2% of instances) need few-shot augmentation or soft-labeling; cross-dataset testing pending. | Open |

---

## 📂 Repository Structure

```
aerial-yolo26s/
├── configs/                          # Model architecture definitions
│   ├── yolo26s-v16-p34-coordatt.yaml #   ← best model (V16)
│   ├── yolo26s-v17-repncsp4.yaml
│   ├── yolo26s-v18-asff.yaml
│   └── yolo26s-v19-widep4.yaml
├── scripts/
│   ├── train_v16.py                  # Training entry point
│   └── eval.py                       # Evaluation with full metrics
├── patches/                          # Custom Ultralytics modifications
│   ├── README.md                     #   ← detailed modification log
│   └── coordatt.py                   #   ← CoordAtt module source
├── data/
│   ├── data.yaml                     # Dataset config (edit path for local use)
│   └── val_samples/                  # 50-image CC BY 4.0 verification subset
├── docs/
│   ├── ablation_table.md             # Complete V8-V20 version log
│   ├── early_experiments.md          # V1-V7 exploratory phase
│   ├── data_cleaning_report.md       # Dataset construction & class mapping
│   ├── technical_report.pdf          # 7-page technical report
│   └── technical_report.tex          # LaTeX source
├── results/                          # Plots and visualizations
└── requirements.txt
```

---

## 📖 Citation

```bibtex
@misc{aerial_yolo26s_2026,
  title        = {Resource-Constrained Aerial Small Object Detection with YOLO26s + CoordAtt},
  author       = {Zou, Haoyi},
  year         = {2026},
  url          = {https://github.com/ZHY9981/aerial-yolo26s}
}
```

## 📜 License

Released under the [MIT License](LICENSE).

<div align="center">
<sub>Built with PyTorch · Ultralytics · CUDA &nbsp;|&nbsp; Developed on RTX 5060 Laptop 8 GB</sub>
</div>
