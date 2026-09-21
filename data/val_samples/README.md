# Validation Sample Set — 50 Images

This directory contains 50 validation images and their YOLO-format labels.

The **images** come from **Roboflow aerial.v1i (CC BY 4.0)** — one of the public
sources that form the `aerial_v9` dataset. The **labels** have been remapped to the
project's 7-class schema (see `docs/data_cleaning_report.md` for the class mapping),
so they match what the trained model expects.

## Purpose

The full aerial_v9 dataset (~10 GB) cannot be open-sourced due to license terms
from multiple merged datasets. These 50 samples allow reviewers and collaborators to:

1. **Verify** the model actually works — run inference on real drone images
2. **Validate** class definitions and label format
3. **Reproduce** a small-scale detection result

## Quick Verification

```bash
# Run inference on the 50-image sample
python scripts/eval.py --weights best.pt --data data/val_samples/val_data.yaml --name sample_verify

# Expected output: mAP@0.5 ~65-75% on this small subset
# (Note: 50 images is not statistically significant; the full 2,224-image
#  validation set result is 74.00% mAP@0.5 as reported in README)
```