# Rodent fMRI pipeline – Wiki

This wiki explains **what to run, what to look at, and what to change** when analysing rat resting-state fMRI with the `code/fmri_wProc` scripts (RABIES + group statistics).

It is written as a practical checklist. Every step has:

1. **Run**: the command.
2. **Check**: the QC images or files you must open.
3. **Decide**: a table "if you see X → change Y, because Z".

> Rule of thumb: **never go to the next step until the QC of the current step is OK.** Errors in early steps (orientation, registration) silently ruin every later result.

---

## The pipeline at a glance

```
BIDS ──► 01 arrange ──► 02 preprocess ──► 03 confound correction ──► 04 analysis ──► 05 collect maps ──► 06 group model ──► 07 correction ──► 08/09 figures
          (orient)       (RABIES)          (RABIES)                    (RABIES)         (prepare)           (3dLMEr / NBR)    (clusters/FDR)   (publication)
             │               │                     │                        │
             └── QC page 00  └── QC page 01        └── QC page 02           └── QC page 03
```

| Step | Script(s) | What it does | Wiki page |
|---|---|---|---|
| 0 | – | Organise BIDS, check headers, TR, slice timing | [00 Setup and BIDS](00_Setup_and_BIDS.md) |
| 1 | `01_arrange.sh` (`ANAT_SOURCE=dbm\|raw`) | Copy data, choose the anatomical (DBM or raw BIDS), fix oblique/orientation | [00 Setup and BIDS](00_Setup_and_BIDS.md) |
| 2 | `02_preproc.sh` (`ROBUST`, `FAST_COMMONSPACE`) | RABIES preprocessing (motion, inhomogeneity correction, registration to SIGMA) | [01 Preprocessing QC](01_Preprocessing_QC.md) |
| 3 | `03_confound.sh` | RABIES confound correction (filters, regressors, censoring, AROMA) | [02 Confound correction QC](02_Confound_Correction_QC.md) |
| 4 | `04-1_analysis_ICA_FCmatrix.sh` | Group ICA + FC matrices (+ data diagnosis with rat priors) | [03 Analysis QC](03_Analysis_QC.md) |
| 4 | `04-2_create_ROIs.sh`, `04-3_analysis_SeedBased.sh` | Seed ROIs and seed-based maps | [03 Analysis QC](03_Analysis_QC.md) |
| 4 | `04-3_analysis_DR.sh` | Dual regression with your group ICA | [03 Analysis QC](03_Analysis_QC.md) |
| 5 | `05_r2z_transform_SB_DR.sh` | Collect maps, Fisher-z for seeds, analysis mask | [04 Group stats](04_Group_Stats.md) |
| 6 | `06_3dLMEr.sh` / `06_NBR.sh` | Voxel-wise mixed models / network-based statistics | [04 Group stats](04_Group_Stats.md) |
| 7 | `07_multiple_comparison_correction.sh` | Cluster correction (ACF + 3dClustSim) and FDR | [04 Group stats](04_Group_Stats.md) |
| 8–9 | `08_plotClusterMaps.sh`, `09_explore_plot_results.sh` | Figures | [05 Figures](05_Figures.md) |

Something broke? → [06 Troubleshooting](06_Troubleshooting.md).
What was reviewed and changed in the code? → [Code review and plan](Code_Review_and_Plan.md) and [Changelog](Changelog.md).

---

## Folder layout of a project

```
/scratch/$USER/<project>/
├── Bids/                                  raw BIDS (never modified)
└── analysis/fmri/
    ├── data/                              RABIES input (from 01) + RID.csv, Mod*.txt, Models.txt
    ├── rabies/
    │   ├── preproc_out/                   step 02  → preprocess_QC_report/
    │   ├── confound_out/                  step 03
    │   ├── analysis_out_functional59/     step 04-1 (group ICA, FC matrices)
    │   ├── analysis_diagnosis_out/        step 04-1 pass 2 (data diagnosis, rat priors)
    │   ├── analysis_DR_out/               step 04-3 DR
    │   ├── analysis_SB_<size>mm_out/      step 04-3 seed
    │   ├── Dual_regression_maps/          step 05 (DR)   + analysis_mask.nii.gz
    │   └── Seed_based_zscored_<size>mm/   step 05 (seed) + analysis_mask.nii.gz
    ├── Dual_regression/  Seed_based/      steps 06–08 (models, Mult_comparison*, figures)
    └── NBR/                               06_NBR.sh / 09
```

## Glossary (short)

| Word | Meaning |
|---|---|
| **Commonspace** | The SIGMA rat template space. All group analyses happen here. |
| **FD** | Framewise displacement: how much the head moved between two volumes (mm). |
| **DVARS** | How much the whole-brain signal changes between two volumes. Spikes = artefacts. |
| **Censoring** | Removing bad volumes (time points) from the time series. |
| **Confound / nuisance regressor** | A signal that is not brain activity (motion, WM, CSF) and that we remove. |
| **Prior / ICA component** | A spatial map (network or noise) used as a reference for dual regression. |
| **Dice** | Overlap (0–1) between a scan's network map and the reference network. Higher = network detected. |
| **BOLD<sub>SD</sub>** | Voxel-wise signal variability map. Should look homogeneous in clean data. |
| **ACF** | Spatial smoothness model of the residuals; used by 3dClustSim to find the minimum cluster size. |
| **FWE / FDR** | Two ways to control false positives over many tests. |

## Key references

- RABIES docs: <https://rabies.readthedocs.io/en/latest/> (especially *How to assess data quality*, *How to optimise confound correction*, *How to troubleshoot registration*).
- Desrosiers-Grégoire et al., 2024, *Nat Commun* – the RABIES QC framework used throughout this wiki.
- Grandjean et al., 2020, *NeuroImage* – rodent fMRI standardisation.
- Cox et al., 2017, *Brain Connect.* – AFNI ACF cluster correction.
- Chen et al., 2013/2020 – AFNI linear mixed-effects (3dLME/3dLMEr).
