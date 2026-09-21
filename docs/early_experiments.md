<div align="center">

# Early Experiments (V1–V7)

**The exploratory phase before the benchmark was fixed**

*Six-class dataset · two failed approaches · one dataset pivot*

</div>

---

## Why this document exists

The [ablation study](ablation_table.md) begins at **V8.0**, where the dataset and class
schema were finally fixed (7 classes, `aerial_v9`). Versions **V1–V7** came before that —
V1–V6 used the 6-class `aerial` dataset, and V7 used the noisy 5-class `aerial_merged`. Their
mAP numbers are therefore **not comparable** to V8+. Rather than silently drop them, this
document summarizes what was learned.

---

## Timeline

All of V1–V6 trained on the **`aerial`** dataset — 6 classes (bicycle, bus, car, motorcycle,
person, truck), 2,985 images (Roboflow aerial.v1i, CC BY 4.0). V7 moved to **`aerial_merged`**
(5 classes, VisDrone-augmented, noisy labels).

| Version | Dataset | Classes | mAP@0.5 (best) | What it was |
|:-------:|:--------|:-------:|:--------------:|:------------|
| V1.0 | aerial | 6 | 49.26% | First working YOLO26s training run |
| V1.1 | aerial | 6 | 47.08% | Augmentation experiments (mixup found harmful) |
| V1.2 | aerial | 6 | 47.70% | Further augmentation tuning |
| V2.0 | aerial | 6 | 46.25% | Hyperparameter search |
| V3.0 | aerial | 6 | 43.82% | Hyperparameter search |
| V4.0 | aerial | 6 | 46.90% | Two-stage freeze/unfreeze — **failed** |
| V5.0 | aerial | 6 | 48.47% | Train-from-scratch baseline |
| V6.0 | aerial | 6 | 49.03% | Continued tuning → best of the 6-class runs |
| V7.0 | aerial_merged | 5 | 55.25% | VisDrone-augmented — **recall only 50%** |

> The dataset changed twice in this phase (`aerial` → `aerial_merged` → noise-cleaned
> `aerial_v8` at V8), and the class count changed twice (6 → 5 → 7). This is why the phase is
> documented separately: it is a record of *finding the problem*, not of *solving it*.
>
> **Pre-project baseline.** Before V1.0, the author's first aerial attempt was YOLOv8s on
> VisDrone (10 classes, 6,471 images), reaching **mAP@0.5 = 42.7%** after 140 epochs
> (recorded in V1.0's training log). V1.0 (YOLO26s, 6 classes, 2,985 images) reached 49.26%,
> a +6.6 pp improvement — though on a different dataset and class count, so again not a
> controlled comparison.

---

## Two failures that shaped later decisions

### V4.0 — Freeze/unfreeze two-stage training (46.90%)

The standard transfer-learning recipe: freeze the backbone, train the head, then unfreeze and
fine-tune at a low learning rate. It **underperformed simple from-scratch training** (V5.0
reached 48.47% on the same 6-class data).

**Lesson:** COCO-pretrained features encode object-shape priors (e.g. "person = upright
rectangle") that are largely orthogonal to the overhead view of aerial imagery, where people
appear as small blobs. Freezing the backbone *blocks* rather than helps domain adaptation.

### V1.1 — Mixup augmentation (47.08%, below the V1.0 baseline)

Mixup creates blended images and labels. On aerial data with dense, structured layouts
(buildings, roads, parking lots), the blends produce physically implausible combinations.

**Lesson:** mixup (and later copy-paste) were permanently disabled. This decision persists
through V20 and is documented in the [ablation config](ablation_table.md#training-configuration).

---

## The V7 → V8 pivot

V7.0 reached 55.25% mAP@0.5 — the highest of the whole early phase — but **recall was only
50.36%**: nearly half of all targets were being missed entirely. This was not a "needs more
tuning" problem; it was a basic-usability failure caused by noisy VisDrone label annotations
leaking into the training set.

Switching to a curated, noise-cleaned dataset (**V8.0**, 7 classes) raised mAP@0.5 to 67.81% —
a **+12.56%** jump that exceeded every architectural change combined.

**This is the single most important finding of the entire project:** *data quality beat
architecture.* V7 is the evidence for it. Excluding V7 from the main table would have hidden
the strongest result.

---

## How this fed into the main study

| Early observation | Where it shows up later |
|:------------------|:------------------------|
| V4 freeze/unfreeze fails | All later runs train from scratch with SGD lr₀=0.01 |
| V1.1 mixup harmful | `mixup=0.0, copy_paste=0.0` in every config |
| V7 recall collapse | The V7→V8 data-quality finding (Key Finding #1) |
| Class schema churn (6→5→7) | Benchmark frozen at 7 classes from V8 onward |

---

<div align="center">
<sub>Part of the <a href="../README.md">aerial-yolo26s</a> project · full results in <a href="ablation_table.md">ablation_table.md</a></sub>
</div>
