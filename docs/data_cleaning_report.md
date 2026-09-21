<div align="center">

# Dataset Construction

**Building `aerial_v9` — how the 7-class aerial dataset was assembled**

*From three public sources to a unified 7-class schema*

</div>

---

## Navigation

[Source Datasets](#source-datasets) •
[Class Mapping](#class-mapping) •
[Dataset Evolution](#dataset-evolution) •
[The VisDrone Problem](#the-visdrone-problem) •
[Honest Note on Records](#honest-note-on-records)

---

## Source Datasets

`aerial_v9` was assembled from four public sources:

| Source | License | Images | Classes | URL |
|:-------|:--------|:------:|:--------|:----|
| aerial.v1i | CC BY 4.0 | 2,985 | 6 | [Roboflow *krauseswelt/aerial-qjpyp*](https://universe.roboflow.com/krauseswelt/aerial-qjpyp) |
| Aerial Vehicle Detection | MIT | ~5,000 | 8 | [Roboflow *sovitopencvuniversity/aerial-vehicle-detection*](https://universe.roboflow.com/sovitopencvuniversity/aerial-vehicle-detection-0uc42) |
| aerial.v3i | MIT | ~5,300 | 1 (`person`) | [Roboflow *mukeshs-workspace-mytt6/aerial-lahmj*](https://universe.roboflow.com/mukeshs-workspace-mytt6/aerial-lahmj) |
| VisDrone2019 | CC BY-NC-SA | 6,471 / 548 | 10 | [VisDrone official](https://github.com/VisDrone/VisDrone-Dataset) |

An `aerial.v1i` class schema: `[bicycle, bus, car, motorcycle, person, truck]`.
The Aerial Vehicle Detection schema: `[PMT, articulated-bus, bus, car, freight, motorbike, small bus, truck]`.

---

## Class Mapping

The final dataset uses **7 classes**: `person, cycle, bus, small-bus, car, truck, freight`.
The mapping from source classes:

| Final class | Sources |
|:------------|:--------|
| `person` | aerial.v1i `person` + VisDrone `pedestrian` + aerial.v3i `person` |
| `cycle` | aerial.v1i `bicycle` + `motorcycle` + VisDrone `person_on_bicycle` |
| `bus` | aerial.v1i `bus` + AVD `bus` + AVD `articulated-bus` |
| `small-bus` | AVD `small bus` |
| `car` | aerial.v1i `car` + AVD `car` |
| `truck` | aerial.v1i `truck` + AVD `truck` |
| `freight` | AVD `freight` |

> The `cycle = bicycle + motorcycle + person_on_bicycle` consolidation is documented
> explicitly in the V7.0 training log. The remaining mappings follow directly from the
> source class names above. AVD's `PMT` (public motor transport) is not used in the final
> schema.

---

## Dataset Evolution

The dataset went through four stages as the class schema and label quality were refined:

| Stage | Dataset | Train | Classes | Used by |
|:------|:--------|:-----:|:-------:|:--------|
| 1 | aerial.v1i | 2,985 | 6 | V1–V6 |
| 2 | aerial_merged | 7,148 | 5 | V7 |
| 3 | aerial_v8 | 3,800 | 7 | V8–V9 |
| 4 | **aerial_v9** | **8,075** | **7** | V10–V20 |

**Stage 1 → 2** merged aerial.v1i with VisDrone (5,058 images filtered to 5 relevant classes),
creating `aerial_merged`. This added data but also imported VisDrone's noisy labels.

**Stage 2 → 3** discarded the VisDrone-merged data and switched to cleaner sources, establishing
the final 7-class schema.

**Stage 3 → 4** expanded to 8,075 images, largely by adding ~5,300 `person` images from
aerial.v3i (MIT-licensed, low-altitude scenes) — which is exactly what fixed the person-class
weakness that had persisted through V8.

---

## The VisDrone Problem

`aerial_merged` (V7) reached **55.25% mAP@0.5** but only **50.36% recall** — nearly half of all
targets were being missed. The cause was label noise from the VisDrone merge: dense overhead
scenes with many small, crowded targets produced inconsistent and often-missing annotations.

Switching to the cleaned dataset (V8.0) raised mAP@0.5 to **67.81%** — a **+12.56%** jump with
no architectural change. This is the project's central finding: **data quality beat architecture.**

---

## Honest Note on Records

This document describes the dataset at the level supported by the preserved training logs
(V7, V8, V10, and the version overview). Those logs record the *sources*, the *class mapping*,
and the *aggregate effect* of the data change.

A per-image cleaning log — specific annotations corrected, images removed, edge cases handled —
was **not systematically preserved** during the project. Rather than reconstruct plausible-sounding
examples after the fact, this document reports only what the surviving records actually support.

---

<div align="center">
<sub>Part of the <a href="../README.md">aerial-yolo26s</a> project · see <a href="ablation_table.md">ablation_table.md</a> f