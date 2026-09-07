# SEZ Boundary Segmentation

A machine learning pipeline that predicts Special Economic Zone (SEZ) boundary polygons in India from satellite imagery, using deep learning semantic segmentation.

## Background

This project is a spinoff of RA work at DevLab (Prof. Dean Yang) mapping SEZ boundary polygons by hand in QGIS/Google My Maps. Each of 265 mapped SEZs is a hand-drawn polygon tagged with a confidence level (`High`, `Medium`, `Low`) reflecting how certain the mapping is. Manually mapping the remaining un-mapped SEZs is slow, so this project trains a model to estimate a polygon automatically from satellite features (imagery, roads, building footprints), using the already-mapped polygons as ground truth.

## Approach

- **Task**: semantic segmentation — the model predicts a per-pixel inside/outside-SEZ mask, which is then converted back into a polygon.
- **Imagery**: Sentinel-2 (10m/pixel) via Google Earth Engine — RGB + Near-Infrared bands.
- **Extra input channels**: roads and building footprints, stacked alongside imagery.
- **Tiling**: one fixed 192×192 px tile per polygon (1920m × 1920m at 10m/pixel), centered on the polygon's centroid. Tile size is implemented as a single named parameter so it can be swept later (see Phase 8.5 below).
- **Architecture**: U-Net with a ResNet34 encoder (ImageNet-pretrained), via `segmentation-models-pytorch`.
- **Loss**: Dice + BCE combined.
- **Optimizer / schedule**: Adam with `ReduceLROnPlateau`, early stopping on validation metric.
- **Split**: train/validation on High + Medium confidence polygons; test (touched once, at the end) on Low confidence polygons.
- **Evaluation**: IoU, Hausdorff distance, and percent area error against the true polygon, benchmarked against a classical CV baseline (Canny/Hough edge detection).
- **Compute**: Google Colab (free-tier GPU) for training.

## Project status

Early stage — Phase 1 (label generation) of an 8-phase pipeline. See `label-generation.ipynb`.

- Done: polygon layer loads from `data/SEZ POLYGONS.gpkg`; reprojected to EPSG:7755 (India-specific Lambert Conformal Conic) for meter-based measurements; polygon areas computed.
- Open decision: rows with an invalid/missing `Confidence` value are currently being overwritten to `"Medium"` — needs a deliberate call (exclude vs. relabel) before proceeding, since silently defaulting unknowns into the training pool risks mislabeled examples.
- Not yet started: bounding-box stats against the current `.gpkg` (to confirm which polygons get clipped at 192px), centroid computation, mask rasterization, mask saving, and a visual spot-check.

## Pipeline phases

1. **Label generation** — polygons → training masks (in progress)
2. **Imagery pull** — Sentinel-2 + roads/buildings from Google Earth Engine
3. **Dataset assembly** — manifest, train/val/test split, PyTorch `Dataset`/`DataLoader`
4. **Model architecture** — U-Net + ResNet34 via `segmentation-models-pytorch`
5. **Training loop** — Dice+BCE loss, Adam, `ReduceLROnPlateau`, early stopping
6. **Classical CV baseline** — Canny/Hough, for comparison
7. **Mask → polygon conversion** — threshold, extract, simplify, reproject
8. **Evaluation** — IoU, Hausdorff distance, percent area error; final test-set run
   - **8.5** Tile-size sensitivity analysis — repeat the pipeline at 2-3 alternate tile sizes bracketing 192px, purely for reporting

## Repository structure

```
SEZ/
├── label-generation.ipynb           # Phase 1: polygons -> training masks
├── data/
│   ├── SEZ POLYGONS.gpkg            # master polygon layer (single source of truth)
│   ├── SEZ_POLIGONS_ALL.kmz         # original KMZ export, kept as raw reference
│   ├── project.qgz                  # QGIS project
│   └── sez_tile_dimension_analysis.html  # tile-size coverage/imbalance analysis
└── papers/                          # background literature (boundary delineation, segmentation)
```

`data/` is intentionally tracked in git — the files are small.

## Environment

- Windows, Python virtual environment (`.venv`)
- Key packages: `geopandas`, `shapely`, `rasterio`, `torch`, `torchvision`, `torchgeo`, `segmentation-models-pytorch`
- Google Earth Engine account registered (noncommercial project approved)

## Data notes

- `SEZ POLYGONS.gpkg` is the only working copy of attributes + geometry (the older `AttributesTable.xlsx` has been retired to avoid sync issues).
- Confidence values are restricted to `High` / `Medium` / `Low`.
