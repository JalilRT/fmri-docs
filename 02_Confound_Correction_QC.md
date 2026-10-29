# 02 – Confound correction (RABIES `confound_correction`) and its QC

[← 01 Preprocessing QC](01_Preprocessing_QC.md) | [Home](Home.md) | Next: [03 Analysis QC →](03_Analysis_QC.md)

Goal: remove noise (motion, physiology, scanner drifts) **without removing the brain signal**. There is no single correct strategy: you choose it **per dataset**, using the RABIES data-quality reports.

> Too little cleaning → spurious connectivity (false positives).
> Too much cleaning → network signal removed (false negatives), and over-cleaning is harder to detect.

---

## 1. The current default strategy (`03_confound.sh`)

| Option | Value | Plain meaning |
|---|---|---|
| `--frame_censoring` | `FD_censoring=true, FD_threshold=0.05, DVARS_censoring=false, minimum_timepoint=480` | Remove volumes where the head moved > 0.05 mm. Drop the scan if < 480 volumes remain. |
| `--ica_aroma` | `apply=true, dim=10, random_seed=1` | Automatic ICA denoising (classifier adapted by RABIES for rodents, built on **mouse** data). |
| `--nuisance_regressors` | `WM_signal CSF_signal mot_6` | Regress out mean WM, mean CSF and the 6 motion parameters. |
| `--detrending` | `order=1` | Remove linear drift. |
| `--highpass` / `--lowpass` | `0.01` / `0.1` Hz | Keep the classic resting-state band. |
| `--edge_cutoff` | `30` s | Drop the first and last 30 s (filter edge artefacts). |
| `--smoothing_filter` | `0` | No spatial smoothing. |

Run:

```bash
sbatch neuropsytox_utils/code/fmri_wProc/03_confound.sh <project> [exclusions.txt | "sub-001_ses-T1,..."]
# 03_confound_linear.sh = same options, runs Linear (serial) if MultiProc crashes
```

`TR` is read at the top of `03_confound.sh`. It must be the same value used in 02.

## 2. How to QC confound correction (the loop)

The QC images for confound correction are produced **at the analysis stage** with `--data_diagnosis` (step 04-1 pass 2, see [03 Analysis QC](03_Analysis_QC.md)). So the loop is:

```
03 confound (strategy A) ──► 04-1 pass 1 (group ICA) ──► classify components ──► 04-1 pass 2 (data diagnosis)
       ▲                                                                                  │
       └──────────── change ONE option, rerun 03, compare reports ◄───────────────────────┘
```

Recommended protocol (RABIES, Desrosiers-Grégoire et al. 2024):

1. **Start minimal**: FD censoring + `mot_6` (+ light smoothing). Generate reports.
2. Evaluate them (sections 3–5 below).
3. Add **one** correction, chosen with the table in section 6.
4. Rerun and compare. **Keep the change only if the reports improve.**
5. Repeat until the reports are acceptable.

Before each rerun, rename the previous output so you can compare: `mv rabies/confound_out rabies/confound_out_<strategyName>` (and the same for the analysis folder). Keep a table `confound_strategies.csv` with: strategy name, options, number of scans kept, mean Dice, comments.

Write down the final strategy **before** looking at group differences.

## 3. Scan-level check: the spatiotemporal diagnosis

Folders (after 04-1 pass 2): `rabies/analysis_diagnosis_out/data_diagnosis_datasink/figure_temporal_diagnosis/` and `figure_spatial_diagnosis/`. One figure per scan.

Look at these **4 key features** first:

| Feature | Good (network "specific") | Bad |
|---|---|---|
| **BOLD<sub>SD</sub>** (signal variability map) | Homogeneous, no clear pattern | Bright edges (motion), bright ventricles/vessels (physiology), bright stripes (ghosting) |
| **Global signal covariance** | Grey-matter pattern that looks like a network | Edges, ventricles, whole-brain uniform or noise pattern |
| **Network map** (DR or seed) | Looks like the expected network (e.g. somatosensory, DMN-like) | No structure (**absent**) or noise/edge patterns (**spurious**) |
| **Network vs confound time courses** | Low correlation | Network amplitude follows the confound time course (**spurious**) |

Classify every scan into one of 4 categories:

| Category | What it means | Action |
|---|---|---|
| **Specific** | Network present, noise low | Keep |
| **Unspecific** | Network present but mixed with broad/global signal | Keep, but consider more cleaning (table section 6, row 4) |
| **Absent** | No network visible | Exclude (unless absence is the biology you study, e.g. deep anaesthesia) |
| **Spurious** | Noise drives the "connectivity" | Exclude, or improve cleaning |

Other panels help find the **source** of the problem:

| Panel | Use |
|---|---|
| Power spectrum | Spikes at high frequency → lowpass. Very strong low-frequency power → drift → highpass. |
| Carpet plot | Vertical bands = global events (motion, breathing). Red marks = censored frames. |
| FD and DVARS | Spikes in DVARS not explained by FD → consider DVARS censoring. |
| Variance removed by regression | If large peaks of noise are **not** captured, the regressors are not modelling the confound. |
| Mean WM/CSF/edge signals | If edge/CSF signal tracks the global signal → motion or physiological confounds. |
| CR<sub>SD</sub> maps | Where regression removed variance: edges → motion; ventricles → CSF/physiology. |

## 4. Choose scan-exclusion thresholds (Dice / confound correlation)

Folder: `data_diagnosis_datasink/analysis_QC/sample_distributions/` → `DR<i>_sample_distribution.png` and `DR<i>_outlier_detection.csv` (one per network in `--prior_bold_idx`).

1. Find the scans you rated **absent/spurious** in section 3.
2. Look up their **Dice** (network specificity) and **Conf** (correlation with confound time courses) values in the CSV.
3. Choose thresholds that separate them from the good scans. Do **not** copy thresholds from papers: they depend on SNR and on your cleaning (e.g. lowpass raises Conf).
4. Apply them with `--scan_QC_thresholds` in 04-1 pass 2 / 04-3 (see page 03), e.g.

```
--scan_QC_thresholds '{DR:{Dice:[0.3,0.3,0.3],Conf:[0.25,0.25,0.25],Amp:false}}'
```

(one value per network in `--prior_bold_idx`, in the same order).

## 5. Group-level check: the group statistical report

Folders: `analysis_QC/parametric_stats/` and `non_parametric_stats/` → `DR<i>_QC_maps.png` and `DR<i>_QC_stats.csv`.

| Map | Good | Bad → meaning |
|---|---|---|
| Network variability (SD across scans) | Looks like the network | Edges/ventricles/random → between-scan differences are driven by noise, not biology. Group stats on this network will be unreliable. |
| Correlation with mean FD | Small inside the network | Strong → motion drives connectivity differences. Add mean FD as covariate in step 06, improve motion cleaning. |
| Correlation with temporal DOF | Small | Strong → the amount of censoring changes connectivity. Consider `--match_number_timepoints`. |

Ask: *Is the effect I expect much larger than these confound effects?* If not, the study cannot answer the question without better cleaning.

## 6. Decide – what to change in `03_confound.sh` (in this order of priority)

Adapted from the RABIES "optimise confound correction" table:

| # | You see | Report | Try (in order) | Why |
|---|---|---|---|---|
| 1 | DVARS spikes that affect the network time course | Temporal diagnosis | `DVARS_censoring=true` in `--frame_censoring` | Censoring runs first, so spikes are handled first. |
| 2 | Slow global drifts in the carpet plot | Temporal diagnosis | `--highpass` (already 0.01); raise detrending order | Removes slow fluctuations. |
| 3 | Spike in the power spectrum at high frequency | Temporal diagnosis | `--lowpass` (already 0.1) | Removes high-frequency noise (e.g. breathing aliasing). |
| 4 | Global signal covariance dominated by noise (edges, CSF) | Spatial diagnosis | 1) ICA-AROMA 2) `aCompCor_5` 3) WM/CSF 4) `mot_24` 5) `global_signal` 6) lowpass 7) `--scale_variance_voxelwise` | Global disturbances are the main driver of bad connectivity estimates. |
| 5 | Some scans fail the confound-correlation threshold | Distribution plot | 1) `aCompCor_5` 2) WM/CSF 3) `global_signal` | Removes shared noise from the network time courses. |
| 6 | Some scans fail the network-specificity (Dice) threshold | Distribution plot | 1) `aCompCor_5` 2) `--scale_variance_voxelwise` | Recovers network contrast to keep more scans. |
| 7 | Group network-variability map not specific | Group report | 1) `aCompCor_5` 2) `mot_24` 3) `--scale_variance_voxelwise` | Group statistics need specific variability. |
| 8 | Significant group correlation with mean FD | Group report | 1) `mot_24` 2) remove outlier scans | Reduces motion-driven group effects. |

How to write the options in the script (only change the lines in `03_confound.sh`):

```bash
--nuisance_regressors WM_signal CSF_signal mot_6            # current
--nuisance_regressors aCompCor_5 mot_6                      # aCompCor instead of mean WM/CSF
--nuisance_regressors WM_signal CSF_signal mot_24           # 24 motion parameters
--nuisance_regressors WM_signal CSF_signal mot_6 global_signal   # GSR (controversial: report it!)
--frame_censoring FD_censoring=true,FD_threshold=0.05,DVARS_censoring=true,minimum_timepoint=480
--scale_variance_voxelwise
--smoothing_filter 0.3                                      # mm FWHM
```

### Notes on specific options

| Option | Advice |
|---|---|
| **ICA-AROMA** | Usually the most effective first addition. But the classifier was adapted on **mouse** data and is not validated in rats. Open `confound_out/confound_correction_datasink/aroma_out/` for a few scans and check that the removed components are really noise. `dim=10` is fixed; if obvious noise remains, try `dim=20`; `dim=0` (automatic) can produce too many components. |
| **Lowpass** | Increases temporal autocorrelation, so confound correlations look higher. Re-check the Conf thresholds after changing it. |
| **mot_24** | Only if the 6 motion parameters show clear motion **and** motion signatures appear in other features. Costs degrees of freedom. |
| **Global signal regression** | Can strongly improve specificity in rodents, but it changes the meaning of correlations (anti-correlations appear). If used, use it for all projects you compare and report it. |
| **Standardize variance** (`--scale_variance_voxelwise`) | Always last: it does not remove confounds, it only reduces the weight of noisy voxels. Use if BOLD<sub>SD</sub> shows confound patterns. |
| **Smoothing** | Currently 0. For voxel-wise group stats (seed/DR + 3dLMEr) light smoothing (0.3–0.45 mm FWHM ≈ 1–1.5 voxels) increases SNR. 07 measures the real smoothness (ACF), so cluster correction stays valid. For NBR/FC matrices smoothing is not needed. |
| **FD threshold** | 0.05 mm is the RABIES rodent default. If many scans drop below 480 volumes, check if a few animals move a lot (exclude them) rather than raising the threshold. |
| **minimum_timepoint** | 480 volumes at TR = 1 s = 8 min. Lower it only if you also check that connectivity does not depend on the number of volumes (group report, temporal DOF). |
| **`--match_number_timepoints`** | Makes all scans have the same number of volumes after censoring; useful when censoring differs between groups. |

## 7. Checklist before moving on

- [ ] Each scan rated specific / unspecific / absent / spurious.
- [ ] Final confound strategy chosen with the one-change-at-a-time loop, written in `confound_strategies.csv`.
- [ ] Dice / Conf thresholds chosen from your own data; excluded scans listed with reasons.
- [ ] Group report: network variability is specific; confound correlations are small compared to the expected effect.
- [ ] If motion or censoring differs between groups: mean FD (and/or remaining volumes) planned as covariates.
