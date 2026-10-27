# Multi-Channel Ablation for Cross-Speed Motor Fault Diagnosis

Reproduction code for the manuscript
*"Multi-Channel Ablation Reveals Per-Class Sensor Modality Preference in
Cross-Speed Motor Fault Diagnosis"* (submitted to ISA Transactions).

## Overview

This repository contains the complete experimental pipeline for a systematic
sensor-channel ablation benchmark on the public **MCC5-THU Motor dataset**:
six sensor configurations (single-axis vibration up to 6-channel
vibration + current fusion) are evaluated across four cross-speed
domain-generalization scenarios (500-3000 rpm) with five independent seeds,
using a shared lightweight TinyCNN backbone.

## Dataset

The MCC5-THU Motor dataset is publicly available at
<https://doi.org/10.17632/6s3dggj9mw.1>
(Mendeley Data; companion article: Data in Brief 65 (2026) 112583,
DOI 10.1016/j.dib.2026.112583).
Download and extract it, then point the code to the extracted root:

```bash
# Linux / macOS
export MCC5_ROOT=/path/to/mcc5_thu_s1_full
:: Windows cmd
set MCC5_ROOT=D:\path\to\mcc5_thu_s1_full
```

If `MCC5_ROOT` is unset, `./raw_data` is used as the fallback. The raw dataset
is **not** redistributed here; all *result* JSON files needed to regenerate the
figures without re-training are included under `data/`.

**Dataset version note.** The official Mendeley release contains 282 CSV
recordings. The working copy used for this study contains 288 CSVs because one
operating condition was recorded twice (two recordings of
`bearing_outer_H ... 20Nm_3000rpm`, both retained); all extracted segments are
nevertheless derived deterministically from the raw files by
`scripts/extract_steady_state.py`. Extraction yields 274 single-label steady
segments (composite-fault files excluded); segment names embed the exact
segment RPM rounded to one decimal, and re-extraction can differ by +/-0.1 rpm
in the name only (boundary samples, labels, and signal content are bit-exact —
checkable with `--verify`). None of the experiments use the embedded RPM value;
domain assignment uses the integer speed band only.

## Data pipeline

All experiments use **steady-state segments** extracted from the raw
`speed_circulation` (constant-torque variable-speed) recordings. The pipeline
is fully scripted and deterministic:

1. **Steady-state extraction** — `scripts/extract_steady_state.py`
   detects keyphase pulses (rising-edge crossings of 1.0 V), computes the
   instantaneous RPM per inter-pulse interval (`RPM = 60 * 12800 / interval`),
   groups consecutive intervals within +/-10% of each target speed
   (500/1000/.../3000 rpm) into segments, keeps segments with >= 100 pulses,
   and estimates the exact RPM per segment by linear regression on the pulse
   times. Segments are cut exactly at pulse positions and saved as 9-column
   CSVs (`time, keyphase, torque, vib_X, vib_Y, vib_Z, cur_A, cur_B, cur_C`).
   The script has been verified to reproduce the segments used in this study
   bit-exactly (identical boundaries, RPM values, pulse counts, and sample
   content). Usage:

   ```bash
   python scripts/extract_steady_state.py --raw-dir <raw_csv_dir> --out-dir <steady_csv_dir>
   # optional: verify against an existing steady_csv directory
   python scripts/extract_steady_state.py --verify --raw-dir <raw> --steady-dir <steady_csv>
   ```

2. **Window cache** — `scripts/cache_multichannel.py` (uses
   `scripts/data_loader.py` for filename parsing) loads the 274 single-label
   steady segments, extracts 2048-sample windows with 50% overlap
   (stride 1024), z-scores each channel per window, and stores per-domain
   NPZ caches (`domain_{500..3000}.npz`, X: (N, 6, 2048)).
   Composite-fault recordings (filenames containing "and") are excluded.

3. **Experiments** — everything under `experiment/` consumes the NPZ caches.

## Installation

```bash
pip install -r requirements.txt
```

A CUDA-capable GPU is recommended for training (all training runs together
take roughly 43 GPU-hours---18.2 h for the multi-seed campaign, 16.1 h for the recording-disjoint check, plus ~9 h of auxiliary experiments---on an RTX 5070 Ti Laptop GPU (12 GB)). Figure generation
from the shipped result JSONs is CPU-only and takes minutes.

## Repository layout

```
experiment/   training & evaluation pipeline (TinyCNN, ablation, seeds)
scripts/      figure generation and supplementary analyses
data/         result JSON files produced by the pipeline (figures read these)
```

## Reproducing the paper

Main training entry points (each prints progress and supports resuming):

| Script | Produces |
|---|---|
| `experiment/run_multichannel_full.py` | single 6-channel configuration across all 4 scenarios, seed 42 with confusion logging (writes a timestamped `experiment/results_full_*.json`) |
| `scripts/convert_multichannel_full.py` | converts that timestamped raw into `data/multichannel_full_results.json` (confusion counts -> error-normalized fractions; asserts seed 42) |
| `scripts/build_perclass_optimal.py` | derives `data/per_class_optimal_multiseed.json` (per-class oracle) from `multi_seed_results_full.json` |
| `experiment/run_multi_seed.py` + `run_multiseed_extension.py` | first 3-seed campaign (Tables 2-3) |
| `experiment/run_extra_seeds.py` | extends every Table-2 cell to n=5 seeds (adds seeds 789/1011) |
| `experiment/run_ablation.py` | channel ablation across 4 scenarios |
| `experiment/run_leave1000.py` | leave-1000 diagnostic-floor experiment |
| `experiment/run_gradcam_perchannel.py` | per-channel attribution & occlusion |
| `experiment/run_tsne_features.py` | feature extraction for t-SNE |
| `experiment/pareto_analysis.py` | latency/accuracy measurements |
| `scripts/run_window_ablation_multiseed.py` | window-length ablation (3 seeds) |
| `scripts/run_cross_load.py` | cross-load check |
| `experiment/run_fill_gaps_multiseed.py` | re-runs the remaining Table-2 cells at the standard 30-epoch protocol |
| `experiment/run_disjoint_multiseed.py` | recording-disjoint robustness check (leakage audit + A/B split; supports multiple random partitions via a split-seed argument) |
| `experiment/run_learningsize_control.py` | learning-curve control: source-window subsampling (25/50/75%) under the standard split, to separate the data-volume effect from the disjoint constraint |
| `experiment/run_remaining_analysis.py` | combined analysis driver: Grad-CAM training run (outer race), cross-load robustness (20 vs 40 Nm), and window-length ablation |

Core library modules (imported by the entry points above; not run directly):

| Module | Role |
|---|---|
| `experiment/config.py` | shared constants: data root (`MCC5_ROOT`), device, class list, scenario definitions (imported by 14 scripts) |
| `experiment/data.py` | domain loading, windowing, channel selection (imported by 11 scripts) |
| `experiment/models.py` | TinyCNN and dual-branch architectures (imported by 13 scripts) |

**Provenance note.** All Table-2 cells come from raw 30-epoch, 5-seed runs in
`data/multi_seed_results_full.json` (140 entries = 28 cells x 5 seeds
(42/123/456/789/1011), no summary-only or single-run cells remain;
`experiment/run_fill_gaps_multiseed.py` completed the previously missing cells
and `experiment/run_extra_seeds.py` added seeds 789/1011 at the standard
protocol). The earlier
`multi_seed_summary.json` (summary statistics from a 20-epoch campaign) is
deprecated and no longer shipped or read by any figure script.
`data/multichannel_full_results.json` (the source of the confusion-pair table) was converted from a 2026-07-29 seed-42 run of `experiment/run_multichannel_full.py`; the timestamped raw output of that particular run was not retained. The conversion is now codified in `scripts/convert_multichannel_full.py`, so the chain is reproducible for future runs (raw outputs are timestamped and kept under `experiment/`).

The two provenance utilities (`scripts/convert_multichannel_full.py` and `scripts/build_perclass_optimal.py`) instead run **from the repository root**, because they resolve `data/` relative to the working directory.

All figure scripts must be run **from the `scripts/` directory**
(`cd scripts && python fig2_ablation.py`), because they import the local
`fig_common.py` helper; output paths are resolved relative to the script
location, so figures still land in `<repo_root>/figures/`.

Figures (read result JSONs from `data/`, write PDF/PNG to `figures/`; script
names match the paper's figure numbers):

| Script | Paper figure |
|---|---|
| `scripts/fig2_ablation.py` | Fig. 2: channel-ablation accuracy |
| `scripts/fig3_heatmap.py` | Fig. 3: scenario x configuration heatmap |
| `scripts/fig4_confusion_analysis.py` | Fig. 4: waveform + per-class confusion analysis |
| `scripts/fig5_500rpm_evidence.py` | Fig. 5: 500 rpm evidence panels |
| `scripts/fig6_window_ablation.py` | Fig. 6: window-length ablation |
| `scripts/fig7_cross_load.py` | Fig. 7: cross-load results |
| `scripts/fig8_tsne_features.py` | Fig. 8: t-SNE feature visualization |
| `scripts/fig9_decision_tree.py` | Fig. 9: sensor-selection decision tree |
| `scripts/fig10_gradcam.py` | Fig. 10: attribution panels |
| `scripts/fig11_pareto.py` | Fig. 11: efficiency vs. accuracy |

**Waveform panels.** The waveform panels of Figs. 4-5 plot the exact samples
shipped in `data/fig4_signals.json` / `data/fig5_signals.json` (the raw
steady-state CSVs are not redistributed). The figure scripts read these JSONs
and stop with an explicit error if they are missing — they never synthesize
substitute data. With dataset access you can re-extract the snippets via
`scripts/extract_fig_waveforms.py` (segment speeds are matched exactly on the
`_D_speed_<rpm>_` filename token, not on the recording's nominal speed).

## Data dictionary

Every JSON in `data/` is listed below; files not listed here are development
artifacts of the working copy and are intentionally not shipped.

| File | Produced by | Read by / role |
|---|---|---|
| `multi_seed_results_full.json` | `run_multi_seed.py` + `run_multiseed_extension.py` + `run_fill_gaps_multiseed.py` + `run_extra_seeds.py` | main results (28 cells x 5 seeds); fig2/fig4/fig5/fig11 and the paper's Tables 2-3 |
| `multi_seed_results.json` | `run_multi_seed.py` | intermediate of the first campaign; merged by `run_multiseed_extension.py` |
| `per_class_multiseed_summary.json` | `run_multiseed_extension.py` | first-campaign per-class summary (paper tables use the raw 5-seed entries) |
| `per_class_optimal_multiseed.json` | `build_perclass_optimal.py` | per-class oracle; fig3, fig9 |
| `multichannel_full_results.json` | `run_multichannel_full.py` + `convert_multichannel_full.py` | confusion-pair table (seed-42 6-channel run) |
| `run_disjoint_multiseed_results.json` | `run_disjoint_multiseed.py` | recording-disjoint robustness table |
| `learningsize_control_results.json` | `run_learningsize_control.py` | learning-curve control numbers in the text |
| `leave1000_multiseed_results.json` | `run_leave1000.py` | leave-1000 diagnostic-floor claim |
| `window_ablation_multiseed_results.json` / `window_ablation_multiseed_summary.json` | `run_window_ablation_multiseed.py` | fig6 |
| `gradcam_perchannel_results.json` | `run_gradcam_perchannel.py` | fig10 |
| `tsne_features.json` | `run_tsne_features.py` | fig8 |
| `pareto_analysis.json` | `pareto_analysis.py` | fig11 |
| `cross_load_results.json` | `run_cross_load.py` | fig7 |
| `fig4_signals.json` / `fig5_signals.json` | `extract_fig_waveforms.py` | waveform panels of fig4 / fig5 |

## License

MIT (see `LICENSE`).
