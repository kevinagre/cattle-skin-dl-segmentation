# Changelog

All notable changes to this project are documented in this file. The format is
based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and this
project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## v1.1.0 — 2026-04-22

Additive release adding pre-trained deep-learning model weights.

### Added

- **Pre-trained 2.5D U-Net models** attached as assets to the `v1.1.0`
  GitHub Release (Dragonfly Deep Learning Tool ZIP exports; ~78 MB each):
  - `HAIR_SegWiz_Standard_v1_<uuid>.zip` — hair-follicle segmentation.
  - `SG_SegWiz_Standard_v1_<uuid>.zip` — sweat-gland segmentation.
  Blood-vessel segmentation is performed manually; no DL model ships
  for blood vessels in this release.
- **README.md** — new "Trained models" section covering download,
  Deep Learning Tool **Import Zip** workflow, and training-envelope
  applicability notes.
- **protocols/03_AI_Segmentation_Wizard_Automation.md** — new
  "Quick Start — Using the Published Models" section at the top of the
  protocol for operators who only need to apply the models (no training).

### Notes

- The model ZIPs are distributed as GitHub Release assets rather than
  committed to the git tree to keep repository clones lean. The
  `release_assets/` local staging directory and bare `*.zip` files are
  git-ignored.
- Models were trained on bovine skin biopsies scanned on a Zeiss Xradia
  Versa system at 0.00784727 mm isotropic voxel size inside a 2 mm
  cylinder mask. Applying them outside this training envelope may
  require intensity normalization (`scripts/normalize_volume.py`) or
  local fine-tuning (Protocol 03 Part B).

## v1.0.0 — 2026-04-22

Initial public release of the bovine skin micro-CT segmentation pipeline.

### Added

- Four operator protocols:
  - `protocols/01_TXM_to_TIFF_Conversion.md` — raw reconstruction conversion.
  - `protocols/02_Manual_Segmentation_Dragonfly.md` — threshold-based manual
    segmentation of hair, sweat glands, and blood vessels in Dragonfly.
  - `protocols/03_AI_Segmentation_Wizard_Automation.md` — DL-assisted
    segmentation with the Dragonfly Segmentation Wizard (2.5D U-Net).
  - `protocols/04_Data_Management_and_Scripts.md` — master CSV schema,
    per-sample CSV format, and scripts reference.
- Four analysis scripts in `scripts/` with `argparse`-driven command-line
  interfaces and no hardcoded paths:
  - `populate_master_csv.py`
  - `audit_csv_placement.py`
  - `normalize_volume.py`
  - `rebuild_production_csv.py`
- Two TXM-to-TIFF conversion utilities in `conversion/`:
  - `convert_txm_to_tiff.py` — single-volume converter.
  - `batch_convert_all.py` — resumable batch converter.
- `tutorials/Dragonfly_Tutorials.md` — supplementary Dragonfly notes.
- Repository infrastructure: `LICENSE` (MIT for code, CC BY 4.0 for
  protocols / tutorials), `README.md`, `CITATION.cff`, `requirements.txt`,
  `.gitignore`, and `data/README.md`.
