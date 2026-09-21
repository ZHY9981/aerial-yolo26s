# Patches to Ultralytics 8.4.12 for Aerial Small-Object Detection

This directory documents every custom modification made to the base Ultralytics 8.4.12 codebase.
These patches enable:

1. **Coordinate Attention (CoordAtt)** in the detection neck
2. **WIoU v3 loss** for small-object gradient enhancement
3. **TAL 4 px threshold** in TaskAlignedAssigner for tiny targets
4. **Class-weighted loss** for long-tail class imbalance

## Patch Index

| File | Modification | Purpose |
|:---|:---|:---|
| `coordatt.py` | New module: CoordAtt coordinate attention | Inject 2D positional encoding into channel attention |
| `loss.py` | Class weights in v8DetectionLoss | Address person/car/cycle imbalance |
| `loss.py` | WIoU v3 replacement for CIoU | Box regression loss with adaptive attention |
| `tal.py` | TAL positive-sample threshold: 8 px → 4 px | Improve small-object assignment |

## 1. CoordAtt (Coordinate Attention)

**Paper:** Hou et al., "Coordinate Attention for Efficient Mobile Network Design," CVPR 2021.

**Where:** Injected at P3 and P4 neck positions, immediately before the Detect head.

**Why here and not backbone:** Per-scale placement avoids cross-FPN degradation.
A CoordAtt at P5 backbone would require two nearest-neighbor upsample steps to reach
P3 detection features, losing spatial precision. Per-scale CoordAtt at P3 and P4
directly enhances the 120×120 and 60×60 feature maps respectively.

**V16 architecture (best model):**
- P3 CoordAtt[32] — enhances 120×120 features for person/cycle/small-bus
- P4 CoordAtt[32] — enhances 60×60 features for car/truck/bus
- No CoordAtt on P5 backbone (removed — dual-head has no P5 detector)

**Code:** See `coordatt.py` in this directory.

---

## 2. WIoU v3 Loss

**Paper:** Tong et al., "Wise-IoU: Bounding Box Regression Loss with Dynamic Focusing Mechanism," arXiv 2023.

**Change:** Replaced the default CIoU loss in `ultralytics/utils/loss.py` with WIoU v3.

**Why:** Standard CIoU penalizes large and small objects equally in box regression.
Small aerial targets (person < 30 px, cycle < 20 px) get disproportionately
weak box regression gradients. WIoU v3 uses an attention-based outlier suppression
mechanism that adaptively focuses on small targets without penalizing large ones.

**Impact:** +1.78% mAP@0.5 (V8.0 → V8.1), zero VRAM cost.

**Modification location:** `ultralytics/utils/loss.py`, in the `bbox_iou` call within
`BboxLoss.forward()`. Replace the default `iou=CIoU` with `iou=WIoUv3`.

---

## 3. TAL Small-GT Threshold (8 px → 4 px)

**Paper:** Feng et al., "TOOD: Task-aligned One-stage Object Detection," ICCV 2021 (original TAL).

**Change:** In `ultralytics/utils/tal.py`, inside `TaskAlignedAssigner.select_candidates_in_gts()`,
the small-box mask threshold was halved:

```python
# before (default)
wh_mask = gt_bboxes_xywh[..., 2:] < self.stride[0]          # < 8 px for P3
# after (V14 onward)
wh_mask = gt_bboxes_xywh[..., 2:] < (self.stride[0] / 2)    # < 4 px for P3
```

**Why:** Ground-truth boxes smaller than the smallest stride may contain **no anchor center**,
so they receive no positive assignment and contribute no localization gradient. The assigner
handles this by expanding such boxes to `stride_val` so at least one anchor lands inside. The
default cutoff (8 px) only caught boxes smaller than 8 px; halving it to 4 px extends this
rescue to boxes in the **4–8 px** range — exactly where the smallest aerial targets live.

**Impact:** Person +6%, cycle +7% (V8 → V10 comparison, holding other factors constant).

**Location:** `ultralytics/utils/tal.py`, `select_candidates_in_gts()` — the `wh_mask` line.

> **Note:** this is a threshold on the *ground-truth box size*, not on the distance between an
> anchor and a GT center. The goal is identical — give tiny targets positive anchors — but the
> mechanism operates on GT boxes, not anchor-center distance.

---

## 4. Class-Weighted Loss

**Why:** The aerial_v9 dataset has extreme class imbalance. Direct counts from the training
labels (8,075 images):

```
person    69,061      car       68,280
cycle     14,191      truck     13,278
bus        6,106      freight    1,180
small-bus     987
```

Standard equal-weight training causes the model to optimize for person/car and under-serve the
rare classes.

**Applied weights (`ultralytics/utils/loss.py`, `v8DetectionLoss`):**

```python
cls_weight = torch.tensor([2.0, 3.0, 1.8, 1.5, 1.0, 1.0, 1.0])
```

The vector is indexed by the class order
