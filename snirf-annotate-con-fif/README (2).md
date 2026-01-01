# SNIRF → Annotated FIF (MNE) using Excel TimeFrames

This script batch-creates **MNE annotations** for fNIRS recordings by
reading start/end timeframes from an Excel sheet and writing out
**annotated `.fif` files** for each subject's **pre (pr)** and **post
(po)** sessions. It also includes an optional interactive visualization
step.

------------------------------------------------------------------------

## What this script does

For each subject:

1.  Loads two SNIRF files (`SXX-pr.snirf`, `SXX-po.snirf`)
2.  Reads timing information from `TimeFrames_Epoch.xlsx`
3.  Converts Excel timeframes (ms → s)
4.  Aligns timestamps using the recording start time (`meas_date`) and a
    manual timezone offset
5.  Adds MNE annotations:
    -   `Baseline` → always added to the **pr** recording
    -   Task trials → `{fatigue}-{activity}-{stimulation}-{trial}`
6.  Saves annotated files as:
    -   `SXX-pr-annotated_raw.fif`
    -   `SXX-po-annotated_raw.fif`
7.  Optionally opens an interactive MNE raw viewer

------------------------------------------------------------------------

## Inputs

### Required per subject

-   `SXX-pr.snirf`
-   `SXX-po.snirf`

### Required Excel file

`TimeFrames_Epoch.xlsx` with columns: - `Subject` - `Start_timeFrame`
(ms) - `End_timeFrame` (ms) - `Activity` - `Fatigue` (`pr` / `po`,
optional for Baseline) - `Stimulation` - `Trial`

All files are expected in the same directory as the script unless paths
are modified.

------------------------------------------------------------------------

## Outputs

For each processed subject: - `SXX-pr-annotated_raw.fif` -
`SXX-po-annotated_raw.fif`

Console logs indicate missing files or successful saves.

------------------------------------------------------------------------

## How to run

### Jupyter Notebook

The script uses:

``` python
%matplotlib qt
```

Run directly inside a Jupyter notebook with a Qt backend installed.

### Python script

Replace the Jupyter magic with:

``` python
import matplotlib
matplotlib.use("Qt5Agg")
```

Then run:

``` bash
python annotate_snirf_to_fif.py
```

------------------------------------------------------------------------

## Dependencies

-   `mne`
-   `pandas`
-   `pyqt5` or `pyside2` (for interactive plots)

Install example:

``` bash
pip install mne pandas pyqt5
```

------------------------------------------------------------------------

## Configuration notes

### Subjects

Edit the subject list:

``` python
subjects_to_analyze = ["S5", "S6", ..., "S24"]
```

### Timezone offset

The script applies a manual offset:

``` python
+21600  # seconds
```

Use `18000` for earlier subjects or `0` if timestamps are already
aligned.

### Visualization target

``` python
file_to_visualize = "S19-pr-annotated_raw.fif"
```

------------------------------------------------------------------------

## Annotation format

-   **Baseline**
    -   Label: `Baseline`
    -   Added only to the pre-fatigue (`pr`) file
-   **Task trials**
    -   Format:

            {fatigue}-{activity}-{stimulation}-{trial}

    -   Example:

        -   `pr-TWEO-NG-T1`
        -   `po-FPEONF-G-T2`

These labels are designed for downstream epoch-based processing
pipelines.

------------------------------------------------------------------------

## Troubleshooting

-   **Missing files**
    -   Ensure SNIRF files exist and filenames match subject IDs.
-   **Shifted annotations**
    -   Verify the timezone offset and Excel reference time origin.
-   **Interactive plot not opening**
    -   Install a Qt backend and restart the kernel.

------------------------------------------------------------------------

## Notes

-   Assumes a single Baseline segment at experiment start.
-   Rows without `Fatigue` are skipped unless the activity is
    `Baseline`.
-   Missing `Stimulation` or `Trial` fields are filled with placeholders
    (`?`, `T?`).

------------------------------------------------------------------------
