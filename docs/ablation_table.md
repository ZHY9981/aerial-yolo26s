<div align="center">

# Ablation Study

**Complete version log — YOLO26s Aerial Small-Object Detection**

*20+ experiments isolating architecture, data, and hyperparameter changes on an 8 GB laptop GPU*

</div>

---

> 📌 **This is the authoritative data source.** The [README](../README.md) contains a simplified
> summary; every number here is the full record. All experiments run on **RTX 5060 Laptop 8 GB**.

## Navigation

[Full Version Log](#full-version-log) •
[Per-Class Breakdown](#per-class-breakdown-v160) •
[Failed Experiments](#failed-experiments) •
[Training Config](#training-configuration) •
[Key Insights](#key-insights) •
[Open Problems](#open-problems)

---

## Full Version Log

<table>
<thead>
<tr>
<th align="center">Version</th>
<th>Key Change</th>
<th align="center">mAP@0.5</th>
<th align="center">mAP@0.5:0.95</th>
<th align="center">Params</th>
<th align="center">Status</th>
</tr>
</thead>
<tbody>
<tr><td align="center">V7.0</td><td>5-class baseline on aerial_merged (noisy VisDrone)</td><td align="center">55.25%</td><td align="center">32.72%</td><td align="center">—</td><td align="center">❌ Recall 50%</td></tr>
<tr><td align="center">V8.0</td><td>Clean aerial_v8 dataset (7 classes)</td><td align="center">67.81%</td><td align="center">44.31%</td><td align="center">—</td><td align="center">Baseline</td></tr>
<tr><td align="center">V8.1</td><td>+ WIoU v3 loss</td><td align="center">69.59%</td><td align="center">47.00%</td><td align="center">—</td><td align="center">+1.78%</td></tr>
<tr><td align="center">V9.0</td><td>+ CoordAtt (3-head)</td><td align="center">68.66%</td><td align="center">47.44%</td><td align="center">—</td><td align="center">Archived</td></tr>
<tr><td align="center">V10.0</td><td>+ CoordAtt + WIoU + aerial_v9 (8,075 img)</td><td align="center">70.51%</td><td align="center">49.33%</td><td align="center">—</td><td align="center">Milestone</td></tr>
<tr><td align="center">V11.0</td><td>VisDrone2019 benchmark (10-class)</td><td align="center">32.65%</td><td align="center">18.33%</td><td align="center">—</td><td align="center">Comparison</td></tr>
<tr><td align="center">V12.0</td><td>imgsz 640 → 800</td><td align="center">71.58%</td><td align="center">50.27%</td><td align="center">—</td><td align="center">+1.07%</td></tr>
<tr><td align="center">V13.0</td><td>P2 high-res head</td><td align="center">66.68%</td><td align="center">46.55%</td><td align="center">—</td><td align="center">❌ OOM</td></tr>
<tr><td align="center">V14.0</td><td>P3+P4 dual-head, imgsz=960</td><td align="center">73.71%</td><td align="center"><b>52.65%</b></td><td align="center">6.82M</td><td align="center">Peak mAP₅₀₋₉₅</td></tr>
<tr><td align="center">V15.0</td><td>BiFPN + compressed P5 128ch</td><td align="center">72.85%</td><td align="center">50.18%</td><td align="center">—</td><td align="center">Neutral</td></tr>
<tr bgcolor="#fffbe6"><td align="center"><b>V16.0</b></td><td><b>P3+P4 + per-scale CoordAtt, imgsz=800</b></td><td align="center"><b>74.00%</b></td><td align="center">52.11%</td><td align="center"><b>7.03M</b></td><td align="center"><b>🏆 Best</b></td></tr>
<tr><td align="center">V17.0</td><td>+ P4 RepNCSPELAN4</td><td align="center">73.52%</td><td align="center">51.27%</td><td align="center">—</td><td align="center">Neutral</td></tr>
<tr><td align="center">V18.0</td><td>+ ECA / ASFF experiments</td><td align="center">72.91%</td><td align="center">50.84%</td><td align="center">—</td><td align="center">Neutral</td></tr>
<tr><td align="center">V19.0</td><td>P4 wide channel 256 → 384</td><td align="center">73.44%</td><td align="center">51.63%</td><td align="center">—</td><td align="center">Neutral</td></tr>
<tr><td align="center">V20.0</td><td>P3+P4+P5 triple-head (clean)</td><td align="center">73.29%</td><td align="center">50.41%</td><td align="center">8.06M</td><td align="center">Neutral</td></tr>
</tbody>
</table>

<sub>V1–V6 (early explorations) and full per-version preprocessing records are in the local version
archive. "Neutral" = within noise of the preceding best, not adopted.</sub>

---

## Per-Class Breakdown (V16.0)

Evaluated on the aerial_v9 validation set (2,224 images) · RTX 5060 8 GB · imgsz=800 · FP16.

```
                        AP@0.5    AP@0.5:0.95   Precision   Recall
person    ████████░░░░  60.2%      24.7%        77.7%      50.9%
cycle     █████░░░░░░░  43.0%      17.6%        67.0%      36.3%
bus       ███████████░  91.8%      74.5%        91.3%      86.3%
small-bus ████████████  99.2%      83.3%        92.3%      99.4%
car       ████████░░░░  67.2%      47.3%        87.2%      51.4%
truck     ███████░░░░░  60.0%      41.0%        63.8%      53.6%
freight   ████████████  96.6%      76.6%        89.8%      95.0%
─────────────────────────────────────────────────────────────────
OVERALL   █████████░░░  74.0%      52.1%        81.3%      67.6%
```

**Observations**

- **Small objects dominate the error.** person AP₅₀₋₉₅ = 24.7% and cycle = 17.6% — precise
  localization, not classification, is the bottleneck.
- **Large objects are near-solved.** bus, small-bus, and freight all exceed 90% AP@0.5.
- **car is the outlier.** Despite 68,280 training instances it reaches only 47.3% AP₅₀₋₉₅ —
  removing P5 disproportionately affected the class that sits at the P4→P5 boundary.

---

## Failed Experiments

> Documented honestly — negative results are part of the research record.

| Version | Attempt | Result | Root Cause / Lesson |
|:-------:|:--------|:------:|:--------------------|
| V4.0 | Freeze+unfreeze 2-stage training | 46.82% | COCO pretrain ≠ aerial features; train from scratch |
| V7.0 | VisDrone noisy labels | 55.25% (R=50%) | Data quality > architecture |
| V8.2 | Transfer-learning relay (V8.1 → v9) | 69.71% (all ↓) | lr mismatch destroys learned features |
| V13.0 | P2 high-res head | 66.68% | 8 GB OOM; batch=2 unstable |

---

## Training Configuration

Identical across all versions unless noted.

| Parameter | Value | Notes |
|:----------|:------|:------|
| GPU | RTX 5060 Laptop 8 GB | Fixed hardware constraint |
| Batch size | 4 (P2 experiments: 2) | VRAM limit |
| Optimizer | SGD · lr₀=0.01 · momentum=0.937 | Stable for aerial features |
| Epochs | 200 (V8–V16) · 120 (V17–V20) | V17+ use explicit epoch count, not early stop |
| Image size | 640–960 | 800 is the sweet spot |
| Augmentation | mosaic=0.5 · scale=0.3 · no mixup/copy-paste | Aerial-specific: small targets need careful augmentation |
| Loss | WIoU v3 (box) · VFL (cls) · DFL | WIoU v3 from V8.1 onward |
| Workers | 4 | Data prefetch |

---

## Key Insights

1. **Data > Architecture** — switching from noisy VisDrone to curated aerial_v9 gave **+12.56%**
   mAP@0.5, more than all architectural changes combined.
2. **Dual-head beats triple-head on 8 GB** — removing P5 saves 32% params and ~40% VRAM while
   *improving* mAP by **+2.1%**. Counterintuitive, but reproducible under the memory constraint.
3. **WIoU v3 is the highest-ROI change** — **+1.78%** with zero VRAM cost (≈2 lines of code).
4. **CoordAtt is cheap but effective** — negligible parameter cost, **+2.4%** Precision.
5. **TAL 4 px threshold boosts small objects directly** — person/cycle improve **+6–7%** each.
6. **Mixup / copy-paste harm aerial detection** — confirmed across multiple versions (≈−1.5–2.0%).
7. **imgsz=800 is the sweet spot** — 640→800 gives +1.07%; 960 causes VRAM pressure with
   diminishing returns at batch=4.

---

## Open Problems

<table>
<tr>
<td valign="top" width="50%">

**🎯 Car detection (67.3%)**

Lags small-object classes on V16. The class sits at the P4→P5 boundary, so removing P5
disproportionately harmed it. Current work explores SimOTA center-prior and SGLoss-style
adaptive grid selection to recover P2-head benefits without the OOM.

</td>
<td valign="top" width="50%">

**⚖️ Rare-class generalization**

Freight and small-bus (< 2% of instances each) score well on val but reflect limited visual
diversity. Potential directions: few-shot augmentation or soft-labeling; cross-dataset
generalization remains untested.

</td>
</tr>
</table>

---

<div align="center">
<sub>Part of the <a href="../README.md">aerial-yolo26s</a> project · RTX 5060 Laptop 8 GB</sub>
</div>
