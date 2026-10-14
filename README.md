# Turbine-Blade Defect Detection – Master_Arbeit_Majd

Code used in the Master's thesis on YOLO-based defect detection on turbine blades trained on synthetic, 3D-rendered datasets.

## Pipeline order

1. `view_capture_render.py` – render images and YOLO labels from 3D models
2. `inject_cooling_holes.py` – add cooling-hole labels to the rendered labels
3. `remove_occluded_cooling_holes.py` – drop cooling-hole labels hidden by defect
4. `split_build_gui.py` – leakage-free train/val/test split and YOLO dataset zip
5. (training with Ultralytics YOLOv8 on the resulting `data.yaml`, not included)
6. `blade_inspection_app.py` – apply a trained model and the repair/scrap rules

## Files

### `view_capture_render.py`

Renders each `.obj` model from two fixed orthographic views (`Yneg`, `Ypos`) on a white background, at multiple scales (default: 10 log-spaced blade lengths from 82 mm to 2500 mm) and four lighting presets. The defect bounding box from an nTop BB file is projected into image space and written as a YOLO label.

* Output name: `<Type>_ID_<n>__<view>__<length>mm__<lighting>.png/.txt`
* Class ids come from `class_map.json`.
* Pre-flight report of obj-to-BB matches before rendering.
* Pause / cancel / resume via a job manifest (`render_job_<timestamp>.json`), progress bar with time left.
* Run: `python view_capture_render.py` (GUI), or CLI with `--obj`, `--bb`/`--bb-folder`, `--out`; resume with `--resume <manifest.json>`.
* Needs: `numpy`, `trimesh`, `pyrender`, `Pillow`, optional `tqdm`.

### `inject_cooling_holes.py`

Cooling holes are annotated manually only on 10 reference images (10 scales x 2 views). This script copies those boxes into every label file with the same (view, scale) key, independent of defect class, ID and lighting. Remaps the class index from the CVAT export (`0`) to the dataset index (`4`). Skips files that already contain cooling holes.

* Run: `python inject_cooling_holes.py --labels-dir <labels> --reference-dir <reference labels> [--dry-run]`

### `remove_occluded_cooling_holes.py`

Removes cooling-hole boxes (class `4`) that lie under an occluding defect box (default class `2`, erosion). A box is removed if its centre lies inside the occluding box or if at least the given fraction of its area overlaps it (default 0.3).

* Run: `python remove_occluded_cooling_holes.py --labels-dir <labels> [--occluding-classes 2 3] [--overlap-threshold 0.3] [--dry-run]`

### `split_build_gui.py`

GUI tool that creates the dataset split and builds the YOLO zip in one run.

* Split on model (stem) level, so images of the same defect never end up in different splits (no leakage).
* Stratified by defect type x size x relative height x relative chord position, exact global train/val/test ratios (largest-remainder allocation).
* Negative (defect-free) images split per image and per scale.
* Reports Kolmogorov-Smirnov distances train vs. val/test and writes ECDF, coverage and per-type figures, CSV/JSON assignments and a precheck report.
* Builds `dataset/images|labels/{train,val,test}` and `data.yaml` from `class_map.json`; aborts if label class ids do not match the map.
* Run: `python split_build_gui.py`
* Needs: `numpy`, `pandas`, `scipy`, `matplotlib`, `tkinter`, optional `openpyxl` (DoE file).

### `blade_inspection_app.py`

Desktop decision-support tool (German UI). Loads a blade image and a trained YOLO weights file, detects defects, optionally merges nearby boxes, and applies the rule-based repair/scrap logic to each defect. Shows an overall verdict, per-defect cards, and exports an annotated image with a result box and legend.

Decision rules, applied in this order:

1. No defect detected: the blade is intact.
2. Location: a defect that does not lie within the upper 10 % of the blade length is scrap. A defect reaching into this band counts as inside.
3. Area: a defect covering 30 % or more of the blade area is scrap.
4. Zone: the defect is assigned to Zone B (interior), Zone C (border along the top and both side edges) or both. A defect spanning both zones is checked against the stricter Zone B limit.
5. Width: the horizontal bounding-box width is compared with the limit of its zone. A wider defect is scrap, otherwise it is repair. Limits: Zone B 1/32 inch; Zone C 5/32 inch for stages 1 to 4 and 1/8 inch from stage 5, converted to mm.

The overall verdict is scrap if any defect is scrap, otherwise repair if any defect is repair, otherwise intact. The stage follows from the blade length (nearest of the 10 render sizes).

* Needs `rules_config.json` in the same folder (zone fractions, width limits, area limit, stage breakpoint).
* Without weights it runs in test mode with example detections.
* Run: `python blade_inspection_app.py`
* Needs: `numpy`, `scipy`, `Pillow`, `ultralytics`, optional `opencv-python`, `tkinter`.

## Not included

Datasets, rendered images, 3D models, nTop BB files, trained weights (`*.pt`) and `rules_config.json`.
