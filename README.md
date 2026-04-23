# Cattle Skin Micro-CT Segmentation Pipeline

A reusable pipeline for segmenting hair follicles, sweat glands, and blood
vessels in bovine skin biopsies scanned by micro-computed tomography. The
pipeline combines threshold-based manual segmentation with 2.5D U-Net deep
learning models inside [Dragonfly](https://dragonfly.comet.tech/) and produces
per-structure CSV exports that are consolidated into a single master
measurements spreadsheet.

The repository distributes the operator protocols, conversion utilities,
analysis scripts, and Dragonfly tutorial notes needed to reproduce the
workflow on your own data. No experimental dataset is shipped; see
[`data/README.md`](data/README.md).

## Preview

A short volumetric fly-through of a DL-segmented sweat-gland network in a
bovine skin biopsy (sample 22-10), rendered in Dragonfly:

![Sweat-gland network fly-through (DL-segmented, Dragonfly 3D render)](docs/media/sweatgland_preview.gif)

## Gallery

Selected Dragonfly renders from the production cohort. Each image is a 3D
volumetric rendering of structures segmented by the pipeline.

### Hair-follicle segmentation

The `HAIR_SegWiz_Standard_v1` model returns one Multi-ROI per follicle
(random colour per instance).

<p align="center">
  <img src="docs/media/wizard_76_22-09_dl_v9_multiroi_side.png" width="49%" alt="Hair follicles segmented by HAIR_SegWiz_Standard_v1, angled side view (sample 22-09)">
  <img src="docs/media/23-102_hair_multi_roi_final.png" width="49%" alt="Hair follicles segmented by HAIR_SegWiz_Standard_v1 (sample 23-102)">
</p>

### Sweat-gland segmentation

The `SG_SegWiz_Standard_v1` model returns one Multi-ROI per coiled gland
(random colour per instance).

<p align="center">
  <img src="docs/media/22-17_sg_3d.png" width="70%" alt="Sweat glands segmented by SG_SegWiz_Standard_v1 (sample 22-17)">
</p>

### Full-sample composites (hair + sweat gland + blood vessels)

Combined output of the two DL models plus manual blood-vessel segmentation,
rendered inside the 2 mm cylinder mask that defines the region of interest.
Hair follicles sit in the dense cluster at the top of each biopsy; blood
vessels are the large branching trees; sweat-gland coils occupy the space
between them.

Sample 23-120 — all three structures shown overlaid on the μCT tissue
cylinder (left) and isolated (right); hair follicles (gray, on top),
sweat-gland coil (multi-colored), blood-vessel tree (purple, branching):

<p align="center">
  <img src="docs/media/23-120_core_all_structures.png" width="49%" alt="Sample 23-120: all segmented structures overlaid on the micro-CT tissue cylinder">
  <img src="docs/media/23-120_hair_sg_bv_composite.png" width="49%" alt="Sample 23-120: hair, sweat gland, and blood vessels isolated from the tissue">
</p>

Additional examples across different biopsy diameters, with the 2 mm cylinder
mask visualised as a blue grid:

<p align="center">
  <img src="docs/media/23-114_hair_sg_bv_fullscene_pd6.00.png" width="32%" alt="Sample 23-114: full-scene composite within the cylinder mask (6 mm biopsy)">
  <img src="docs/media/23-108_bv_sg_overview.png" width="32%" alt="Sample 23-108: overview of sweat glands and a large branching blood vessel">
  <img src="docs/media/23-110_bv_sg_context.png" width="32%" alt="Sample 23-110: sweat glands on top with an orange blood-vessel tree inside the cylinder mask">
</p>

### Notable morphology

A single elongated "snake-like" sweat-gland coil captured in isolation
(sample 22-07, left) and a top-down view of a shallow biopsy where the
blood-vessel tree dominates the segmentation plane alongside the sweat
glands (sample 25-53, right):

<p align="center">
  <img src="docs/media/22-07_sg_snake_gland_closeup.png" width="49%" alt="Sample 22-07: close-up of a single elongated 'snake-like' sweat-gland coil">
  <img src="docs/media/25-53_bv_sg_topdown.png" width="49%" alt="Sample 25-53: top-down view of blood-vessel tree and sweat glands in a shallow biopsy">
</p>

## Quick start

1. **Clone the repository** and create a Python environment (Python ≥ 3.10).

    ```bash
    git clone https://github.com/csantosvet/cattle-skin-dl-segmentation.git
    cd cattle-skin-dl-segmentation
    python -m venv .venv
    .venv\Scripts\activate           # Windows
    # source .venv/bin/activate       # macOS / Linux
    pip install -r requirements.txt
    ```

2. **Convert raw reconstructions to TIFF** using the conversion utilities
   (see [`protocols/01_TXM_to_TIFF_Conversion.md`](protocols/01_TXM_to_TIFF_Conversion.md)):

    ```bash
    python conversion/convert_txm_to_tiff.py \
        --input  <path-to-recon.txm> \
        --output <path-to-output.tif>
    ```

    For a directory tree of reconstructions, use the resumable batch runner:

    ```bash
    python conversion/batch_convert_all.py \
        --source-root <raw-scan-root> \
        --output-root <tiff-stack-root>
    ```

3. **Segment each sample in Dragonfly**, following either:

    - [`protocols/02_Manual_Segmentation_Dragonfly.md`](protocols/02_Manual_Segmentation_Dragonfly.md) — threshold-based manual segmentation, or
    - [`protocols/03_AI_Segmentation_Wizard_Automation.md`](protocols/03_AI_Segmentation_Wizard_Automation.md) — DL-assisted segmentation (recommended once training samples are available).

    Export per-structure CSVs from Dragonfly into `03_RESULTS/Hair/`,
    `03_RESULTS/SweatGlands/`, and `03_RESULTS/BloodVessels/` as described
    in [`protocols/04_Data_Management_and_Scripts.md`](protocols/04_Data_Management_and_Scripts.md).

4. **Consolidate per-sample CSVs** into a master measurements file:

    ```bash
    python scripts/populate_master_csv.py \
        --results-dir <project-root>/03_RESULTS \
        --master-csv  <project-root>/master_measurements.csv
    ```

5. **(Optional) Audit CSV placement** to catch mis-filed exports, and
   **normalize** intensity distributions between scanning sessions:

    ```bash
    python scripts/audit_csv_placement.py --results-dir <project-root>/03_RESULTS
    python scripts/normalize_volume.py \
        --source    <path-to-source.tif> \
        --reference <path-to-reference.tif> \
        --output    <path-to-normalized.tif>
    ```

## Repository structure

```text
cattle-skin-dl-segmentation/
├── LICENSE                       # MIT (code) + CC BY 4.0 (protocols & tutorials)
├── README.md
├── CITATION.cff                  # How to cite this repository
├── CHANGELOG.md
├── requirements.txt              # Pinned Python dependencies
├── .gitignore
├── protocols/
│   ├── 01_TXM_to_TIFF_Conversion.md
│   ├── 02_Manual_Segmentation_Dragonfly.md
│   ├── 03_AI_Segmentation_Wizard_Automation.md
│   ├── 04_Data_Management_and_Scripts.md
│   └── images/                   # Operator-training screenshots
├── scripts/
│   ├── populate_master_csv.py    # Consolidate per-structure CSVs
│   ├── audit_csv_placement.py    # QC for mis-filed CSV exports
│   ├── normalize_volume.py       # Intensity normalization between sessions
│   └── rebuild_production_csv.py # Parse a Markdown production log into CSVs
├── conversion/
│   ├── convert_txm_to_tiff.py    # Single-volume TXM → TIFF converter
│   └── batch_convert_all.py      # Safe, resumable batch converter
├── tutorials/
│   └── Dragonfly_Tutorials.md
└── data/
    └── README.md                 # Data policy and project layout
```

## Trained models

Pre-trained 2.5D U-Net models for hair and sweat-gland segmentation are
distributed as Dragonfly Deep Learning Tool exports (ZIP archives) and
attached to the **`v1.1.0` GitHub Release**. Download them from the
[Releases page](https://github.com/csantosvet/cattle-skin-dl-segmentation/releases/tag/v1.1.0).

| Asset                                                        | Segments     | Size   |
| ------------------------------------------------------------ | ------------ | ------ |
| `HAIR_SegWiz_Standard_v1_<uuid>.zip`                         | Hair follicles | ~78 MB |
| `SG_SegWiz_Standard_v1_<uuid>.zip`                           | Sweat glands   | ~78 MB |

Blood-vessel segmentation is performed manually; no DL model ships for
blood vessels in this release.

### Importing a model ZIP into Dragonfly

The ZIPs are Dragonfly **Deep Learning Tool** exports. Import them once per
workstation; after import, the models are also available inside the
**Segmentation Wizard** without any further action.

1. Open Dragonfly 2024.1 or later.
2. From the menu bar, choose **Artificial Intelligence → Deep Learning Tool**.
3. In the **Model Overview** panel, click **Import Zip** and select the
   downloaded `.zip` file. Repeat for the second model.
4. The imported models now appear in the Model Overview list with their
   type (semantic segmentation) and class count. You can preview them from
   the Deep Learning Tool directly, or use them from the Segmentation
   Wizard via **Start with Model** as described in
   [`protocols/03_AI_Segmentation_Wizard_Automation.md`](protocols/03_AI_Segmentation_Wizard_Automation.md).

### Applicability and expected behavior

These models were trained on bovine skin biopsies scanned on a Zeiss Xradia
Versa system at **0.00784727 mm isotropic voxel size** inside a **2 mm
cylinder mask**. When applied to new samples:

- Expected to work best on samples whose gray-value distribution overlaps
  the training envelope. Out-of-distribution samples (very different
  intensity range) may fail completely; see Protocol 03 Part E for the
  failure modes and the `scripts/normalize_volume.py` calibration option.
- The operator is expected to review every prediction and run Connected
  Components cleanup (Protocol 02) before measurement.
- For cohorts that differ substantially in scanner settings, sample
  preparation, or species, fine-tuning on 2–3 locally segmented samples is
  recommended (see Protocol 03 Part B).

## Software requirements

- **Dragonfly** 2024.1 or later (external; required for all segmentation
  steps and for training/running the DL models). Dragonfly itself is not
  distributed with this repository.
- **Python** ≥ 3.10 with the packages pinned in [`requirements.txt`](requirements.txt).
- **Operating system**: all scripts are platform-agnostic; protocol examples
  are written for Windows paths but work unchanged on macOS and Linux when
  the path arguments are adjusted.

## Data

This repository distributes code, protocols, and operator-training images only.
No experimental dataset is shipped; see [`data/README.md`](data/README.md) for
the expected project layout and how to apply the pipeline to your own data.

## Citation

If this pipeline is useful in your work, please cite this repository using the
metadata in [`CITATION.cff`](CITATION.cff).

## License

Code is released under the **MIT License**; protocols and tutorials under
**CC BY 4.0**. See [`LICENSE`](LICENSE) for the full text.
