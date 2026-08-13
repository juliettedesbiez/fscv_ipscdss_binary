# iPSC Binary Classification Pipeline (Baseline / Serotonin)

This README explains how to run the iPSC binary FSCV classification
pipeline end to end: from raw recordings + labels, to a trained RF+XGB+MLP
ensemble, to a held-out test evaluation, to interpretability figures.

17-feature pipeline (12 original + rise_time, decay_time, ox_red_ratio,
rise_slope, ox_red_lag) — same feature set as the organoid binary pipeline,
run here on iPSC data.

Classes: `0 = baseline`, `1 = serotonin`

---

## 1. Run order

| Step | Script | Reads | Writes |
|---|---|---|---|
| 1 | `make_windows_ipsc_binary.py` | raw recordings (`.npy`/`.txt`/`.csv`) + labels CSV | `window_arrays/*.npy` + `windows_metadata.csv` |
| 2 | `extract_features_ipsc_binary.py` | `windows_metadata.csv` + `window_arrays/` | `features_ipsc_binary.csv` (train only) + `windows_metadata_test_ipsc_binary.csv` (held-out test) |
| 3 | `train_models_ipsc_binary.py` | `features_ipsc_binary.csv` + `window_arrays/` | `models_ipsc_binary_17/rf_model.pkl`, `xgb_model.pkl`, `mlp_model.pkl` (+ prints CV metrics) |
| 4 | `test_models_ipsc_binary.py` | `windows_metadata_test_ipsc_binary.csv` + `window_arrays/` + trained models | `results_ipsc_binary_17/{rf,xgb,mlp}_test.json` + `*_proba.npy` |
| 5 | `ensemble_ipsc_binary.py` | the three `*_proba.npy` files from step 4 | `results_ipsc_binary_17/ensemble_test.json` + `ensemble_proba.npy` |
| 6 | `analyse_ipsc_binary.py` | trained models + test set | interpretability figures (SHAP, permutation importance, gradient saliency) |

**Important workflow rule:** steps 1–3 are where you iterate. Steps 4–6 touch
the held-out test set and should only be run **once**, at the end, when the
CV results from step 3 look good.

`utils_ipsc_binary.py` is not run directly — it's a shared helper module
imported by steps 3–6 (metrics, data loading, `RANDOM_STATE`).

---

## 2. What needs changing before you run anything

Nothing here takes an input/output folder as a command-line argument — paths
are hardcoded at the top of each file.

### `make_windows_ipsc_binary.py`
```python
PLOT_DIR   = r"...\data for binary annotations"   # folder of raw recording files — confirm this still points to the right raw data
LABELS_CSV = r"...\binary output\FSCV_Labels_Nov.csv"   # your annotation CSV — confirm this still points to the right raw data
BASE       = r"...\binary output retrain"   # covers window_arrays/ and windows_metadata.csv
```

### `extract_features_ipsc_binary.py`, `utils_ipsc_binary.py`, `train_models_ipsc_binary.py`, `test_models_ipsc_binary.py`, `ensemble_ipsc_binary.py`, `analyse_ipsc_binary.py`
Each has a `BASE` constant near the top:
```python
BASE = r"C:\Users\julie\OneDrive - Imperial College London\binary output retrain"
```
**This must be identical across all seven files (including `make_windows_ipsc_binary.py`)** — each script writes into `BASE\...` and the next one reads from `BASE\...`.

### `fscv_config_ipsc.yaml`
Check these values match your recording setup before running step 1:
```yaml
fscv_hz: 10.0                 # sampling rate
v_oxidation_start: 200        # oxidation band row indices
v_oxidation_end: 400
v_reduction_start: 800        # reduction band row indices
v_reduction_end: 1000
balance_ratio: 2              # baseline:serotonin ratio when balancing classes
```

---

## 3. How to run

From the folder containing all the scripts and `fscv_config_ipsc.yaml`:

```bash
python make_windows_ipsc_binary.py --config fscv_config_ipsc.yaml
python extract_features_ipsc_binary.py --config fscv_config_ipsc.yaml
python train_models_ipsc_binary.py all
python test_models_ipsc_binary.py all
python ensemble_ipsc_binary.py
python analyse_ipsc_binary.py
```

`train_models_ipsc_binary.py` and `test_models_ipsc_binary.py` accept `rf`,
`xgb`, `mlp`, or `all` — run `all` for the full pipeline, or a single model
name to re-run just one. With no arguments, either script drops into an
interactive prompt asking which model(s) to run.

---

## 4. Outputs you'll end up with (inside `BASE`)

```
binary output retrain/
├── window_arrays/                          (step 1)
├── windows_metadata.csv                    (step 1 — all windows)
├── features_ipsc_binary.csv                (step 2 — train features, 17 cols + label/group/window_id)
├── windows_metadata_test_ipsc_binary.csv   (step 2 — held-out test windows)
├── models_ipsc_binary_17/
│   ├── rf_model.pkl
│   ├── xgb_model.pkl
│   ├── mlp_model.pkl
│   └── mlp_oof_ytrue.npy / mlp_oof_yproba.npy
└── results_ipsc_binary_17/
    ├── rf_test.json / rf_proba.npy
    ├── xgb_test.json / xgb_proba.npy
    ├── mlp_test.json / mlp_proba.npy
    └── ensemble_test.json / ensemble_proba.npy   (step 5 — final result)
```

`ensemble_test.json` is the final number to report for this bundle
(soft-voting RF+XGB+MLP, F1_macro on held-out test).

---

## 5. Quick sanity checks

- Step 1 print-out should show a plausible split of baseline vs. serotonin
  files and a non-zero window count for both classes.
- Step 2 print-out shows the group-aware train/test split — check `Test
  groups` isn't empty and doesn't overlap with train groups.
- Step 3 prints CV `F1_macro` per model — this is your working number,
  iterate here.
- Steps 4–6 should only be run once real CV results look acceptable.
- Step 5 requires all three `*_proba.npy` files to exist in
  `results_ipsc_binary_17/` first.
