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
<tr><td align="center">V8.2</td><td>Relay V8.1 → aerial_v9 (larger data)</td><td align="center">69.71%</td><td align="center">48.17%</td><td align="center">—</td><td align="center">Marginal (+0.12%) *</td></tr>
<tr><td align="center">V9.0</td><td>+ CoordAtt (3-head)</td><td align="center">68.66%</td><td align="center">47.44%</td><td align="center">—</td><td align="center">Archived</td></tr>
<tr><td align="center">V10.0</td><td>+ CoordAtt + WIoU + aerial_v9 (8,075 img)</td><td align="center">70.51%</td><td align="center">49.33%</td><td align="center">—</td><td align="center">Milestone</td></tr>
<tr><td align="center">V11.0</td><td>VisDrone2019 benchmark (10-class)</td><td align="center">32.65%</td><td align="center">18.33%</td><td align="center">—</td><td align="center">Comparison</td></tr>
<tr><td align="center">V12.0</td><td>imgsz 640 → 800 <i>(batch 8→4 forced by VRAM)</i></td><td align="center">71.58%</td><td align="center">50.27%</td><td align="center">—</td><td align="center">+1.07% §</td></tr>
<tr><td align="center">V12.1</td><td>+ copy_paste=0.3 / multi-scale fine-tune</td><td align="center">70.89%</td><td align="center">49.29%</td><td align="center">—</td><td align="center">❌ Failed (−0.69%)</td></tr>
<tr><td align="center">V12.2</td><td>Continue V12.0 (cls 0.5 → 0.7)</td><td align="center">72.10%</td><td align="center">50.43%</td><td align="center">—</td><td align="center">+0.52%</td></tr>
<tr><td align="center">V13.0</td><td>P2 high-res head</td><td align="center">66.68%</td><td align="center">46.55%</td><td align="center">—</td><td align="center">❌ OOM</td></tr>
<tr><td align="center">V14.0</td><td>P3+P4 dual-head — <i>5 changes bundled</i> †</td><td align="center">73.71%</td><td align="center"><b>52.65%</b></td><td align="center">6.82M</td><td align="center">Peak mAP₅₀₋₉₅ (+1.61%)</td></tr>
<tr><td align="center">V14.1</td><td>car weight 1.0 → 1.3 (fine-tune)</td><td align="center">73.69%</td><td align="center">52.62%</td><td align="center">6.82M</td><td align="center">Neutral (−0.02%)</td></tr>
<tr><td align="center">V15.0</td><td>BiFPN + compressed P5 128ch</td><td align="center">73.37%</td><td align="center">52.09%</td><td align="center">7.44M</td><td align="center">Regression (−0.34%)</td></tr>
<tr bgcolor="#fffbe6"><td align="center"><b>V16.0</b></td><td><b>P3+P4 + per-scale CoordAtt, imgsz=800</b></td><td align="center"><b>74.00%</b></td><td align="center">52.11%</td><td align="center"><b>7.03M</b></td><td align="center"><b>🏆 Best</b></td></tr>
<tr><td align="center">V17.0</td><td>+ P4 RepNCSPELAN4</td><td align="center">73.97%</td><td align="center">51.56%</td><td align="center">—</td><td align="center">Neutral (−0.03%)</td></tr>
<tr><td align="center">V18.0</td><td>+ ECA / ASFF experiments</td><td align="center">73.46%</td><td align="center">51.16%</td><td align="center">—</td><td align="center">Neutral (−0.51%)</td></tr>
<tr><td align="center">V19.0</td><td>P4 wide channel 256 → 384</td><td align="center">73.29%</td><td align="center">50.89%</td><td align="center">7.82M</td><td align="center">Neutral (−0.17%) ‡</td></tr>
<tr><td align="center">V20.0</td><td>P3+P4+P5 triple-head (clean)</td><td align="center">73.29%</td><td align="center">50.41%</td><td align="center">8.06M</td><td align="center">Neutral (−0.71% vs V16)</td></tr>
</tbody>
</table>

<sub>† V14's jump bundles five simultaneous changes (head removal, TAL threshold, class weighting,
augmentation scale, imgsz 800→960) — not a single-variable ablation. The isolated head-count
effect is measured by V16 vs V20: <b>+0.71%</b>.<br>
‡ V19 is the only version that improved car detection (67.6% → 68.9%, +1.3%), but at the cost of
every other class (person −2.0%, cycle −1.5%, truck −1.2%). Net effect was negative — widening P4
globally is not the answer; car needs a targeted fix.<br>
§ V12's +1.07% is not a pure imgsz ablation: raising imgsz to 800 forced batch from 8 to 4, so the
two changed together. The batch reduction is a direct consequence of the resolution increase under
the 8 GB limit, not an independent choice.<br>
* V8.2 relayed V8.1's weights onto the larger aerial_v9 set (V8.0–V8.1 used aerial_v8), so it is
also a dataset change, not a pure training-schedule ablation. The two source records disagree:
the CSV shows a marginal +0.12% overall, while the version overview reports all classes declining.
It was not adopted for production either way.<br>
V1–V7 used earlier, differently-scoped datasets (V1–V6: 6-class <code>aerial</code>; V7: noisy
5-class <code>aerial_merged</code>) and are not comparable to the V8+ benchmark — see
<a href="early_experiments.md">early_experiments.md</a>. "Neutral" = within noise of the
preceding best; "Regression" = measurably below the preceding best. Neither was adopted.</sub>

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
| V8.2 | Transfer-learning relay (V8.1 → v9) | 69.71% | Marginal overall; records disagree on per-class behavior — relay lr likely too low |
| V12.1 | copy_paste + multi-scale fine-tune | 70.89% | copy_paste adds person/cycle false positives |
| V13.0 | P2 high-res head | 66.68% | 8 GB OOM; forces batch=2 + imgsz=640, cancelling P2's benefit |

---

## Training Configuration

Identical across all versions unless noted.

| Parameter | Value | Notes |
|:----------|:------|:------|
| GPU | RTX 5060 Laptop 8 GB | Fixed hardware constraint |
| Batch size | 8 (V8–V11) · 4 (V12+, P2 experiments: 2) | VRAM limit; V12 dropped to 4 because imgsz=800 required it |
| Optimizer | SGD · lr₀=0.01 · momentum=0.937 | Stable for aerial features |
| Epochs | 200 (V8–V16) · 160 (V17) · 120 (V18–V20) | V17+ use explicit epoch count, not early stop |
| Image size | 640–960 | 800 is the sweet spot |
| Augmentation | mosaic=0.5 · scale=0.3 · no mixup/copy-paste | Aerial-specific: small targets need careful augmentation |
| Loss | WIoU v3 (box) · VFL (cls) · DFL | WIoU v3 from V8.1 onward |
| Workers | 4 | Data prefetch |

---

## Key Insights

1. **Data > Architecture** — switching from noisy VisDrone to curated aerial_v9 gave **+12.56%**
   mAP@0.5, more than all architectural changes combined.
2. **Dual-head beats triple-head on 8 GB** — the controlled comparison (V16 vs V20, both with
   CoordAtt at imgsz=800; only head count differs) gives **+0.71%** mAP. Note that the V12→V14
   pair shows a larger +2.1% gap, but that comparison changed five variables at once (head count,
   TAL, class weighting, augmentation scale, imgsz) — we report the isolated effect as +0.71%.
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
