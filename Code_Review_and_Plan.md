# Code review and improvement plan (`code/fmri_wProc`)

[Home](Home.md) | [Changelog](Changelog.md)

Review date: 2026-10-16. Backup of the code before the fixes: `code/fmri_wProc_backup_20261016/`.

## Overall verdict

The structure is good: RABIES for preprocessing/confound correction/analysis, then AFNI 3dLMEr + cluster correction, plus a well-designed NBR module (provenance hashing, two-stage tests, functional tests). However, several **critical bugs** made results invalid or made scripts fail silently. They were fixed (see table and [Changelog](Changelog.md)).

**Results produced before the fixes for dual regression, data diagnosis and voxel-wise cluster correction must be regenerated.**

---

## 1. Critical issues found and fixed

| # | Script | Problem | Impact | Fix |
|---|---|---|---|---|
| 1 | `04-1` | `--data_diagnosis` ran with RABIES' default **mouse** priors | Data-quality reports (Dice, confound correlation, group QC) meaningless for rats | Two passes: PASS 1 = group ICA + FC; PASS 2 = diagnosis with your rat `melodic_IC` and component indices |
| 2 | `04-1`, `09`, `08` | Script folder found via `BASH_SOURCE`, which points to the Slurm spool copy under `sbatch` | Helper files not found | Fixed path (`FMRI_WPROC_DIR` override) |
| 3 | `05` | Fisher z (atanh) applied to **dual-regression maps** | DR maps are regression coefficients, not correlations; atanh distorts/clips them | DR maps copied unchanged; atanh only for seed r maps |
| 4 | `05`, `06`, `07` | Statistics masked with the SIGMA **EPI** mask, which is a different space/extent from the RABIES commonspace (InVivo) | Wrong voxels included/excluded | `analysis_mask.nii.gz` = group-ICA mask on the maps' grid |
| 5 | `06`, `07` | Smoothness (ACF) estimated from statistic maps; no residuals saved | Wrong cluster-size thresholds | 06 saves 3dLMEr `-resid`; 07 runs `3dFWHMx -acf` on residuals |
| 6 | `07` | Voxel p used in 3dClustSim ≠ p used in 3dClusterize; χ² bricks treated like Z; mod number ignored; cluster CSV parser broken | FWE not controlled at the stated level | Single `PTHR` for both; bi-sided for Z, right-tail for χ²; CSV parser rewritten; tested on synthetic data |
| 7 | `model_template.txt` | `-qVarCenters 0` with `Age*Group` | Group effects tested at Age = 0 | Mean-centring (default). **Project `Mod*.txt` files still need fixing** |
| 8 | `04-2` | Coordinate→voxel conversion used only the diagonal and an undefined variable; header counted as ROI; registration to the wrong template; inverse warp never produced → always failed | Seeds wrong or not created | Full affine conversion, registration to SIGMA InVivo, forward transforms, analysis grid |
| 9 | `04-3_SeedBased` | Hard-coded paths of another user/system (`/misc/austin`); exclusions not passed as paths | Script not portable; exclusions ignored | `/scratch/$USER`, same exclusion handling as 04-1 |
| 10 | `08` | Underlay = SIGMA EPI template (other space); contour level wrong for cluster-ID maps | Misaligned figures | InVivo template resampled to the stats grid; contour 0.5 |
| 11 | `02` | TR hard-coded; resampling from an accidental `printf` of the template voxel size; same STC pattern for every project | Possibly wrong STC; huge/odd resolution | `TR`, `STC`, `STC_PATTERN`, `STC_AXIS`, `RESAMPLING` variables + TR/SliceTiming checks |
| 12 | NBR | NBR permutes within animal → **between-subject terms (Group, Sex, …) have invalid p-values** | False claims possible | `09` flags terms (`permutation_valid`), no figures for invalid terms. Hashed NBR scripts untouched (results stay valid) |
| 13 | `06` | All ~30 DR components modelled (mostly noise) | Many useless tests | `DR_COMPONENTS` selects BOLD components |
| 14 | `07` | Cluster alpha fixed at 0.05 per map | No way to correct across maps | `ALPHA` variable (e.g. Bonferroni) |

## 2. Remaining issues (not changed yet) and plan

### Phase 1 – correctness and consistency (next)

| Priority | Item | Suggested action |
|---|---|---|
| High | Project model files (`Dominance_project`, `sudmex_alcohol_rat` `data/Mod1.txt`) use `-qVarCenters 0`; alcohol uses `(1|Batch)` with few batches | Edit them (remove `-qVarCenters 0`, Batch as fixed effect), rerun 06–07 |
| High | NBR cannot test between-subject effects | Add an NBR mode that permutes labels **between** animals (keeping each animal's scans together; `permute_between_id` already exists in `network_utils.R`), as a new script so existing hashed results remain valid |
| High | ICA-AROMA classifier is mouse-adapted, not validated in rats | For each project, inspect `aroma_out/` on ~10 scans; compare strategies with/without AROMA in the data diagnosis (page 02) |
| Done | Two different `01` scripts; `fmri_woProc` duplicated 01–04-1 only to allow RABIES anatomical bias correction | Merged: `01_arrange.sh` with `ANAT_SOURCE`, `DEOBLIQUE`, `REORIENT`; `02_preproc.sh` with `ROBUST`, `FAST_COMMONSPACE` (see [Changelog](Changelog.md)). Still to do: automatic orientation QC image |
| Medium | Exclusion `.txt` must contain full paths, IDs only work on the command line | Accept IDs in the `.txt` too (resolve them like the comma list) |
| Medium | `03` writes always to `confound_out` | Add `CONFOUND_TAG` to write `confound_out_<tag>` so strategies can be compared side by side |
| Medium | `Seed_based/fc_csv` is shared by all seed sizes | Use `Seed_based_<size>mm/` for models too |
| Medium | 3dLMEr jobs request 192 tasks but use `-jobs 20` | Match `--ntasks`/`-jobs` (e.g. 32/32) to queue faster |
| Medium | 08 plots dual regression only; voxel size 0.3 hard-coded in plots | Add seed mode; read voxel size from the header |
| Low | `03` loads unneeded modules (FreeSurfer) | Clean module loads |
| Low | `fc_dataset.csv` written with `quote=FALSE` (fails if a value contains a comma) | Write with quotes (requires bumping NBR identity) |
| Done | `fmri_woProc` (old version) still present | Archived in `code/archive/fmri_woProc` |

### Phase 2 – standardisation

1. **One project config file** (`analysis/fmri/data/project.conf`) read by every script: TR, STC settings, resampling, exclusion file, confound tag, DR components, PTHR/ALPHA. Removes copy-paste of parameters across scripts.
2. **One QC tracker** per project (`QC_tracker.csv`: scan, step, rating, reason) filled during pages 01–03; scripts read it to build exclusions.
3. **Automatic QC summary** after each step (HTML page with all PNGs + FD table), so QC is quick.
4. **Provenance**: write RABIES/container versions, git hash of the code and all parameters to `analysis/fmri/provenance.txt` at every run.
5. Put `neuropsytox_utils` under **git**, tag versions used for each paper.
6. Extend the synthetic tests (like `NBR/tests/functional_atlas.R`) to 05–07.

### Phase 3 – new analyses (extensions)

| Analysis | What it adds | Tool |
|---|---|---|
| ALFF / fALFF | Local amplitude of low-frequency fluctuations | AFNI `3dRSFC` on confound-corrected data |
| ReHo | Local synchrony | AFNI `3dReHo` |
| Graph theory | Network topology (efficiency, modularity, hubs) from FC matrices | R `igraph` / `brainGraph` |
| NBR between-subject | Valid group main effects on edges | Phase 1 item |
| Dynamic FC / CAPs | Time-varying connectivity states | sliding window or CAPs in Python |
| Alternative voxel-wise inference | TFCE / non-parametric permutation | FSL `randomise`, AFNI `3dttest++ -ETAC` |
| Batch harmonisation | Multi-batch/multi-site data | neuroCombat (ComBat) |
| Prediction | Brain–behaviour prediction | CPM / cross-validated regression |

Add each new analysis as `04-x` (map generation) + reuse 05–08 for statistics/plots, and add its QC section to this wiki.
