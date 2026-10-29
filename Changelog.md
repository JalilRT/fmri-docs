# Changelog (`code/fmri_wProc`)

[Home](Home.md) | [Code review and plan](Code_Review_and_Plan.md)

## 2026-10-28 - preprocessing exclusions and voxel-spacing choices

Shared `fmri_wProc/02_preproc.sh` now accepts `[exclusion_input] [suffix]` after the project argument. Comma-separated subject/session IDs or a text file containing IDs or staged BOLD paths are resolved to container-visible paths. All matching runs are excluded, duplicates are removed, and unmatched entries fail explicitly. Blank/comment lines and CRLF text files are supported. Exclusions are saved with the run settings.

`RESAMPLING=template` reads spatial voxel spacing from SIGMA (rat) or bundled DSURQE (mouse) without rounding, using ANTs in the container. `RESAMPLING=inputs_defined` delegates to RABIES' functional-input-based resolution inference. Explicit voxel sizes and the existing `0.3x0.3x0.3` wrapper default remain unchanged. Template-native spacing is optional, not recommended automatically for low-resolution EPI.

## 2026-10-26 – adolescence_thc slice timing

The project-specific adolescence_thc preprocessing now enables STC for the confirmed 26-slice interleaved order (`0,2,...,24,1,3,...,25`). Settings are `alt+z`, RAS axis `Y`, and reference time `0` seconds. The RABIES 0.6.1 command uses `--apply_STC --stc_axis Y --args_3dTshift="-Fourier -tpattern alt+z -tzero 0"`; it does not accept `--slice_timing_args` JSON. Pending no-STC job 2656950 was cancelled before preprocessing started and replaced with job 2658666.

## 2026-10-16 – merge of `fmri_woProc` (anatomical source)

`fmri_woProc` (steps 01–04-1 with RABIES anatomical inhomogeneity correction) is merged into `fmri_wProc` and archived in `code/archive/fmri_woProc/`. Replaced `fmri_wProc` scripts are kept in `code/archive/fmri_wProc_replaced_20261016/`.

| Script | Change |
|---|---|
| `01_arrange.sh` | Single step-01 script. `ANAT_SOURCE=dbm\|raw` (DBM `pp_lsq6` or BIDS anatomical, averaged if 2+ volumes; an existing sMRI `average.nii.gz` is reused); the source is stored in `analysis/fmri/anat_source.txt` and mixing is blocked. `DEOBLIQUE=refit\|warp`, `REORIENT=false\|true` (`FUNC_ORIENT_FROM`/`FUNC_ORIENT_TO`), `ANAT_SUFFIX`. Sessions `ses-*` (not only `ses-T*`). Writes `logs/01_arrange_orientation.tsv` |
| `01_deoblique.sh` | Removed → `DEOBLIQUE=warp REORIENT=true` |
| `02_preproc.sh` | `--anat_inho_cor` chosen from the anatomical source (`dbm` → disable, `raw` → SyN + multiotsu). New `ROBUST`, `FAST_COMMONSPACE`, `ANAT_INHO_COR`, `BOLD2ANAT_COREG`. Settings saved in `logs/02_preproc_settings.txt`. `MPLCONFIGDIR` set per job |
| `02_preproc_fastc.sh` | Removed → `FAST_COMMONSPACE=true` |
| `fmri_woProc/02_preproc_robust.sh` | Archived → `ROBUST=true` (+ `BOLD2ANAT_COREG=masking=true,brain_extraction=true` to reproduce it exactly) |

**Existing projects:** before rerunning 02, create `analysis/fmri/anat_source.txt` (see the project table on [page 00](00_Setup_and_BIDS.md)).

New environment variables: `ANAT_SOURCE`, `ANAT_SUFFIX`, `DEOBLIQUE`, `REORIENT`, `FUNC_ORIENT_FROM`, `FUNC_ORIENT_TO` (01); `ROBUST`, `FAST_COMMONSPACE`, `ANAT_INHO_COR`, `BOLD2ANAT_COREG` (02). New files/folders: `analysis/fmri/anat_source.txt`, `analysis/fmri/anat_raw_average/`, `analysis/fmri/logs/`.

## 2026-10-16 – review fixes

Backup of the previous version: `code/fmri_wProc_backup_20261016/`.

### Action required for existing projects

1. **Dual regression**: rerun `05` → `06` → `07` → `08`. The old `Dual_regression_zscored/` (Fisher z applied to DR maps) is **obsolete**; do not use it.
2. **Data diagnosis**: rerun `04-1` so PASS 2 runs the diagnosis with rat priors (old reports used mouse priors).
3. **Model files**: remove `-qVarCenters 0` from project `Mod*.txt` files unless you really want effects at covariate = 0.
4. **TR/STC**: check `TR`, `STC`, `STC_PATTERN`, `STC_AXIS` for each project before running `02` (see [00_Setup_and_BIDS](00_Setup_and_BIDS.md)).
5. **NBR**: ignore p-values of terms flagged `permutation_valid = FALSE` (between-subject terms).

### Changes per script

| Script | Change |
|---|---|
| `02_preproc.sh`, `02_preproc_fastc.sh` (later merged, see above) | `TR`, `STC`, `STC_PATTERN`, `STC_AXIS`, `RESAMPLING` env vars; TR compared with the BIDS JSON (handles ms and lists); warning when `SliceTiming` is missing; STC options only added when `STC=true`; fixed usage paths |
| `03_confound.sh`, `03_confound_linear.sh` | `TR` overridable; usage fixed |
| `04-1_analysis_ICA_FCmatrix.sh` | Two passes (PASS 1 group ICA + FC; PASS 2 data diagnosis with rat `melodic_IC` → `analysis_diagnosis_out/`); `FMRI_WPROC_DIR`; `SCAN_QC_THRESHOLDS`; `TR` overridable; content hash for FC provenance |
| `04-2_create_ROIs.sh` | Full-affine coordinate→voxel conversion; header row skipped; registration to SIGMA InVivo with forward transforms; ROIs resampled to the analysis grid; usage fixed |
| `04-3_analysis_DR.sh` | Rat priors; `TR` overridable |
| `04-3_analysis_SeedBased.sh` | Removed `/misc/austin` paths; exclusion handling as in 04-1; `TR` overridable |
| `05_r2z_transform_SB_DR.sh` | DR maps copied without Fisher z → `Dual_regression_maps/`; seed r maps → `Seed_based_zscored_<size>mm/`; `analysis_mask.nii.gz` built from the group-ICA mask on the maps' grid |
| `resources/arranging_dataset_DR.R`, new `resources/arranging_dataset_SB.R` | Use the new map folders; seed-based table builder |
| `model_template.txt` | Mean-centred covariates (no `-qVarCenters 0`), documented options |
| `06_3dLMEr.sh` | `maps_dir`/`analysis_mask`; saves `-resid`; `DR_COMPONENTS` filter; old `fc_dataset_*.txt` removed before regeneration |
| `07_multiple_comparison_correction.sh` | ACF from residuals; one `PTHR` for ClustSim and Clusterize; bi-sided for Z, right-tail for χ²; `ALPHA` (3dClustSim `-athr`); fixed cluster CSV parser; `thresholds_used.txt`; outputs in `Mult_comparison/` (DR) or `Mult_comparison_<size>mm/` (seed) |
| `08_plotClusterMaps.sh` + `.R` | InVivo underlay resampled to the stats grid (`figures_underlay/`); contour level 0.5 |
| `09_explore_plot_results.sh` | `FMRI_WPROC_DIR` for helper files |
| `NBR/explore_plot_results.R` | `term_permutation_valid()`; `permutation_valid` column in the FWE summary; no figures for invalid terms. `nbr_run_single_model.R` and `network_utils.R` unchanged (identity hashes preserved) |

### New environment variables

| Variable | Default | Used by | Meaning |
|---|---|---|---|
| `FMRI_WPROC_DIR` | `/scratch/$USER/neuropsytox_utils/code/fmri_wProc` | 04-1, 08, 09 | Location of the scripts (needed under `sbatch`) |
| `TR` | `1.0` | 02, 03, 04-1, 04-3 | Repetition time (s) |
| `STC` | `true` | 02 | Apply slice-timing correction |
| `STC_PATTERN` | `seq-z` | 02 | 3dTshift `-tpattern` |
| `STC_AXIS` | `Y` | 02 | Slice axis for RABIES `--stc_axis` |
| `RESAMPLING` | `0.3x0.3x0.3` | 02 | `--commonspace_resampling` (mm) |
| `SCAN_QC_THRESHOLDS` | unset | 04-1 (PASS 2) | RABIES `--scan_QC_thresholds` |
| `DR_COMPONENTS` | unset (all, with warning) | 06 | 0-based BOLD component indices, e.g. `"5,9,12"` |
| `PTHR` | `0.001` | 07 | Voxel-wise p threshold |
| `ALPHA` | `0.05` | 07 | Cluster-level FWE alpha per map (e.g. `0.05/N`) |

### New / renamed output folders

| Folder | Content |
|---|---|
| `analysis_diagnosis_out/` | RABIES data diagnosis with rat priors (04-1 PASS 2) |
| `Dual_regression_maps/` | DR maps for statistics (replaces `Dual_regression_zscored/`) |
| `Seed_based_zscored_<size>mm/` | Fisher-z seed maps per seed size |
| `figures_underlay/` | Underlay on the statistics grid (08) |
| `Mult_comparison/`, `Mult_comparison_<size>mm/` | Cluster-corrected results (07): DR, seed of a given size |
