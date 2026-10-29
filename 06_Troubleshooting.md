# 06 – Troubleshooting

[← 05 Figures](05_Figures.md) | [Home](Home.md)

Find your symptom, apply the fix. If it is not here, check the log first:
`rabies/<stage>_out/rabies_<stage>.log`, `slurm-<jobid>.out`, and `crash-*.pklz` files (open with `nipypecli crash <file>` inside the RABIES container).

---

## Slurm / environment

| Symptom | Cause | Fix |
|---|---|---|
| `Cannot find .../resources/nbr_atlas.sh (set FMRI_WPROC_DIR)` | Code not in `/scratch/$USER/neuropsytox_utils/code/fmri_wProc` | `export FMRI_WPROC_DIR=/path/to/fmri_wProc` before `sbatch` |
| `3dClustSim: command not found` | AFNI module not loaded | `module load cobralab` then `module load afni` (two separate commands) |
| RABIES crashes with memory errors | Too many parallel nodes | Use `03_confound_linear.sh`, or lower `--local_threads`, raise `--scale_min_memory` |
| Job killed at time limit | Big dataset | Raise `#SBATCH --time`; for 02, try `FAST_COMMONSPACE=true` |
| `Extraction inputs changed. Use a separate analysis directory` (04-1) | Atlas, confound log or the 04-1 script changed since the cached FC extraction | Rename the old `analysis_out_<atlas>` folder and rerun, so old and new results are not mixed |

## Step 01–02

| Symptom | Cause | Fix |
|---|---|---|
| 01: `ERROR: set ANAT_SOURCE=dbm or ANAT_SOURCE=raw` | First run of the project, no source chosen | add `ANAT_SOURCE=...` before `sbatch` |
| 01/02: `ERROR: ANAT_SOURCE=... but this project uses ...` | Trying to mix anatomical sources | Keep the project source, or delete `data/` + `anat_source.txt` and rerun 01 for everyone |
| 01: `... already has anatomical images but no .../anat_source.txt` | Project arranged with the old scripts | `echo dbm > analysis/fmri/anat_source.txt` (or `raw`), see the project table on page 00 |
| 02: `ERROR: .../anat_source.txt not found` | Same as above | Same as above, or pass `ANAT_SOURCE=` to 02 once |
| 01: `anatomical not found (...pp_lsq6.nii.gz); skipping` | DBM not run for that session | Run the sMRI DBM preprocessing, or use `ANAT_SOURCE=raw` for the whole project |
| 01: `ERROR: averaging failed` (raw) | ANTs averaging of the T2w volumes failed | Check `analysis/fmri/anat_raw_average/preproc/<sub>/<ses>/anat`; check `ANAT_SUFFIX` |
| `WARNING: STC is on but ... has no SliceTiming` | No slice timing in JSON | Confirm `STC_PATTERN`/`STC_AXIS` from the Bruker method file, or `STC=false` |
| `WARNING: TR=... but ... RepetitionTime=...` | TR in 02 differs from the JSON | Check the scanner protocol; set `TR=` correctly (also for 03/04) |
| Registration fails for all scans | Orientation | fix orientation in step 01 (`01_arrange_orientation.tsv`, `REORIENT`) |
| Registration fails for some scans | Masking/inhomogeneity |  decision table (`ROBUST=true` first) |
| Output files huge | Resampling too fine | `RESAMPLING=0.3x0.3x0.3` (default now) |

## Step 03

| Symptom | Cause | Fix |
|---|---|---|
| Scans missing in `confound_out` | Fewer than `minimum_timepoint` volumes after censoring | Check FD of those scans (page 01); exclude or lower the minimum with care |
| Exclusion list ignored | `.txt` contains IDs instead of full paths | One **full path** of the BOLD file per line (page 00 §5) |
| ICA-AROMA fails or removes everything | Few volumes / bad classification | Check `aroma_out/`; try `dim=20`, or disable AROMA and use `aCompCor_5` |

## Step 04

| Symptom | Cause | Fix |
|---|---|---|
| Data diagnosis maps look nothing like rat networks | Mouse default priors were used (old script) | Use 04-1 PASS 2 with your rat `melodic_IC.nii.gz` |
| `--prior_maps must exist and needs both --prior_bold_idx and --prior_confound_idx` | Missing argument | Give all three; indices are 0-based, quoted: `"3 5 9"` |
| 04-3 DR: `Group ICA prior not found` | 04-1 PASS 1 not run | Run 04-1 first, or pass the prior as 4th argument |
| 04-2: antsApplyTransforms cannot read `rois_registered1Warp.nii.gz` | Template→SIGMA registration failed | Check the template path, its `*_mask.nii.gz`, and the ANTs output in the Slurm log |
| 04-2: `... is empty after registration` | Sphere smaller than a voxel after resampling | Increase the radius |
| 04-3 seed: `Seed list not found` | 04-2 not run for this size | Run 04-2 with the same size |
| 04-2: `Group ICA mask not found` | 04-1 PASS 1 not run (it defines the analysis grid) | Run 04-1 first |

## Step 05–07

| Symptom | Cause | Fix |
|---|---|---|
| 06/07: `Analysis mask not found` | 05 not run (or old 05) | Run 05 for that analysis type |
| 06: rows dropped message | Scan without phenotype row, or phenotype without scan | Fix `RID.csv` (`RID`, `Session` must match file names) |
| 06: "Result exists, skipping" | Old result present | Delete/rename `Dual_regression/<Mod>/` to rerun |
| 3dLMEr: `-qVarCenters` / factor errors | Numeric group codes, or variable missing | Use text labels; check column names; remove `-qVarCenters 0` |
| 07: `Residuals not found ... skipping` | Model run with the old 06 (no `-resid`) | Rerun 06 for that model |
| 07: no clusters at all | No effect, or very small N | Check effect maps and N; don't lower PTHR to "find" something |
| 07: huge single cluster covering the brain | Global confound differences between groups | Back to page 02 (group report, mean FD covariate) |
| 08: underlay misaligned | Old 08 used the SIGMA EPI template (different space) | Use the new 08 (InVivo template resampled to the stats grid) |

## NBR

| Symptom | Cause | Fix |
|---|---|---|
| `permutation_valid = FALSE` for a term | Between-subject term with within-subject permutations | Not interpretable; see page 04 |
| Model skipped: fewer than 10 scans | Filters too strict | Check `Models.txt` filter values (case-insensitive, comma-separated) |
| Results not updated after changing data | Cached results with old identity hash | Delete `NBR/<atlas>/results_Mod<id>/` to rerun |
