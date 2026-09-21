<div align="center">

# Data Cleaning Report

**Building `aerial_v9` — a 7-class aerial detection dataset from public sources**

*Label-noise analysis and curation procedures*

</div>

---

## Navigation

[Pipeline](#pipeline) •
[Cleaning Criteria](#cleaning-criteria) •
[Case Studies](#case-studies) •
[Quantitative Impact](#quantitative-impact)

---

## Pipeline

```
aerial.v1            Roboflow · CC BY 4.0 · 6 classes · 2,985 images
       │
       ├── merge ────  Aerial Vehicle Detection  (MIT · ~5,000 images)
       │
       ├── merge ────  aerial.v3i                (MIT · additional images)
       │
       ├── clean ────  manual inspection of ~500 random samples
       │
       └── relabel ──  class consolidation + corrected boxes
                      │
                      ▼
              aerial_v9   ·  7 classes
              8,075 train / 2,224 val / 738 test
```

---

## Cleaning Criteria

Label quality was the single largest driver of performance in this project
(**+12.56%** mAP@0.5 — see [Quantitative Impact](#quantitative-impact)). Five rules were applied:

| # | Criterion | Rule |
|:-:|:----------|:-----|
| 1 | **Label noise removal** | Remove boxes that clearly misclassify the object |
| 2 | **Missing annotation** | Add boxes for visible objects that were unlabeled |
| 3 | **Occlusion threshold** | Remove objects occluded > 70% |
| 4 | **Edge truncation** | Keep objects truncated at image boundary if > 50% visible |
| 5 | **Class consolidation** | `motorcycle` + `bicycle` → `cycle` · `van` → `car` |

---

## Case Studies

Five representative examples of the label noise found and how each was resolved.

### 1 · Tricycle labeled as bicycle

| | |
|:--|:--|
| **Source** | VisDrone2019 |
| **Issue** | A tricycle with a canopy was labeled `bicycle`. |
| **Fix** | Re-labeled as `cycle` with a bounding box that fully captures the vehicle. |
| **Why it matters** | Canopied tricycles are a distinct vehicle type in Chinese urban scenes; labeling them `bicycle` causes false positives for the cycle detector. |

### 2 · Adjacent-frame inconsistency

| | |
|:--|:--|
| **Source** | aerial.v1 (Roboflow) |
| **Issue** | The same parked truck is `truck` in frame *N* but `car` in frame *N+1* (~0.5 s apart), despite near-identical box coordinates. |
| **Fix** | Re-labeled frame *N+1* to `truck` — the cargo bed is visible from the aerial angle. |

### 3 · Missed small objects

| | |
|:--|:--|
| **Source** | Aerial Vehicle Detection (MIT) |
| **Issue** | In a parking-lot scene, 3 of 8 visible cars were unlabeled (partially shadow-occluded). |
| **Fix** | Added boxes for all visible cars; shadow-occluded vehicles boxed over the visible portion only. |

### 4 · Manhole cover labeled as person

| | |
|:--|:--|
| **Source** | aerial.v1 (Roboflow) |
| **Issue** | A circular dark spot (~20 px) was labeled `person`. At low resolution, manhole covers and standing pedestrians have similar visual signatures. |
| **Fix** | Removed the false annotation — verified stationary and metallic across adjacent frames. |

### 5 · Articulated truck labeled as bus

| | |
|:--|:--|
| **Source** | aerial.v3i (MIT) |
| **Issue** | A long articulated truck + trailer was labeled `bus` (elongated rectangular shape resembles a bus from above). |
| **Fix** | Re-labeled `truck` — the visible gap between tractor and trailer distinguishes it from a bus. |

---

## Quantitative Impact

Approximate label-noise reduction by merge stage:

| Stage | Noise rate | Nature of errors |
|:------|:----------:|:-----------------|
| VisDrone merge | ~15% | Mostly class confusion (motorcycle↔bicycle, bus↔truck) |
| aerial.v1 → aerial.v8 | ~3% | Fewer errors, larger dataset |

**End-to-end effect on detection:**

```
V7 (uncleaned labels)    55.25%  mAP@0.5
V8 (cleaned labels)      67.81%  mAP@0.5
─────────────────────────────────────────
Δ                         +12.56%   ← exceeds all architectural changes combined
```

This confirms the project's central finding: **data quality beat architecture** by a wide margin.

---

<div align="center">
<sub>Part of the <a href="../README.md">aerial-yolo26s</a> project · see <a href="ablation_table.md">ablation_table.md</a> for full results</sub>
</div>
