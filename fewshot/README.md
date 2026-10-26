# Few-shot organoid segmentation for shape measurement

> **What is the cheapest way to get accurate organoid shape measurements from few or no annotations?**

This folder extends [MOrgAna](../README.md) (Gritti et al., *Development* 2021) with a benchmark study. MOrgAna showed that a small pixel classifier trained on a handful of masks can segment organoids well enough for quantitative morphology. Since then, pretrained vision models (DINOv2, Cellpose, Segment Anything / micro-sam) have changed what "a handful of masks" can buy. This study measures that, using the same downstream readouts MOrgAna was built for.

The study does not favour any method. The result is a cost/accuracy map: for a given annotation budget (0, 1, 3 or 10 masks) and compute budget, which method gives the smallest error on the shape measurements biologists actually report. If MOrgAna's classical pipeline is still the best choice at 1–3 masks, that answers the question as fully as any other outcome would.

## Why shape error, not just Dice

Dice and IoU are dominated by the organoid interior. A model whose masks are consistently a few pixels too tight or too loose, or that smooths away protrusions, can reach Dice > 0.95 and still bias the measurements:

- **Area**: a systematic boundary offset scales with perimeter, so small organoids are hit hardest.
- **Elongation / eccentricity** and **form factor** (perimeter² / 4π·area): very sensitive to edge smoothing and jagged boundaries.
- **Perimeter**: sensitive to boundary texture; also the main input to form factor.

Large pretrained models such as SAM are known to produce smooth, slightly offset boundaries, so pixel overlap alone would hide exactly the failure this study needs to catch. **Shape-measurement error is therefore a primary metric**, reported alongside the pixel metrics.

## Methods compared

| Method | Annotations used | Notes |
|---|---|---|
| MOrgAna, logistic regression | 1 / 3 / 10 | Original pipeline, ilastik-style features (`morgana/MLModel`) |
| MOrgAna, MLP | 1 / 3 / 10 | Same features, `deep=True` |
| DINOv2 features + linear/MLP head | 1 / 3 / 10 | Frozen backbone, patch features upsampled to pixels |
| U-Net | 1 / 3 / 10 | From scratch and with a pretrained encoder, heavy augmentation |
| Cellpose | 0 / 1 / 3 / 10 | Zero-shot generalist model, then fine-tuned |
| micro-sam | 0 / 1 / 3 / 10 | Zero-shot automatic instance segmentation, then fine-tuned |

"0" is zero-shot: no organoid masks from this dataset are used.

## Protocol

- **Annotation budgets:** k ∈ {0, 1, 3, 10} training masks.
- **Seeds:** each k-shot subset is drawn with several fixed random seeds (the same subsets for every method), so results show variance from *which* images were annotated, not only the mean.
- **Validation:** the validation split is used for early stopping and hyperparameters only. Each method has the same small tuning budget.
- **Test set:** held out and evaluated once per method/k/seed.
- **Cost accounting:** for each run, record the number of masks, the approximate annotation time, training time, inference time per image, and hardware (CPU vs GPU). "Cheapest" means cost in human time and compute, not only in number of masks.

## Metrics

Pixel / object level:
- Dice and IoU (per image)
- Boundary F1 and 95th-percentile Hausdorff distance (boundary quality)
- Instance detection precision/recall at IoU ≥ 0.5 (images contain several organoids)

Shape level (per matched organoid, computed the same way for prediction and ground truth, reusing `morgana/ImageTools/morphology/computemorphology.py` where possible):
- Area: relative error
- Perimeter: relative error
- Elongation (major/minor axis ratio) and eccentricity: absolute error
- Form factor: absolute error
- Bias (signed mean error) as well as spread, because a consistent bias matters differently from noise in downstream statistics

Headline plot: error on each shape metric against the annotation budget, one line per method, with seed variance shown.

## Data

`Dataset/` is local and not tracked by git (~7 GB). Current contents:

| Folder | Content |
|---|---|
| `OriginalData/` | train (2000 augmented, 52 pre-augmentation) / val (14) / test (28) brightfield images with masks, 750×750 |
| `MouseOrganoids/` | train (500 augmented, 11 pre-augmentation) / val (4) / test (4), 2592×1944 |
| `GemcitabineScreen/` | 42 BF + PI images with OrganoID outputs, no ground truth (qualitative / downstream check) |

k-shot subsets are drawn **only from the pre-augmentation images**. The augmented copies are derived from them, so sampling those would leak extra annotations into the budget.

These images contain several organoids each, whereas MOrgAna assumes one organoid per image. Instance handling (connected components and prediction ↔ ground-truth matching) is part of the evaluation harness, so every method is scored in the same way.

A benchmark on the original MOrgAna gastruloid data would be a natural second dataset if the Trivedi lab can share it.

## Planned layout

```
fewshot/
  README.md          this file
  data.py            dataset loading, mask conversion, k-shot split generation
  splits/            frozen k-shot subsets (JSON), one per k and seed
  metrics.py         pixel, boundary, instance and shape metrics
  methods/           one runner per method, common predict(image) -> instance mask API
  run.py             runs a method × k × seed grid, writes results to a table
  report/            figures and results tables
```

## Roadmap

Progress is tracked in the fork's [GitHub issues](https://github.com/alexisvannson/MOrgAna/issues). In order:

1. Data loading and frozen k-shot splits
2. Evaluation harness with shape metrics
3. Baselines: MOrgAna LR/MLP → DINOv2 → U-Net → Cellpose → micro-sam
4. Cost accounting and results report
5. Outputs: results or a fine-tuned organoid model contributed to micro-sam, and/or a small napari plugin; a run on the MOrgAna benchmark data

## Relation to MOrgAna

This is a follow-up to MOrgAna's own question: how little annotation is needed for quantitative organoid morphology. The original code is unchanged. The study imports it as one of the compared methods and reuses its morphology code for measurement. If you use this work, please cite the original paper:

> Gritti N., Lim J.L., Anlaş K., Oriola D., Pandya M., Aalderink G., Martinez Ara G., Trivedi V. MOrgAna: accessible quantitative analysis of organoids with machine learning. *Development* (2021) 148 (18): dev199611. https://doi.org/10.1242/dev.199611
