# fNIRS Epoch-Based Processing Pipeline (MNE/MNE-NIRS)

This repository contains an **epoch-based fNIRS processing pipeline** built on **MNE-Python** and **MNE-NIRS**. It loads **annotated `.fif` raw files**, applies a configurable cleaning pipeline, converts OD → Hb (MBLL), and exports **per-channel features**, **ROI-level features (two methods)**, and **QC logs**.

---

## What this script does

### 1) Loads subject `.fif` files (CF + SD)
- **CF subjects** (S1–S24) are read from:
  - `CF_DIR/<Subject>-pr-annotated_raw.fif`
- **SD subjects** (S25–S48) are mapped to source files:
  - `SD_DIR/S<SubjectNum-24>-6pm-annotated_raw.fif`

> Example: `S26` reads `SD_DIR/S2-6pm-annotated_raw.fif`

### 2) Processing pipeline (configurable toggles)
The pipeline is applied in the following conceptual order:

1. Raw → **Optical Density (OD)**
2. *(Optional)* **Linear detrend** on OD  
3. **SCI-based channel dropping** (scalp coupling index)
4. **TDDR** (Temporal Derivative Distribution Repair)
5. **Short-channel regression (SCR)** (if short channels exist)
6. **Band-pass filter (BPF)** (default 0.01–0.5 Hz, Butterworth IIR)
7. **Post-filter QC drop** based on range/variance outliers
8. *(Optional)* **Power-based BAD annotations** and masking
9. **Keep long channels only** (optional)
10. **Prune & reorder wavelength pairs** (ensure valid pairs for MBLL)
11. OD → **Hemoglobin (HbO/HbR/HbT)** using Beer–Lambert (MBLL)
12. **Baseline subtraction** using the **middle X minutes** of the single “Baseline” annotation
13. Parse trials using annotations (Activity–Stim–Trial) and compute features

---

## Trial parsing requirements

The script expects trial annotations with names that match:

```
<something>-<Activity>-<Stimulation>-<T1|T2>
```

Examples:
- `S1-pr-TWEO-NG-T1`
- `S1-pr-TWEO-G-T2`

**Baseline annotation**
- The baseline segment must include the word `Baseline` (case-insensitive).
- Baseline subtraction uses **the middle `BASELINE_MIDDLE_MIN` minutes** of that segment.

---

## ROI definitions (HbO only)

ROIs are defined in the dictionary `brain_regions`, mapping ROI names to **HbO channel labels** (e.g., `"S1_D1 hbo"`).  
Two ROI feature methods are computed:

### ROI Method 1 — Channel-wise features then average
- Compute features for each HbO channel in the ROI
- Average feature values across ROI channels  
- Output columns are prefixed with `ChWise_`

### ROI Method 2 — Average waveform then compute features
- Average HbO waveforms across ROI channels (per trial)
- Compute features on the ROI-averaged waveform  
- Output columns are prefixed with `ROIWise_`

A combined file compares both methods per ROI.

---

## Outputs

### Per-channel feature table
- **`features_get_per_channel_then_average.xlsx`**  
  One row per (trial × HbO channel) with:
  - Mean, AUC, AUC_abs, RMS, SD
  - PeakSigned, PeakPos, PeakPosWinMean2s, PeakNeg, PeakAbs
  - PeakToPeak, PeakWinMean2s

### (Optional) Per-channel time series (wide)
- **`timeseries_get_per_channel_then_average.xlsx`**  
  Exported only if `EXPORT_TIMESER = True`.

### ROI feature table (Method 2: ROI waveform → features)
- **`features_channelAverageFirst.xlsx`**  
  One row per (trial × ROI), includes:
  - `N_ch` = number of contributing HbO channels

### (Optional) ROI time series (wide)
- **`timeseries_channelAverageFirst.xlsx`**  
  Exported only if `EXPORT_TIMESER = True`.

### ROI methods comparison table (Method 1 vs Method 2)
- **`roi_features_both_methods.xlsx`**  
  One row per (trial × ROI), includes:
  - `ChWise_*` features + `ChWise_N_ch`
  - `ROIWise_*` features + `ROIWise_N_ch`

### QC logs (per subject)
Saved in **`qc_logs/`**:
- `<Subject>_sci.csv` — SCI values per OD channel
- `<Subject>_chanvar_qc.csv` — post-filter range/variance QC metrics
- `<Subject>_baseline.csv` — baseline window details
- `<Subject>_trial_qc.csv` — which trials were kept / skipped
- `<Subject>_channel_inclusion_by_trial.csv` — kept/dropped channels and reasons

### (Optional) Plots
Generated only if `DO_PLOTS = True`:
- `Per_Channel_HbO/` — 2×2 panel per subject/activity with all HbO channels
- `perROI_HbO/` — 2×2 panel per subject/activity with ROI overlay lines + legend

---

## Required files / inputs

1. **Annotated `.fif` files**
   - Must contain fNIRS channels with valid wavelength pairing info for MBLL.
   - Must contain trial annotations as described above.

2. **Template Excel**
   - `all_subjects_expected_trials.xlsx`
   - Must include columns:
     - `trialID, Subject, Fatigue, Activity, Stimulation, Trial`

---

## Setup / dependencies

Install the main dependencies:

```bash
pip install numpy pandas matplotlib openpyxl mne mne-nirs scipy
```

> Notes  
- The script uses `glue`-free pandas Excel export; `openpyxl` is recommended.
- Deprecation warnings for `np.trapz` may appear with newer NumPy; it’s safe but you can replace with `np.trapezoid` later.

---

## How to run

### Option A: Run as a script
Save the notebook cell code into a Python file (e.g., `fnirs_epoch_pipeline.py`) and run:

```bash
python fnirs_epoch_pipeline.py
```

### Option B: Run in a Jupyter notebook
Run the main cell; outputs will be written to the working directory.

---

## Key configuration you’ll likely edit

At the top under **CONFIG**:

- `CF_DIR`, `SD_DIR` — paths to annotated FIF folders
- `SUBJECTS_CF`, `SUBJECTS_SD` — which subjects to process
- `TEMPLATE_XLSX` — expected-trials template file

Pipeline toggles (common ones):
- `DO_SCI`, `DO_TDDR`, `DO_SCR`, `DO_BPF`
- `KEEP_LONG_ONLY`
- `DO_BASELINE_SUBTRACT`
- `DO_POSTFILTER_VAR_QC`
- `EXPORT_TIMESER` *(careful: makes very large files)*
- `DO_PLOTS`

QC parameters:
- `SCI_THRESHOLD`
- `BPF_LF`, `BPF_HF`, `IIR_ORDER`
- `QC_METRIC`, `QC_MULT`
- `BASELINE_MIDDLE_MIN`

---

## Interpreting “Kept” vs “Dropped” channels

Channel inclusion is decided per trial using reasons like:
- `short` (short channel)
- `SCI` (SCI below threshold)
- `qcvar` (range/var too large)
- `unpaired_wavelength` (missing wavelength pair after pruning)
- `not_present_after_pipeline`
- `power_masked_majority` (too many NaNs after BAD_power masking)

See: `qc_logs/<Subject>_channel_inclusion_by_trial.csv`

---

## Troubleshooting

### “No short channels available (or removed).”
- SCR requires short channels. If your montage has no short channels (or they were dropped), SCR is skipped.

### “No 'Baseline' annotation found …”
- Baseline subtraction will be skipped; check your raw annotations.

### Large Excel files / slow export
- Set `EXPORT_TIMESER = False` unless you really need timeseries exports.

### “pick_channels() is a legacy function”
- Safe warning; the script still works. You can replace with `inst.pick(...)` later.

---

## Suggested repo name
- `fnirs-epoch-processing`
- `mne-nirs-epoch-pipeline`
- `fnirs-roi-features-qc`

---

## License
MIT
