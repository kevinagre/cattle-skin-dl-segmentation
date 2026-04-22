# Protocol 03: AI Segmentation Wizard — Automation

## Purpose

This protocol describes how to use Dragonfly's **AI Segmentation Wizard** to semi-automate hair, sweat gland, and blood vessel segmentation after you have built a small library of manually segmented samples using Protocol 02. The Wizard trains a deep learning model on those manual labels and then applies the model to new samples so the operator only corrects and measures instead of segmenting from scratch.

---

## Quick Start — Using the Published Models

Pre-trained 2.5D U-Net models for **hair** (`HAIR_SegWiz_Standard_v1`) and
**sweat glands** (`SG_SegWiz_Standard_v1`) are distributed as Dragonfly
**Deep Learning Tool** exports (ZIP archives) attached to the
[`v1.1.0` GitHub Release](https://github.com/csantosvet/cattle-skin-dl-segmentation/releases/tag/v1.1.0).
If you only need to apply the models to new samples — not retrain them —
start here.

### Importing the ZIPs

1. Download `HAIR_SegWiz_Standard_v1_<uuid>.zip` and
   `SG_SegWiz_Standard_v1_<uuid>.zip` from the Release page.
2. Open Dragonfly 2024.1 or later.
3. From the menu bar, choose **Artificial Intelligence → Deep Learning Tool**.
4. In the **Model Overview** panel, click **Import Zip** and select a
   downloaded `.zip` file. Repeat for the second model.
5. Both models now appear in the Model Overview list as semantic
   segmentation models. Close the Deep Learning Tool.

Imported models are stored in Dragonfly's Deep Trainer folder (typically
`%LocalAppData%\ORS\Dragonfly2025.1\Deep Learning\...`) and are
automatically available inside the Segmentation Wizard after import.

### Applying a published model to a new sample

1. Follow Protocol 02 Part A to import the TIFF reconstruction, correct
   voxel spacing, align the sample, and apply the 2 mm cylinder mask.
2. Right-click the cylinder-masked volume in **Data Properties and Settings**
   → **Segmentation Wizard**.
3. In **Model Selection**, pick the published Hair or SG model → **Start
   with Model**.
4. Click **Predict** to run inference on the full sample.
5. Review the prediction in the main workspace, clean it up as described in
   Protocol 02 (Process Islands, Connected Components, manual erasures as
   needed), then measure and export the Multi-ROIs.
6. Repeat for the other structure. Segment blood vessels manually
   (Protocol 02 Part D); no DL model ships for blood vessels in this release.

### Training-envelope notes

The published models were trained on bovine skin biopsies scanned on a
Zeiss Xradia Versa system at **0.00784727 mm isotropic voxel size** inside
a **2 mm cylinder mask**. When you apply them to new samples:

- Samples whose gray-value distribution falls inside the training envelope
  should produce clean predictions with little manual cleanup.
- Samples whose gray-value lower bound falls far above the training
  distribution can trigger complete model failure (out-of-distribution).
  Use `scripts/normalize_volume.py` to match the new volume's
  tissue-intensity distribution to a reference, or fall back to manual
  segmentation for that sample (see Part E of this protocol).
- For cohorts that differ substantially in scanner settings, sample
  preparation, or species, fine-tune on 2–3 locally segmented samples
  (Part B) rather than relying on the published models as-is.

The remainder of this protocol covers **training your own models from
scratch** (or fine-tuning the published ones). If you only need to apply
the shipped models, the steps above are sufficient.

---

## How It Works (In a Nutshell)

1. **Training data**
   Your manual segmentations are the labels. For each training sample you have: one **image volume** (the cylinder-masked recon) and the **clean Multi-ROIs** (Hair, SweatGlands, BloodVessels). The Wizard uses these pairs to learn "this image pattern → this segmentation."

2. **Training**
   You open the **Segmentation Wizard**, point it at your training samples (image + ROIs), define classes (e.g. Hair vs Background), and run training. Train **one model per structure** (Hair, Sweat Glands, Blood Vessels). Training uses the GPU and can take from minutes to an hour depending on data size and hardware.

3. **Application**
   For a **new** sample: import the TIFF, apply the same cylinder mask, run the **trained model** to get predicted ROIs, then do a short manual review, run Connected Components and measurements, and export. Most of the heavy segmentation work is done by the model.

4. **Iteration**
   As you process more samples, add newly corrected segmentations to the training set and retrain so the model improves over time.

**No extra step is required during manual segmentation.** You follow Protocol 02, save the session (with clean Multi-ROIs and consistent names). Those saved sessions are what you later load and feed into the Wizard as training data.

---

## Recommended Training Set Diversity

Aim to cover in the training set:

- Both (or all) experimental groups you intend to segment — if your cohort has, for example, two breed groups with different morphological profiles, the model must see samples from both.
- Samples spanning the realistic range of image intensity distributions across your scanning sessions. Scanner drift between acquisition sessions can shift the gray-value histogram substantially; diverse training samples plus intensity calibration (section A.6) are the main defenses.
- Samples that represent both typical and atypical morphology (e.g. low and high structure counts; samples with and without segmentable blood vessels).

**Key observations that affect model training:**

- **Gray-value ranges differ between scanning sessions.** Hair thresholds can shift substantially between sets acquired under different scanner conditions. The model learns from raw voxel patterns rather than threshold values, so this should not prevent a single unified model from working — but it is why diverse training data matters. **Caveat:** samples whose gray-value lower bound sits far above the training distribution can trigger complete model failure (out-of-distribution / OOD) and have to be re-segmented manually or normalized first; see E.4.
- **Morphological differences between groups.** If your experimental groups differ in hair density or sweat-gland size/count, include samples from both groups in training.
- **Blood vessel variability.** Samples can legitimately have 0, 1, or 2 BVs. The BV model must learn both "vessel present" and "vessel absent" patterns. Include BV-absent samples as negative examples (label = empty ROI or background-only).

---

## Scientific Rationale: Why This Training Strategy?

This section documents the evidence supporting the training approach — a single unified model, ~7 diverse samples with 6–8 frames each, iterative fine-tuning in diversity-first order — drawing from Dragonfly's official documentation and peer-reviewed literature on U-Net segmentation.

### What Dragonfly's Documentation Recommends

The Dragonfly Segmentation Wizard documentation (Comet Technologies, 2024.1) is deliberately non-prescriptive about exact frame counts. Key guidance:

- Training can begin with **"at least one frame"** — the tool is designed for iterative refinement, not one-shot training.
- **Sparse labeling** is fully supported (since version 2024.1). After the first training cycle, the Wizard supports filling frames from the model's own prediction, editing errors, and retraining — exactly the Predict → Correct → Retrain loop we use (B.8).
- The docs emphasize a **"strategy for continuous improvement"** of training data and recommend **data augmentation** when data is limited (we have augmentation enabled at 10×).
- For universal models, they recommend **calibrated datasets** — achieved in this workflow through consistent preprocessing (fixed voxel spacing, a fixed cylinder mask, and Process Islands up to 50 voxels) **and Intensity Scale Calibration** (Air=0, Hair=10,000 — see section A.6).
- The default 2.5D U-Net strategy uses **3 slices, depth 5, 64 initial filters** — our exact configuration.

**Source**: [Dragonfly Help — Training Models with the Segmentation Wizard](http://www.theobjects.com/dragonfly/dfhelp/2024-1/Content/Artificial%20Intelligence/Segmentation%20Wizard/Training%20Models%20with%20the%20Segmentation%20Wizard.htm); [Workflow and Best Practices](http://www.theobjects.com/dragonfly/dfhelp/2024-1/Content/Artificial%20Intelligence/Deep%20Learning%20Tool/Workflow%20and%20Best%20Practices.htm).

### What the Academic Literature Says About Dataset Size

| Study | Finding | Relevance to Our Setup |
|-------|---------|----------------------|
| **Lee et al. (2025)**, PLOS ONE — "How much data is needed for segmentation in biomedical imaging?" | Residual U-Nets on 2D polyp segmentation (1,000 images) and 3D brain tumor segmentation (258 volumes). Performance **plateaued at ~80% of available data**; gains beyond that were marginal. "Collecting a moderate amount of high-quality data is often sufficient for developing clinically viable DL models." | A small diverse training set can reach most of the achievable performance; diminishing returns are expected past that core set. |
| **Kim et al. (2026)**, Annals of Biomedical Engineering — Transfer learning for mice lung segmentation in synchrotron X-ray tomographic microscopy | U-Net pretrained on lung sections, fine-tuned on whole lung images using **only 5 labeled samples** (+ 2 for testing). Achieved DSC = 0.89, mIOU = 0.89. | Closest analogue: micro-CT imaging, U-Net, limited annotations, transfer learning / iterative fine-tuning. Validates that 5–8 samples with fine-tuning is a proven strategy. |
| **Laprade et al. (2021)**, MICCAI — "How few annotations are needed for segmentation using a multi-planar U-Net?" | Multi-planar U-Net achieved **similar performance with half the annotations** by doubling the number of automatically generated training planes (data augmentation). | With 10× data augmentation the effective training set is much larger than the raw 6–8 frames per sample; more frames beyond 8 per sample have diminishing returns. |
| **Springer (2023)** — Effect of dataset size on CNN segmentation for CT/MR renal tumors | CT kidney segmentation reached performance plateau with **54–125 images**; MR required 122–389 images. | Binary hair/background segmentation is simpler than multi-organ segmentation, requiring less total data; standardized preprocessing further reduces the variability the model must handle. |
| **Multi-stage fine-tuning (2025)**, MICCAI — Pancreatic tumor segmentation | Cascaded pre-training across multiple datasets with **increasing specificity** is a proven strategy for limited-data scenarios. | Supports the diversity-first training order below: start with one group/session, progressively add different gray-value ranges, groups, and morphologies. |

### Why a Single Unified Model (Not Per-Session)

- Per-session models would have only a handful of training samples each — right at the minimum viable threshold based on the literature above.
- A unified model with samples drawn from several scanning sessions and both experimental groups develops **invariance** — the ability to recognize the target structure regardless of the specific gray-value distribution. This is the same principle behind how ImageNet pretraining works: diversity in training data teaches the model to focus on structural patterns rather than absolute intensity values.
- Gray-value differences between sessions are handled through **data augmentation** (intensity shifts, contrast changes), **intensity calibration** (section A.6), and the model's learned feature hierarchy — convolutional filters detect edges, textures, and shapes, not raw pixel values.
- If a unified model underperforms on a specific session after evaluation, session-specific models remain a fallback option.

### Why ~7 Primary Training Samples Are Sufficient

A training plan built around ~7 primary samples, evaluated against held-out samples before continuing with any further rounds, is supported by:

1. **Task simplicity**: Binary segmentation (hair vs. background) is among the simplest segmentation tasks. Multi-class organ segmentation requires far more data.
2. **Standardized preprocessing**: When all samples share the same voxel spacing, cylinder mask, and cleanup pipeline, inter-sample variability that the model must account for is reduced.
3. **Iterative fine-tuning**: Each round builds on the cumulative knowledge of all previous rounds. By round 7, the model has seen roughly 40–55 annotated frames across several intensity distributions and groups.
4. **Literature precedent**: transfer learning with 5 labeled micro-CT samples has achieved DSC ≈ 0.89 (Kim et al., 2026); 7 diverse samples sit comfortably above that floor.
5. **Diversity coverage**: the seven-sample core should span multiple scanning sessions, both experimental groups, both sparse and dense structure counts, and both BV-present and BV-absent samples.

Plan the sample order so that each new round introduces a diversity axis the model has not yet seen (a new session, a new group, a new morphological extreme). Reserve at least one sample per group as a held-out evaluation set, never used for training, to gate the decision of whether additional rounds are justified.

**The critical evaluation checkpoint is after round 7.** If validation metrics are acceptable on all held-out samples, additional rounds provide diminishing returns and can be skipped or deferred. If validation is poor for one group, prioritize adding more samples from that group.

### Why 6–8 Frames Per Sample (Not More)

- Dragonfly's Wizard is designed for **a handful of representative frames**, not exhaustive slice-by-slice annotation. Attempting to load a whole stack of labeled slices (hundreds of frames) can crash the Wizard.
- With 10× data augmentation, each frame generates ~10 augmented training patches (flipped, rotated, scaled), so 6–8 frames effectively produce **60–80 training patches per sample**.
- The multi-planar U-Net study (Laprade et al., 2021) showed that doubling augmentation compensates for halving annotations — so 6 frames plus augmentation is roughly equivalent to ~12 frames without augmentation.
- **Negative frames** (1–2 dermis-only frames per sample, labeled entirely as Background) are the highest-value addition for reducing false positives. They teach the model explicitly what is NOT the structure of interest, which matters more than adding a 9th or 10th positive frame.

### Why Iterative Fine-Tuning (Not Retraining From Scratch)

When the model is loaded via **"Start with Model"** and trained on a new sample, it **continues training from its existing weights** (fine-tuning). It does not erase prior knowledge. This is crucial because:

1. **Knowledge retention**: the model retains hair follicle patterns learned from all previous samples while adapting to the new sample's characteristics.
2. **Faster convergence**: fine-tuning starts from a strong baseline, so each successive round converges in fewer epochs than the initial training run.
3. **Invariance learning**: exposure to multiple gray-value ranges forces the model to learn structural features (edges, shapes, spatial relationships) rather than memorizing specific intensity thresholds — the same principle behind multi-stage fine-tuning in the pancreatic tumor segmentation literature (MICCAI 2025).
4. **Diminishing catastrophic forgetting**: with data augmentation and diverse samples, the model generalizes across distributions rather than overwriting old knowledge with new.

### Training Log Template

Keep a per-round training log with at least the following columns so that later rounds can be audited and compared:

| Round | Sample | Group | Frames (Pos+Neg) | Steps/Epoch | Stopped at Epoch | Training Time | Best val_loss | Model Version |

**Also record for each round:**

- The hair gray-value lower bound on that sample (or the SG gray-value band for SG models), to document the breadth of intensity coverage.
- The BG:Hair (or BG:SG) voxel ratio in the labeled frames; very large imbalances (e.g. >40:1) tend to produce noisier training curves.
- Notable events (loss spikes, early stopping, LR reductions, frame re-selection).
- A one-line per-model verdict on each hold-out sample (count Δ, volume Δ, mean Feret Δ, mean EqSphD Δ) and whether the verdict passes the acceptance rules in section D.

### Validation — DL vs. Manual Ground Truth

After each round, predict on the reserved hold-out samples and record the deltas between DL-derived and manual measurements:

| Round | Sample | Count (M→DL) | Count Δ | Volume Δ | Mean Feret Δ | Mean EqSphD Δ | Verdict |

Use the acceptance rules in Part D to produce the verdict (GOOD / ACCEPTABLE / NEEDS IMPROVEMENT / POOR). Trends to watch:

- **Training time** typically decreases round-over-round as the model starts from stronger weights.
- **BG:Hair ratio** in the labeled frames tends to correlate with training stability — work toward balanced crowded frames rather than adding more background-only frames.
- **Seesaw effect**: a single fine-tuned model can improve on one group while regressing on the other. When that occurs, either (a) adopt a two-model strategy (one per group), or (b) switch to multi-dataset training (see section B.12) which generally resolves the seesaw in one shot.

### When to Add More Data

| Scenario | Action |
|----------|--------|
| Validation acceptable on both held-outs after round 7 | Proceed to **Phase 2** — manually segment 2 samples from any previously unrepresented scanning session or group and fine-tune. |
| Poor on one group but good on the other | Add 1–2 reserve samples from the under-performing group **before** moving on to a new session. |
| New scanning session ready for production | Manually segment 2 samples from that session (pick from the start and end of the batch for scanner-drift diversity), fine-tune, then test on a 3rd unseen sample from the same session to confirm generalization. |
| Borderline validation after 2 new-session samples | Add a 3rd sample from that session and re-fine-tune. |
| Persistent false positives in dermis after 7 samples | Increase negative (Background-only) frames to 2–3 per sample. |
| Model degrades after adding a new sample | Roll back to the last checkpoint and try a different frame selection. |
| Recency bias toward one group | Add one reserve sample from the opposite group to rebalance. |

---

## Prerequisites

- **Protocol 02** (Manual Segmentation) completed for **at least 10 samples** with high-quality segmentations.
- Each session saved (**File > Save Session**) and containing:
  - The **image volume** (the cylinder-masked recon copy — e.g. `<sample_id>_recon (Copied)`).
  - The **clean Multi-ROIs** used for analysis: `Hair`, `SweatGlands`, `BloodVessels`. These are the ROIs you ran Connected Components on, cleaned, and exported to CSV — **not** the initial uncleaned ROIs (`H`, `SG`).
- **Consistent ROI naming** across all training samples (same names on every project). Prefer: `Hair`, `SweatGlands`, `BloodVessels`.
- **Same preprocessing** for all samples:
  - Voxel spacing: 0.00784727 mm (isotropic).
  - Cylinder mask: 2 mm diameter (1 mm radius).
  - Process Islands: up to 50 voxels, 26-connected (applied before Connected Components).
  - **Intensity Scale Calibration**: Air=0, Hair=10,000 (see section A.6). Mandatory for SG models; recommended for all structures.
- If two people segment: sessions from both PCs can be combined for training; ensure cross-validation (Protocol 00, Section 2.7) and consistent naming.

---

## Part A: Prepare Training Data

### A.1 Decide Which Samples to Use

**Recommended: train a single unified model using a diverse subset of your manually segmented samples.** Do not train one model per scanning session.

- A single model trained on diverse data (different gray-value distributions, different experimental groups, different structure sizes) generalizes better than narrow per-session models.
- Per-session models would have only a few training samples each — right at the minimum viable amount. A unified model with roughly 10–15 samples is substantially stronger.
- If the unified model struggles with a specific session, you can fall back to session-specific models later.

**Suggested split of your samples by role:**

| Role | Count (typical) | Purpose |
| ---- | --------------- | ------- |
| **Training — Phase 1** | ~7 | Diversity-first core: span the intensity distributions and experimental groups in your cohort. |
| **Training — Phase 2** | 2–3 | Samples from any previously unrepresented scanning session or group, added after the Phase 1 validation checkpoint. |
| **Validation (held out)** | ≥ 2 | One sample per experimental group, never used for training. Used to decide whether additional training rounds are justified. |
| **Reserve** | variable | Samples available for targeted retraining rounds if validation reveals gaps (e.g. a group that still under-segments). |

### A.2 What the Wizard Needs per Sample

For each training sample the Wizard needs:

| Item | Description |
|------|-------------|
| **Image volume** | The 3D volume the model will see as input. Use the **cylinder-masked** recon — this is typically the original-named volume (e.g. `<SampleID>_recon`) after the mask was applied to it, while the `(Copied)` version is the backup of the unmasked original. Check which volume has the cylinder applied by inspecting slices. |
| **Label ROIs** | The **clean Multi-ROIs** only: `Hair`, `SweatGlands`, `BloodVessels`. Do **not** use the initial uncleaned ROIs (`H`, `SG`). |

You do **not** need to create a separate "label volume" or export labels to another format. The ROIs in the Dragonfly session are the labels.

**For samples with no BVs:** include them in Hair and SG model training. For the BV model, these samples serve as **negative examples** — the model learns that not every sample has blood vessels. If the Wizard requires a label ROI, create an empty BV ROI for these samples.

### A.3 Naming and Consistency

- Use the **same ROI names** on every training project: `Hair`, `SweatGlands`, `BloodVessels`.
- If your sessions use different names (e.g. `H (Multi-ROI)`, `SG (Multi_ROI)`, `BV_1`), **rename them** before training so every session is consistent.
- Do **not** mix naming across sessions unless the Wizard allows explicit class mapping.

### A.4 Pre-Training Checklist

Before starting the Wizard, verify each training session:

- [ ] Session loads without errors (relocate TIFF paths if needed).
- [ ] Image volume is the **cylinder-masked** recon (not the original).
- [ ] ROIs are named consistently (`Hair`, `SweatGlands`, `BloodVessels`).
- [ ] ROIs are the **clean Multi-ROIs** (post-Connected Components, post-cleanup).
- [ ] Voxel spacing matches the acquisition (same in X, Y, Z) and is identical across all training samples.
- [ ] **Intensity Scale Calibration** applied (see A.6 below). **Mandatory for SG models; recommended for all structures.**

### A.5 Collecting Sessions for Training (Multi-Operator)

If two operators both segment:

1. Copy the session folders from the secondary machine to the training machine (e.g. into `<project_root>/02_DRAGONFLY_PROJECTS/`).
2. On the training machine, open Dragonfly, then for each training sample: **File > Load Session**, and if prompted relocate the TIFF to the local path.
3. Ensure both machines used the same ROI names and the same preprocessing (cylinder, voxel spacing).

### A.6 Intensity Scale Calibration (mandatory for SG, recommended for all)

**Why**: Micro-CT volumes from different scan sessions have different absolute gray-value ranges due to tube condition, reconstruction parameters, and scan settings. A given tissue may appear at ~12,000–18,000 in one sample and at ~8,000–14,000 in another. Without calibration, a model trained on one distribution may produce zero detections on another — even if the tissue looks identical to a human.

Intensity Scale Calibration maps all volumes to the same reference scale using anatomical landmarks that are present in every sample: **air** (lowest density, always ~0) and **hair** (highest density tissue). After calibration, air = 0 and hair = 10,000 in every volume, regardless of original scan settings.

**When to calibrate**: Before any Segmentation Wizard session. Every volume that enters training or prediction must be calibrated to the same scale.

**Discovery context**: during SG model development, sequential fine-tuning on uncalibrated volumes produced a model that yielded 0 SG voxels on new samples whose intensity distribution was shifted relative to the training set (catastrophic forgetting compounded by intensity mismatch). Switching to multi-dataset training combined with explicit calibration resolved both issues.

#### Step-by-step procedure

1. **Load the cylinder-masked volume** into a Dragonfly scene (the same volume you will use for training or prediction).

2. **Open the calibration tool**: Right-click the volume in the Data Properties and Settings panel → **Calibrate Intensity Scale**.

3. **Define landmarks** — two reference points are needed:

   | Landmark | What to select | Calibrated value |
   | -------- | -------------- | ---------------- |
   | **Air**  | A region of pure air/resin outside the tissue cylinder (the black background area). Click on a clearly empty region. | **0** |
   | **Hair** | A region of dense hair tissue (the brightest/densest structures in the volume). Click on a clearly visible hair follicle cross-section. Hair was chosen over dermis because it is denser and provides a more distinct, repeatable landmark. | **10,000** |

4. **Click "Calibrate"**. The tool linearly rescales the entire volume so that:
   - Air voxels → 0
   - Hair voxels → 10,000
   - All other tissue scales proportionally between these anchors.

5. **Verify**: After calibration, the volume's display range should reflect the new scale. Air regions should read ~0, hair cross-sections should read ~10,000.

6. **Repeat** for every volume in the scene before launching the Segmentation Wizard.

#### Important notes

- **Calibration changes intensity values, not geometry.** The volume dimensions (voxel count, spacing) are unchanged. Multi-ROI spatial masks from uncalibrated sessions remain valid because they index by voxel position, not intensity.
- **Calibrate before training AND before prediction.** If you train on calibrated volumes, the model expects calibrated input at prediction time. An uncalibrated volume at prediction time will produce poor or zero results.
- **Consistency**: Always use the same landmarks (Air=0, Hair=10,000) for every volume. Do not switch to different tissue references across samples.
- **The calibration is applied in-place** to the loaded volume. If you want to preserve the original uncalibrated data, keep the original TIFF on disk (calibration doesn't modify the source file on disk unless you explicitly export/overwrite).

#### Reference

- Dragonfly documentation: [Intensity Scale Calibration](http://www.theobjects.com/dragonfly/dfhelp/2024-1/Content/Post-Processing%20Images/Intensity%20Scale%20Calibration.htm)
- Dragonfly documentation: [Workflow and Best Practices](http://www.theobjects.com/dragonfly/dfhelp/2024-1/Content/Artificial%20Intelligence/Deep%20Learning%20Tool/Workflow%20and%20Best%20Practices.htm) — recommends *"calibrating the intensity scale of images to a set of calibration standards to train a 'universal' model."*

---

## Part B: Train the Segmentation Models

Train **one model per structure** (Hair, Sweat Glands, Blood Vessels). Because Dragonfly only allows **one session open at a time**, we use an **iterative approach**: train on one sample, publish the model, open the next sample, import the model, add that sample's data, continue training, and repeat.

> **Important prerequisite**: The Segmentation Wizard button is **grayed out on empty sessions**. You must have at least one volume loaded in the workspace before the Wizard becomes available.

### B.1 Overview: Iterative Training Workflow

Since you cannot load 12 volumes simultaneously, train the model **one sample at a time**, carrying the model forward:

```
Sample 1 → Train from scratch → Publish model
Sample 2 → Import model → Add frames → Continue training → Publish updated model
Sample 3 → Import model → Add frames → Continue training → Publish updated model
...
Sample 12 → Import model → Add frames → Final training → Publish Hair_Model_v1
```

Each round, the model learns from the cumulative data of all previous samples plus the current one.

**Estimated time per sample:**

| Step | Time |
| ---- | ---- |
| Open session | 1–2 min |
| Launch Wizard + Data/Model Selection | 1–2 min |
| Add 5–8 frames + fill from Multi-ROI | 3–5 min |
| Paint Background labels on frames | 2–5 min |
| Training (U-Net 2.5D, 3 slices, 8 frames, 100 epochs) | ~1h 40min (first sample); ~30–50 min (subsequent, with early stopping) |
| Predict + visual evaluation | 2–3 min |
| Refinement (correct predictions, retrain) | 15–30 min per round |
| Exit + Publish model | 1 min |
| **Total per sample (first, from scratch)** | **~2–2.5 hours** |
| **Total per sample (subsequent, with imported model)** | **~45–75 min** |
| **Total per sample (subsequent, with early stopping)** | **~40–60 min** |

A full training campaign of roughly 10 samples × 3 structures ≈ 30 rounds is a multi-day process. Train one structure (e.g. Hair) across all samples first, then the next (SG), then BV. The first few samples take longest; subsequent ones are faster because the imported model already has a head start.

> **Reality check**: A first-pass model trained on a single sample typically produces a recognizable but imperfect prediction with many false positives. Adding more samples and iteratively refining predictions improves accuracy substantially. The investment pays off because applying a trained model to a new sample takes only minutes (vs. 45–85 min manual).

### B.2 First Sample: Start from Scratch

1. **Open the session** for the first training sample.
2. Optionally, **save a backup** of this session under a different name.
3. Click the **Segmentation Wizard** button in the toolbar.

**Data Selection dialog:**

![Data Selection dialog with Image 1 dropdown](images/wizard_01_data_selection_empty.png)

4. In the **Image 1** dropdown, select the **cylinder-masked** volume (the one the mask was applied to, **not** the `(Copied)` backup).

   > **Volume naming convention**: The cylinder mask is applied to the original-named volume (e.g. `<SampleID>_recon`). The `(Copied)` version is the pre-mask backup. Verify by inspecting slices — the masked volume shows a circular cross-section against black background.

   ![Image dropdown showing available volumes](images/wizard_03_image_dropdown.png)

5. Click **Continue**.

**Model Selection dialog:**

![Model Selection dialog — no models available](images/wizard_05_model_selection.png)

6. No models exist yet. Click **"Start without Model"**.

**Segmentation Wizard main interface:**

![Segmentation Wizard fullscreen interface](images/wizard_06_main_interface.png)

The workspace opens with:
- **Center**: 2D slice viewer.
- **Left sidebar ("Main")**: ROI Painter tools.
- **Right sidebar ("Segmentation Wizard")**: Input, Models, and Settings tabs.

### B.3 Define Classes

In the right sidebar, **Classes and Labels** section:

![Classes and Labels section with Hair and Background](images/wizard_07_classes_and_labels.png)

1. Two default classes appear: one named class + "Background."
2. **Rename** the first class to `Hair` (double-click the name).
3. Ensure `Background` is set as the background class (drag it to the "Background class" box if not already).

### B.4 Prepare a Single-Class Multi-ROI

The Wizard's "Fill from Multi-ROI" feature creates **one class per component** in the Multi-ROI. Since your clean `Hair` Multi-ROI has hundreds of individually labeled components (one per hair), filling directly from it creates hundreds of classes — not what we want.

**Solution: create a single-class Multi-ROI before entering the Wizard** (or merge classes inside the Wizard after filling).

**Method A — Convert before the Wizard (recommended):**

1. In the main Dragonfly workspace (before opening the Wizard), select the clean `Hair` Multi-ROI.
2. **Right-click → Convert to ROI** (or **Boolean Union**) to collapse all components into a single binary ROI.
3. Run **Connected Components 3D** (26-connected) on that binary ROI to create a new Multi-ROI where every voxel belongs to one class — call it `Hair Single Class`.
4. Use this single-class Multi-ROI when filling frames in the Wizard.

**Method B — Merge inside the Wizard:**

1. Fill a frame from the original multi-component `Hair` Multi-ROI (choose "Import all classes").
2. The Wizard creates one class per component (Class 1, Class 2, …).
3. Select all component classes, then click **Merge** in the Classes and Labels panel to combine them into a single `Hair` class.
4. Delete any leftover empty classes.

> Method A is faster and cleaner. Method B works but requires extra cleanup per frame.

### B.5 Add Frames and Fill from Multi-ROI

> **CRITICAL: Do NOT import all frames from the Multi-ROI.** Importing every labeled slice (often more than 100 frames) freezes the Wizard and is unnecessary. The Wizard is designed for **5–10 representative frames**, not hundreds.

**How to add frames:**

1. In the **Frames** section of the Input tab, click the **"+" (Add)** button to add a new frame.
2. **Position the frame** on a slice that has clear, representative structures. Navigate to the desired slice in the 2D viewer first, then add the frame.
3. Repeat to add **5–8 frames total**, evenly distributed through the volume:
   - 2 frames near the top (epidermis region — helps the model learn hair cross-sections).
   - 2–3 frames in the middle (bulk of follicles).
   - 1–2 frames near the bottom (hair roots / deeper structures).
   - 1–2 "negative" frames in the dermis region with **no hair** — label these as Background-only to teach the model what is NOT hair (reduces false positives in deep tissue).

**How to fill frames with labels:**

4. **Right-click** a frame → **"Fill from Multi-ROI"** → select the **single-class Multi-ROI** (from B.4 Method A) or the original Multi-ROI (then merge classes per Method B).
5. The frame is now populated with `Hair` labels wherever your manual segmentation exists on that slice.
6. Repeat for each frame.

**How to verify fill is correct:**

7. Click on each frame and visually check that the Hair labels match what you expect on that slice. If some hairs appear "cut in half" or partially labeled, this may be from the manual ROI Painter cuts — it is acceptable for training as long as most of the frame has correct labels.

### B.6 Paint Background Labels

> **CRITICAL: The Train button stays grayed out until BOTH classes have labeled voxels.** Setting a class as "Background" does NOT automatically label unlabeled voxels. You must explicitly paint Background.

1. In the **Classes and Labels** section, select the `Background` class (click on it so it is the active painting class).
2. For **each frame**, use the **ROI Painter brush** to paint a few rough strokes in the areas that are clearly NOT hair (dermis tissue, empty space, gland regions).
   - You do not need to paint every background pixel — a few representative strokes per frame is enough.
   - Focus on areas near hair follicles (so the model learns the boundary) and in the dermis (so it learns not to predict there).
3. After painting, verify the `Background` class count is **> 0** in the Classes and Labels panel.
4. The **Train** button should now become active.

> **For "negative" frames** (dermis-only, no hair): paint the entire frame as Background. This teaches the model that these regions contain no hair.

### B.7 Train the First Model

1. Click **Train** at the bottom of the right sidebar.
2. The **Model Generator** dialog appears. Select **"Generate a single model"** (or a strategy).
3. Configure the model:

   ![Model Generator — U-Net 2.5D configuration](images/wizard_13_model_generator_unet.png)

   **Recommended parameters for Hair and SG models:**

   | Parameter | Value | Why |
   |-----------|-------|-----|
   | Model Type | Deep Learning | Learns spatial patterns, not just intensity thresholds |
   | Architecture | U-Net | Standard for biomedical image segmentation (Ronneberger et al., 2015); works well with small datasets |
   | Input dimension | 2.5D | Captures inter-slice context (hair follicles span many slices) without 3D computational cost |
   | Number of slices | 3 | Target slice + 1 above + 1 below; good balance of context vs. speed |
   | Depth level | 5 | Standard U-Net depth; 5 encoding/decoding levels |
   | Initial filter count | 64 | Standard starting width; doubles at each depth level (64→128→256→512→1024) |

   > **Screenshot cross-check (`images/`):** Some reference screenshots in `images/` were captured with **5** slices and patch **(64, 64, 5)** (an older backbone/optimizer configuration). Use those only as loss-curve examples. The recommended configuration for both hair and sweat-gland models is **3** slices with patch **(64, 64, 3)**.

4. Click **Generate**, then training begins. A progress dialog shows loss curves:

   ![Training progress — early epochs](images/wizard_10_training_progress.png)

   **How to read the training graph:**
   - **X-axis**: Epoch (one full pass through the training data).
   - **Y-axis**: Loss (lower = better).
   - **`loss` (green)**: Training loss — how well the model fits the labeled frames.
   - **`val_loss` (pink)**: Validation loss — how well the model generalizes to data it hasn't trained on directly.
   - **Good sign**: Both curves decrease and eventually plateau.
   - **Overfitting warning**: If `val_loss` starts rising while `loss` keeps falling, the model is memorizing training data instead of learning general patterns. Stop training or add more diverse frames.
   - **Target**: 100 epochs with early stopping (patience = 15). Training halts when `val_loss` hasn't improved for 15 consecutive epochs.

   **Steps per epoch and “time remaining” (U-Net / patch training):** Classic U-Nets are trained on **small patches** sampled from the image (full slices or volumes are too large for GPU memory). Dragonfly does the same: your **frames** and **patch size** (e.g. 64×64×2.5D), **stride ratio** (overlap of the patch grid), **batch size**, and **augmentation factor** determine how many **training patches** exist for that session. One **epoch** = one full pass over those patches; **steps per epoch** is essentially “how many minibatches fit in that pass” (on the order of *train patch count* ÷ *batch size*, as the Wizard’s **Train patch count** line reflects). **More frames**, more labeled voxels inside frames, denser sampling (stride), or heavier augmentation → **more patches** → **more steps** and **longer wall time per epoch**. This is the same idea whether it is the **first** model or a fine-tune; what changes round-to-round is **that session’s** patch count (see the Hair **Steps/Epoch** column in the training log above — Protocol notes that steps correlate with voxels/frames and class balance). The **time remaining** line is usually a **projection** from recent speed (often toward the **maximum** epoch count, e.g. 100). If **early stopping** ends the run at epoch 30, the UI may still have assumed “~100 epochs worth” of time earlier in the run — so remaining time is an estimate, not a guarantee.

5. **What a healthy first-sample training run looks like:**

   ![Training complete — 100 epochs, loss curves converged](images/wizard_14_training_complete_100epochs.png)

   Expect loss to drop sharply in the first ~25 epochs (e.g. from ~0.06 to ~0.005) and then plateau. Val_loss should track alongside loss with only a small gap; if val_loss begins to rise while loss keeps falling, stop early and add more diverse frames. On a mid-range GPU, a first-sample run of 100 epochs with 6–8 frames and 10× augmentation typically takes 1–2 hours; fine-tuning runs on subsequent samples are shorter because the model starts from stronger weights and early stopping is usually triggered.

6. When complete, click **Predict** to see the model's prediction on the current volume. Evaluate quality visually.

   ![First prediction result — quad view](images/wizard_11_first_prediction_quad.png)
   ![First prediction result — 3D view](images/wizard_12_first_prediction_3d.png)

### B.8 Refinement Loop (Predict → Correct → Retrain)

After the first training round, the prediction will likely have errors (false positives in dermis, missed hairs, etc.). Use this iterative loop to improve the model:

1. **Predict**: Click **Predict** to run the model on the current volume.
2. **Evaluate**: Inspect the prediction in 2D and 3D. Look for:
   - False positives (model predicts hair where there is none — e.g. in dermis or gland regions).
   - False negatives (model misses actual hairs).
   - Boundary quality (are hair outlines clean or noisy?).
3. **Fill new frames from the prediction**: Right-click a new frame → **"Fill from Prediction"**. This populates the frame with what the model predicted.
4. **Correct the prediction**: Using the ROI Painter, fix mistakes in the prediction-filled frame:
   - Erase false positives (paint them as Background).
   - Paint in missed hairs (paint them as Hair).
   - Fix boundaries.
5. **Retrain**: Click **Train** again. The model continues from its current weights, now learning from both the original labels AND your corrections.
6. **Repeat** 1–2 rounds per sample until predictions are visually acceptable.

> **How many refinement rounds?** 1–2 per sample is usually enough. Diminishing returns after that — move on to the next sample to add diversity rather than perfecting one sample.

### B.9 Exit and Publish the Model

1. Click **Exit** in the right sidebar.
2. The **Publish Models** dialog appears. Select the model(s) you want to publish (check the box).
3. **Rename** to something clear: e.g. `Hair_v1_sample1`.
4. Click OK. The published model becomes available for future sessions.
5. A **Session Group** is saved automatically in the Data Properties panel. This contains all frames, labels, trained models, and parameters.

### B.10 Subsequent Samples: Import Model and Continue Training

For each additional training sample:

1. **Close** the current session. **Open** the next sample's session.
2. Click the **Segmentation Wizard** button.
3. In **Data Selection**, select the cylinder-masked volume for this sample. Click **Continue**.
4. In **Model Selection**, the model published from the previous round should be available in the dropdown. Select it and click **"Start with Model"**.
5. The Wizard opens with the previously trained model loaded on the **Models tab**.
6. **Define classes** (`Hair`, `Background`) — same names as before.
7. **Add 5–8 frames** manually (B.5), fill them from this sample's single-class Multi-ROI.
8. **Paint Background** on each frame (B.6).
9. Click **Train**. The model **continues training from its previous weights** — it does NOT erase previous training. The model retains everything it learned from prior samples and adds the new sample's data on top. This is fine-tuning, not retraining from scratch.
10. Optionally run the **Refinement Loop** (B.8): Predict → Correct → Retrain.
11. **Exit and Publish** the updated model (e.g. `Hair_v1_sample2`).
12. Repeat for all remaining training samples.

**Typical behavior on the first cross-sample test:** a single-sample model imported into an unseen sample will produce a recognizable prediction on the structures of interest but also substantial noise (false positives in dermis and non-target structures). Fine-tune with 6 properly labeled frames from the new sample to correct this.

![Cross-sample fine-tuning — working view with frame labels (top) and initial model prediction showing noise (bottom).](images/wizard_15_24-22_prediction_vs_frame.png)

> **Known limitation — Wizard 2D view**: The Segmentation Wizard locks the view to a single 2D slice plane. You cannot rotate to a different viewing angle (XZ or YZ) within the Wizard. With the cylinder-masked volume, this results in an **oval-shaped cross-section** view that can make it harder to identify structures compared to the standard quad-view in the main workspace. Work around this by carefully scrolling through slices and using the frame thumbnails to pick well-populated slices.

After the final training sample, publish the model under a descriptive name (e.g. `Hair_Model_v1`).

**Suggested training order (diversity-first):**

Sort candidate training samples so that each round introduces a diversity axis the model has not yet seen. A typical order is:

1. A starting sample from one group at a moderate intensity distribution.
2. A sample from the opposite group, ideally from a different scanning session (covers a different gray-value range).
3. A sample from another scanning session to expand intensity coverage.
4. A sample from the first group with extreme characteristics (e.g. highest or lowest structure count).
5. A sample that sits at the edge of the already-covered intensity range (lowest or highest gray-value lower bound so far).
6. A sample that overlaps only marginally with the intensity ranges seen so far (forces the model to learn from new-but-contiguous data).
7. A reinforcement sample for whichever group is under-represented in the set above.

After round 7, evaluate the model on held-out samples (B.11) before deciding whether additional rounds are justified.

**Held out for validation** (do NOT train on these until the 7-sample model is evaluated):

| Held-out role | Guidance |
| ------------- | -------- |
| Group A hold-out | Pick a sample from group A that was not used in training. It stresses generalization within that group. |
| Group B hold-out | Same, for group B. If your cohort has only one group, reserve at least two samples from different scanning sessions instead. |

**Reserve samples** (available if validation reveals gaps — do NOT train proactively):

| Reserve role | When to use |
| ------------ | ----------- |
| Group A epidermis-correction reserve | If the group A hold-out shows over-detection of epidermis after round 7. |
| Group B rebalancing reserve | If fine-tuning on group A causes group B predictions to regress (seesaw effect). |
| Cross-session stress-test reserve | A sample from a scanning session not represented in training, held back for independent evaluation after the two-model or multi-dataset strategy is finalized. |
| OOD sample | Samples whose gray-value lower bound is far above the training distribution. These will likely fail even after calibration; do **not** fine-tune on them — fall back to manual segmentation (see E.4). |

**What a healthy fine-tuning round looks like:**

When fine-tuning on a new sample with a shifted gray-value distribution, expect:

- Training loss to start higher than the baseline run (the model is adjusting to the new intensity range).
- Validation loss to start noticeably lower than the baseline run (the model is retaining prior knowledge).
- Both curves to converge within ~40–50 epochs, usually ending via early stopping.
- Total time shorter than the first-sample run because training starts from strong weights.

![Fine-tuning — training curves early in the run](images/wizard_16_24-22_finetuning_epoch6.png)

![Fine-tuning — training curves approaching convergence](images/wizard_17_24-22_finetuning_epoch66.png)

> **Tip**: Publish the model after every 2–3 samples with a descriptive name (e.g. `Hair_v1_3samples`). This creates checkpoints you can roll back to if later training degrades quality.

### B.11 Evaluate on Held-Out Samples

1. Open a **validation sample** — one the model has **not** been trained on, ideally one per experimental group.
2. Open the Segmentation Wizard → select the volume → **Start with Model** (select the latest Hair model).
3. Click **Predict** to run the model on this sample.
4. Compare the prediction to your manual segmentation. Check:
   - Does the model capture hair follicles without including epidermis or debris?
   - Are individual hairs separated, or are they merged?
   - Is the overall hair count in the right ballpark?
5. If acceptable → proceed to applying the model to unsegmented samples.
6. If poor → add a reserve sample to training and retrain, or adjust model parameters.

> **Recommended evaluation checkpoint**: evaluate after 7 samples trained. If results are good on all held-out samples, proceed to production. If one group still under-performs, prioritize adding more samples from that group.

### B.12 Train Models for Sweat Glands and Blood Vessels

Repeat the B.2–B.11 **mechanics** for each structure (single-class Multi-ROI, 5–8 frames including 1–2 negative frames, paint Background, U-Net 2.5D settings per B.7). Class names: **`SweatGlands`** + `Background` for glands; **`BloodVessels`** + `Background` for vessels.

1. **Sweat Glands**: Use the **diversity-first order in B.12.1** (7 training rounds, then validation checkpoint on hold-outs). Do **not** train on hold-outs until after the checkpoint.
2. **Blood Vessels**: Same process, importing from `BloodVessels` Multi-ROI. Publish as `BloodVessels_Model_v1` (training order can follow the Hair plan in B.10 or a dedicated BV plan when defined).
   - For BV-absent samples: either skip them for BV training, or create empty BV ROIs so the model learns the "no vessel" pattern.

You then have three published models to apply to new samples.

### B.12.1 Sweat glands — diversity-first training order

Same principle as Hair (B.10): alternate groups, cover scanning sessions and SG gray-value ranges early, one volume per Wizard session. **R1** = Start without Model; **R2+** = Start with Model (previous published checkpoint). Selection heuristics:

- **R1**: a clean starting sample, typically from whichever group you have segmented first.
- **R2**: a sample from the opposite group, from a different scanning session; aim for a meaningful jump in mean EqSphD relative to R1 so the model sees size diversity early.
- **R3**: a sample spanning the highest SG gray-value range seen so far, preferably one that also covers BV/SG intertwining cases.
- **R4**: a group-switched sample with the highest SG count in your cohort (density stress test).
- **R5**: the sample with the lowest SG gray-value lower bound (stretches the intensity envelope).
- **R6**: a morphological variant (e.g. atypical "snake-gland" glands) from a previously unrepresented session.
- **R7**: a reinforcement sample for whichever group is under-represented in the set above.

**Validation hold-outs** (do not use in R1–R7): choose one sample per group, held back from all training. Flag any hold-out whose SG gray-value range extends above the highest-trained upper bound as an **OOD warning** — the upper range may fail even after calibration.

**Reserve samples** (corrective rounds 8+ — use only if the checkpoint or production shows gaps): keep a small pool of samples per session/group to bring in for targeted fine-tuning. Document the gap each reserve sample is meant to close.

**Lessons from Hair to apply**: monitor for a **seesaw effect** (one group improves while the other regresses); if sequential fine-tuning cannot hold both groups, either adopt a two-model strategy or switch to multi-dataset training (B.12). **Watch for OOD gray values** on hold-out samples whose intensity distribution sits outside the training envelope; normalize or manually segment. Dragonfly still allows **one volume per Wizard session** — no change to frame/label workflow (B.4–B.7) except ROI name `SweatGlands`.

### B.13 Transfer Models to Another PC

Published models can be exported as portable `.zip` files and imported on any other Dragonfly installation (same major version required).

**Where published models are stored (Dragonfly 2025.1):**

```
%localappdata%\Comet\Dragonfly2025.1\pythonUserExtensions\PythonPluginExtensions\DeepTrainer\models\
```

Each published model is a subfolder containing `model.ORSModel`, training logs (`.csv`), and training parameters (`.json`).

> **Note:** The [Dragonfly Helpdesk article](https://helpdesk.theobjects.com/support/solutions/articles/48001079916) references `C:\ProgramData\ORS\DragonflyNN\...\DeepTrainer\models` — this is outdated. In Dragonfly 2025.1, user-published models are stored under `%localappdata%\Comet\` (per-user), not `C:\ProgramData\` (system-wide). The `ProgramData` path contains only Dragonfly installation files.

**Method: Export to Zip via Deep Learning Tool (recommended)**

This produces a portable `.zip` file (~100 MB per model) containing only the model — no training data.

On the **source PC** (where models were trained):

1. Open Dragonfly.
2. Go to **Deep Learning Tool** (not the Segmentation Wizard).
3. In the **Model list**, select the model you want to share.
4. Click **"Export to Zip"**.
5. Save the `.zip` file to a flash drive or shared location.
6. Repeat for each model to transfer.

On the **destination PC**:

1. Open Dragonfly (same major version).
2. Go to **Deep Learning Tool**.
3. Click **"Import from Zip"**.
4. Navigate to and select the `.zip` file.
5. The model is now available in the Segmentation Wizard's **"Start with Model"** dropdown and in the **Segment with AI** panel.

**Alternative: Direct folder copy**

1. On the source PC, navigate to the models folder (path above).
2. Copy the model subfolder(s) (e.g. `HAIR_SegWiz_<version>_<uuid>`) to a flash drive.
3. On the destination PC, paste into the equivalent path: `%localappdata%\Comet\Dragonfly2025.1\pythonUserExtensions\PythonPluginExtensions\DeepTrainer\models\`. Create the `models` folder if it does not exist.
4. Restart Dragonfly — the models appear in the dropdowns.

> **Source:** [Dragonfly Social — Export a Trained Model](https://dragonflysocial.comet.tech/c/deep-learning/export-an-trained-model) (Dragonfly support confirms the Export/Import to Zip workflow).

---

## Part C: Apply Models to New Samples (Production Workflow)

Once validation has passed, use the production workflow below for each **new** sample that has not been manually segmented.

### Recommended production models

| Structure | Model name | Training method | Notes |
| --------- | ---------- | --------------- | ----- |
| Hair | `HAIR_SegWiz_Standard_v1` | Multi-dataset (all training samples simultaneously) | Universal — trained to work across both experimental groups |
| Sweat Glands | `SG_SegWiz_Standard_v1` | Multi-dataset (all training samples simultaneously) | Universal — trained to work across both experimental groups |
| Blood Vessels | *(no DL model)* | — | Manual segmentation only. BV morphology is too variable for reliable DL detection with the training-set sizes typical of this workflow. |

Multi-dataset training (B.12) is the recommended path because it resolves the seesaw effect observed during sequential fine-tuning.

### C.1 Import and Preprocess

1. **File > Import** the sample's TIFF stack.
2. Set **voxel spacing** to the value used during scanning (same in X, Y, Z; see Protocol 01).
3. Rotate and align the sample (Protocol 02, A.2–A.3).
4. Save the session (Protocol 02, A.5).
5. Apply the **same cylinder mask** used during training (Protocol 02 A.6). Duplicate the volume first (creates a `(Copied)` backup), then apply the mask to the **original-named** volume.
6. **Intensity Scale Calibration** (mandatory — section A.6 above): calibrate the masked volume using Air=0, Hair=10,000 landmarks. The DL models were trained on calibrated data and expect calibrated input. Skipping this step will produce poor or zero results.

### C.2 Run DL Models (Hair + Sweat Glands)

1. Open the **Segmentation Wizard**, select the calibrated masked volume, and click **Start with Model**.
2. Load the production Hair model and click **Predict**. Rename the predicted ROI to `Hair`.
3. Exit the Wizard. Re-open it with the same volume.
4. Load the production SG model and click **Predict**. Rename the predicted ROI to `SweatGlands`.

> **No BV model**: Blood vessels are segmented manually (see C.5 below).

### C.3 Review and Correct — Hair

Inspect the Hair prediction in 2D and 3D, then follow the appropriate path:

**Path A — No fusions or very mild fusions:**
1. Run Process Islands (up to 50 voxels, 26-connected) to remove small noise. Duplicate before running.
2. Run the `ComputeStandardMeasurements` macro (`Utilities > Macro Player > Play All`) — this creates the 26-connected Multi-ROI and computes all 7 measurements in one step.
3. If a few merged pairs remain, try **6-connected** component analysis — this breaks fusions at thin bridges without manual intervention. Re-run on the cleaned ROI.
4. Proceed to C.6 (Analyze, clean, and export).

**Path B — Moderate fusions:**
1. Run Process Islands (up to 50 voxels, 26-connected).
2. Run Connected Components with **6-connected** first.
3. If fused pairs persist, use the **A - B → Union** batch unfusion protocol (Protocol 02, Appendix A) to separate them.
4. If <5 pairs remain fused, accept as-is — negligible impact on population statistics.

**Path C — Heavy fusions:**
1. If >10 fused groups or the prediction is mostly one large blob, **manual re-segmentation is faster** than the unfusion protocol.
2. Discard the DL prediction and segment Hair manually following Protocol 02, Part B (threshold, ROI Painter cleanup, Connected Components).
3. Budget roughly ~20 min for a full manual Hair segmentation when this path is needed.

> **Decision guide**: the severity of hair fusions does **not** correlate perfectly with any single biological covariate. Always inspect the prediction before choosing a path.

### C.4 Review and Correct — Sweat Glands

1. **Quality gate (mandatory)**: Before accepting SG results, switch to **2D view** and scroll through slices. Verify that the predicted gland outlines match the visible gland boundaries in the volume. Look for:
   - Glands that appear only partially captured (DL detected them but didn't fill the full volume).
   - Missing glands in regions where you can see them in the 2D view.
2. If the quality gate passes: run Process Islands (up to 50 voxels, 26-connected), then run the `ComputeStandardMeasurements` macro (creates 26-connected Multi-ROI + computes all measurements). Proceed to C.6.
3. If the quality gate **fails** (under-segmentation — glands appear incomplete or many are missed): **fall back to manual SG segmentation** (Protocol 02, Part C). In practice this is most common on samples with atypical or intricate gland morphology, where the model detects gland locations but fails to capture their full volume.

> **SG failure mode**: When the SG model fails, it under-segments (partial volumes), not over-segments. Detected objects are real but incomplete. This is a quality issue, not a false-positive issue.

### C.5 Blood Vessel Segmentation (Manual)

There is no DL model for blood vessels. Segment manually:

1. **Visual search**: In the 2D views, scroll through the volume looking for blood vessel cross-sections. These are typically round/oval structures in the mid-to-deep dermis region, darker than hair but distinct from the surrounding tissue.
2. If a BV is found: follow Protocol 02, Part D (threshold, cleanup, Connected Components, measure).
3. If no BV is detected after thorough visual inspection: skip BV export and record the sample as BV-absent.
4. Blood vessels are also sometimes discovered during Hair or SG cleanup (they appear as intertwined structures) — extract them when found.

### C.6 Measurements and Export (Using the `ComputeStandardMeasurements` Macro)

A **recorded Dragonfly macro** can automate Connected Components + Compute Measurements in a single action, replacing manual menu navigation for both steps.

**Macro name**: `ComputeStandardMeasurements`
**Location**: `Utilities > Macro Player` (select from the dropdown, then click **Play All**)
**Applies to**: Hair, Sweat Glands, and Blood Vessels — the same macro works for all three structures.

**What the macro does (in order):**
1. Runs **Connected Components 3D (26-connected)** on the selected ROI, creating a Multi-ROI.
2. **Computes the 7 standard measurements** on the resulting Multi-ROI:
   - 2D Maximum Feret Diameter
   - 2D Area Equivalent Circle Diameter
   - 2D Minimum Feret Diameter
   - Volume
   - Equivalent Spherical Diameter
   - Surface Area (voxel-wise)
   - Sphericity

**Workflow — how to use it:**

1. After the DL model produces a segmentation ROI (or after manual thresholding), **delete the Background class** from the predicted Multi-ROI so only the structure of interest remains as a single ROI.
2. Run any needed cleanup (Process Islands up to 50 voxels, ROI Painter cuts, 3D brush) on the ROI.
3. Open `Utilities > Macro Player`, select `ComputeStandardMeasurements`, and click **Play All**.
4. The macro creates the Multi-ROI and computes all measurements automatically.
5. Proceed to **Analyze and Classify** (right-click Multi-ROI > Measurements and Scalar Values > Analyze and Classify Measurements) to clean remaining debris using the visual batch deletion technique.
6. **Export Scalar Values** to CSV:
   - `<project_root>/03_RESULTS/Hair/<SampleID>_hair.csv`
   - `<project_root>/03_RESULTS/SweatGlands/<SampleID>_sg.csv`
   - `<project_root>/03_RESULTS/BloodVessels/<SampleID>_bv.csv`
7. Save the Dragonfly session.

> **Note**: The macro does not appear in the right-click "Execute Macro..." context menu (the first-step parameter type does not match MultiROI). Always launch it from `Utilities > Macro Player`. If 6-connected separation is needed (e.g. fused SG or fused hairs), run 6-connected manually first, then use the macro's measurement step on the result — or re-run the macro on the 6-connected ROI.

> **Macro file location**: `%LocalAppData%\ORS\Dragonfly2025.1\pythonUserExtensions\Macros\`. To share with another PC, copy the `.py` file to the same path on the other machine.

#### Combined Model+Compute Macros

Two new macros combine the DL model inference with all post-processing into a single automated pipeline. After manual standardisation (adding cylinder, calibrating, etc.), a single macro call takes the reconstruction from standardisation through to measurement-ready Multi-ROI.

**`HairModelCompute`** — Hair end-to-end macro:
1. Runs the Hair DL model on the reconstruction
2. Renames the predicted ROI to "H"
3. Removes the "Background" class from the ROI
4. Runs Connected Components 3D (26-connected), creating a Multi-ROI
5. Renames the Multi-ROI to "Hair"
6. Computes the 7 standard measurements

**`SGModelCompute`** — Sweat Glands end-to-end macro:
1. Runs the SG DL model on the reconstruction
2. Renames the predicted ROI to "SG"
3. Removes the "Background" class from the ROI
4. Runs Connected Components 3D (26-connected), creating a Multi-ROI
5. Renames the Multi-ROI to "SweatGlands"
6. Computes the 7 standard measurements

**Workflow with the combined macros:**
1. Standardise the sample (add cylinder, calibrate, etc.)
2. Run `HairModelCompute` on the reconstruction — wait for completion
3. Run `SGModelCompute` on the reconstruction — wait for completion
4. Perform any manual cleanup needed (Process Islands, unfusion, debris removal) on the resulting Multi-ROIs
5. Re-export measurements if cleanup altered the objects
6. Segment BV manually as before
7. Export CSVs and save session

**Time saving**: These macros eliminate 4 manual steps per structure (rename ROI, delete Background, run Connected Components, compute measurements), saving approximately 2–3 minutes per structure. Combined with the model run, the operator goes from standardisation to measurement-ready output with a single click per structure.

> **Note**: The `ComputeStandardMeasurements` macro remains useful for BV (which has no DL model) and for re-running measurements after manual cleanup on any structure.

**Known limitation — "Shape as Mask" not captured in macros**:

The macros cannot use the interactive "Shape as Mask" shortcut for region-constrained model inference. When running Segment with AI manually, clicking "Shape as Mask" creates a temporary mask from a Shape (e.g. cylinder) on-the-fly — but this is a **UI-layer shortcut** that is not recorded into the macro's `applySegmentationModel` step. At playback, the model runs on the full reconstruction (~2 min instead of ~30 sec). The mask dropdown shows no items because it filters for `ROI` objects, and the cylinder is a `Shape`, not an `ROI`.

**Root cause**: Dragonfly distinguishes between Shapes (geometric primitives like cylinders) and ROIs (voxel-based binary masks). The "Shape as Mask" button converts a Shape to a temporary ROI internally, but macros record the `applySegmentationModel` call without this conversion. The macro's `: mask` parameter (step 5) expects an `ROI`-typed object, and Shapes do not appear in the dropdown.

**Fix — create an ROI from the cylinder**:
1. Create and resize the cylinder for Hair/SG height as usual
2. Select the **reconstruction** → go to **ROI Tools** panel → click **"New"** (creates an empty ROI with reconstruction geometry)
3. **Right-click the cylinder** → **"Add to ROI..."** → select the empty ROI just created
4. This fills all voxels inside the cylinder into the ROI, creating a proper mask
5. Run the macro — the `: mask` parameter auto-selects the ROI (due to "Automatically select the only occurrence when there is a single element available" checkbox)
6. Model runs in **~30 seconds** instead of ~2 minutes
7. Delete the ROI when done with this sample

**Per-sample workflow (auto-ROI macros)**:

Both macros (`HairModelCompute` and `SGModelCompute`) can be updated to **automatically create the ROI from the cylinder**, so the manual steps of creating an empty ROI and adding the cylinder to it are no longer needed. This saves dozens of clicks per sample.

1. Standardise (cylinder, calibration).
2. Resize the cylinder to match the Hair / SG height of interest.
3. Run `HairModelCompute` → ROI auto-created from cylinder, mask auto-selected → ~30 sec.
4. Run `SGModelCompute` → ROI auto-created from cylinder, mask auto-selected → ~30 sec.
5. Segment BV manually on the original reconstruction.

> **Note**: If multiple ROIs exist in the session, the macro will prompt for selection instead of auto-selecting. Keep only one ROI available when running the macros, or manually select the correct one when prompted.

### C.7 Validation and Logging

After exporting, analyze and document the results:

1. Update the Master CSV:
   ```bash
   python scripts/populate_master_csv.py \
     --results-dir <project_root>/03_RESULTS \
     --master-csv <project_root>/master_measurements.csv
   ```
2. Compare the new sample's per-structure means against the per-group ranges you have established from earlier samples (see Protocol 02 B.7 / C.6 / D.5).
3. Record the sample's results in a production log with metrics, timing, and any notable observations (see Protocol 04 Part E for a compatible template).

### C.8 Typical timing

| Sample profile | Typical time | Notes |
| -------------- | ------------ | ----- |
| Clean (no fusions, fast SG cleanup) | ~12–16 min | Best case |
| Moderate (minor fusions, some cleanup) | ~15–25 min | 6-connected cleanup or limited manual work |
| Heavy fusions | ~24–32 min | May require full manual hair re-segmentation |
| Manual fallback (DL discarded, operator re-segments) | ~30–55 min | Budget ~40 min on average when only Hair or only SG needs manual re-segmentation |
| **DL-era mean (pure DL-accepted samples)** | **~15 min** | Substantial speed-up over a fully manual per-sample workflow (typically ~50 min) |

---

## Part D: Iterative Model Improvement

### D.1 Retrain Schedule

| Milestone | Action |
| --------- | ------ |
| After round 7 | **Validation checkpoint**: test the current model on held-out samples from each group. Decide whether reserve samples are needed. |
| New scanning session appears | Manually segment 2 samples from that session and fine-tune (or add to a multi-dataset retraining run). Test on a 3rd unseen sample from the session. |
| After ~10 training samples | **Production-ready model**. Begin AI-assisted segmentation of remaining samples. |
| After every 10–20 AI-processed samples | Add the best corrected segmentations to the training set and retrain (incremental improvement). |
| If predictions degrade for a new batch | Add 2–3 manually segmented samples from that batch and retrain. |

### D.2 Retrain Procedure

1. Add newly corrected segmentations to the training sessions (same ROI names, same preprocessing).
2. Retrain the Hair, Sweat Glands, and Blood Vessel models.
3. Increment the model version: `Hair_Model_v2`, etc.
4. Re-evaluate on a held-out sample before deploying.

### D.3 Tracking

- Record which model version was used for each sample (e.g., in a Master CSV note column or a per-sample log).
- Keep previous model versions saved so you can compare or roll back.

---

## Part E: Troubleshooting

### E.1 Wizard Setup Issues

| Problem | What to try |
|--------|-------------|
| **Segmentation Wizard button is grayed out** | The session must have at least one volume loaded. Open a session that contains a volume, or import one first. An empty session cannot launch the Wizard. |
| **Session won't load / TIFF not found** | Use **Relocate** when loading the session and point to the correct local TIFF path. |
| **"Which ROI is the label?"** | Use only the **clean Multi-ROI** (`Hair`, `SweatGlands`, `BloodVessels`). Not the initial ROIs (`H`, `SG`). |

### E.2 Frame and Label Issues

| Problem | What to try |
|--------|-------------|
| **Wizard freezes after importing frames** | You imported too many frames (e.g. 186). Force-close Dragonfly, restart, and add only **5–8 frames manually** instead. The Wizard's UI is not designed for hundreds of frames. |
| **Fill from Multi-ROI creates hundreds of classes** | Your Multi-ROI has individually labeled components. **Solution**: Convert to a single-class Multi-ROI first (B.4 Method A), or merge all component classes into one after filling (B.4 Method B). |
| **Fill from Multi-ROI only accepts Multi-ROIs, not binary ROIs** | The "Fill from Multi-ROI" command requires a Multi-ROI. If you have a binary ROI, run Connected Components 3D on it first to create a Multi-ROI, then fill from that. |
| **Some hairs appear "cut in half" in filled frames** | This is likely from manual ROI Painter cuts during the original segmentation. Acceptable for training — the model learns from majority patterns. Pick frames where coverage is most complete. |
| **Background class count stays at 0** | Setting a class as "Background" does NOT auto-label voxels. You must **manually paint** background regions on each frame using the ROI Painter while the Background class is active (see B.6). |
| **Cannot change view angle / oval-shaped view** | The Wizard locks the 2D view to a single slice plane — you cannot rotate to XZ or YZ views. The cylinder-masked volume appears as an oval cross-section at oblique slices. Scroll through slices to find well-populated ones for labeling. This is a known Wizard limitation. |

### E.3 Training Issues

| Problem | What to try |
|--------|-------------|
| **Train button is grayed out** | Both classes need labeled voxels. Check that the Background class count is > 0. Paint background areas on frames (B.6). For deep learning models, ensure you have at least 2–3 frames with labels in both classes. |
| **Training very slow** | Use a machine with a capable GPU. With 5 frames and U-Net, expect ~45 min. Reduce frame count for faster iteration during testing. |
| **`val_loss` rises while `loss` falls** | Overfitting — the model memorizes training data. Add more frames from diverse regions, add "negative" dermis-only frames, or stop training earlier. |
| **Model only works on the training sample, fails on others** | Train on too few samples. Add frames from at least 3–4 diverse samples (different breeds, different sets) before evaluating generalization. |

### E.4 Prediction Quality Issues

| Problem | What to try |
|--------|-------------|
| **Model predicts noise or wrong structures** | Ensure training used only **clean Multi-ROIs**. Add "negative" frames (Background-only) in dermis regions. Use the Refinement Loop (B.8) to correct false positives and retrain. |
| **Model predicts large false positives in dermis/gland region** | Add 1–2 frames positioned entirely in the dermis (below hair follicles) labeled as Background-only, and retrain. |
| **Model misses thin or faint hair follicles** | Include samples where those structures are clearly labeled; retrain with more examples. Ensure frames include slices with thin hairs. |
| **Model merges adjacent sweat glands** | After prediction, re-run Connected Components with 6-connected instead of 26-connected. Use the visual batch deletion technique for cleanup. |
| **Model predicts BVs where there are none** | Include BV-absent samples in training as negative examples (empty BV ROI). |
| **Predictions differ a lot between groups** | Ensure samples from both (or all) groups are in training. If still poor, consider a group-specific model as a last resort. |
| **Gray-value differences between sessions** | The model learns patterns, not absolute values. Ensure diverse sessions are represented in training. If a new session has very different values, manually segment 2–3 samples from it and retrain. |
| **Out-of-distribution (OOD) gray-value failure** | If a sample's gray-value lower bound is far above the training distribution, the model may **completely fail** — inverting predictions so noise is classified as the target structure and real tissue as Background. **Do not fine-tune on these outliers** (risk of destabilizing predictions on normal-range samples). Instead, normalize the volume against a reference (see `scripts/normalize_volume.py`) or manually segment the outlier. Flag any sample whose gray-value lower bound is far outside the training envelope for manual processing. |
| **Connected noise attached to BVs** | Use ROI Painter Multi-slice cut in a lateral 2D view to separate the noise from the vessel, then re-run Connected Components (Protocol 02, Part D). |
| **Lots of debris on an unseen sample** | Expected with a model trained on only 1 sample. Add more training samples for diversity. The debris decreases as the model sees more varied data. |

---

## Summary Checklist

**Preparation:**

- [ ] **~10 manual segmentations** completed (Protocol 02) and saved with clean Multi-ROIs and consistent names.
- [ ] All training sessions verified: cylinder-masked volume, correct spacing, consistent ROI names.
- [ ] Single-class Multi-ROIs created for each structure (Hair, SG, BV) per sample (B.4).
- [ ] **At least 2 held-out samples** selected for validation (one per experimental group, if applicable).

**Training (per structure):**

- [ ] 5–8 frames per sample, evenly distributed, with 1–2 negative (dermis-only) frames.
- [ ] Background painted on every frame (B.6) — Train button active.
- [ ] Model trained iteratively across training samples using "Start with Model" for each subsequent sample.
- [ ] Refinement Loop used (Predict → Correct → Retrain) at least once per sample.
- [ ] Model published with checkpoints every 2–3 samples.
- [ ] Final models: `Hair_Model_v1`, `SweatGlands_Model_v1`, `BloodVessels_Model_v1`.

**Validation and Application:**

- [ ] Validation results acceptable on held-out samples.
- [ ] New samples processed: import → cylinder mask → run models → review/correct → Connected Components → measure → export CSV → update Master CSV.
- [ ] Master CSV updated (`scripts/populate_master_csv.py`) and new sample compared against the per-group ranges from earlier samples.
- [ ] Retraining plan in place whenever a new scanning session or unseen sample type appears.
- [ ] Incremental retrain planned after every 10–20 AI-processed new samples.

**Lessons Learned:**

- [x] Do NOT import all frames from Multi-ROI — use 5–8 manual frames.
- [x] Must paint Background explicitly — the "Background class" setting alone is insufficient.
- [x] Convert Multi-ROIs to single-class before filling frames (avoids hundreds of classes).
- [x] A single-sample model produces a recognizable but imperfect prediction — diversity matters more than frame count per sample.
- [x] Expect roughly 1–2 h training time with U-Net 2.5D (3 slices) on 8 frames and 100 epochs on a typical workstation GPU. Subsequent samples are usually faster (~30–50 min) thanks to early stopping.
- [x] Target a final loss / val_loss plateau around epoch 40–50 with no overfitting (val_loss should not drift upward).

---

## References

- **Protocol 01**: `01_TXM_to_TIFF_Conversion.md` — TXM-to-TIFF conversion, voxel spacing.
- **Protocol 02**: `02_Manual_Segmentation_Dragonfly.md` — Full manual segmentation workflow; ROI names; cylinder mask; measurements and export.
- **Protocol 04**: `04_Data_Management_and_Scripts.md` — Master CSV, comparison scripts, data update workflow.
- **Dragonfly tutorials**: `tutorials/Dragonfly_Tutorials.md` — Phase 2 (AI & Automation), especially "AI Segmentation Wizard" and "Applying trained models to new data."
