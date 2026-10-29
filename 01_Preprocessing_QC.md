# 01 – Preprocessing (RABIES `preprocess`) and its QC

[← 00 Setup](00_Setup_and_BIDS.md) | [Home](Home.md) | Next: [02 Confound correction QC →](02_Confound_Correction_QC.md)

Goal: every scan is **motion-corrected and correctly registered to the SIGMA template**. If registration is wrong, the voxel you analyse is not the brain region you think it is.

---

## 1. Run

```bash
# defaults: TR=1.0 STC=true STC_PATTERN=seq-z STC_AXIS=Y RESAMPLING=0.3x0.3x0.3 (see page 00)
sbatch neuropsytox_utils/code/fmri_wProc/02_preproc.sh <project>
# options can be combined, e.g. robust masking + direct anat→atlas registration:
ROBUST=true FAST_COMMONSPACE=true sbatch neuropsytox_utils/code/fmri_wProc/02_preproc.sh <project>
# exclude sessions (all matching task-rest runs), or use a text file:
sbatch neuropsytox_utils/code/fmri_wProc/02_preproc.sh <project> "sub-033_ses-2,sub-040_ses-2"
sbatch neuropsytox_utils/code/fmri_wProc/02_preproc.sh <project> /absolute/path/exclusions.txt
```

The optional second argument is `exclusion_input`: comma-separated subject/session IDs, or a `.txt` file with one ID or full staged BOLD path per line. Blank lines and lines beginning with `#` are ignored. The optional third argument selects the filename suffix (default `task-rest`), e.g. `<project> "sub-033_ses-2" task-other`. IDs exclude all matching runs, not just the first one. Full paths must point inside the staged input dataset; `/data_input/...` paths and exact BOLD filenames are also accepted. Unmatched entries stop the script with an error. Resolved exclusions are logged and passed to RABIES before preprocessing; the BIDS originals are not deleted. If every BOLD in a session is excluded, its anatomical does not enter the new study templates. Use a fresh preprocessing output directory when changing the included dataset or preprocessing strategy; do not assume cached templates will be rebuilt correctly.

02 reads the anatomical source written by 01 (`analysis/fmri/anat_source.txt`) and chooses the anatomical bias-field correction automatically:

| Source | `--anat_inho_cor` | Why |
|---|---|---|
| `dbm` | `method=disable` | The DBM `pp_lsq6` image is already bias-corrected and brain-aligned |
| `raw` | `method=SyN,otsu_thresh=2,multiotsu=true` | Raw T2w has a strong coil bias field; SyN masking + multi-Otsu handles the bright/dark gradient of surface coils |

Options (environment variables):

| Variable | Default | What it adds | When to use |
|---|---|---|---|
| `SPECIES` | `rat` | `rat`: SIGMA template/masks; `mouse`: RABIES bundled DSURQE mouse template/masks | Confirm species before starting; never register mouse data to SIGMA |
| `FMRI_INPUT_DIR` | `analysis/fmri/data` | Existing absolute path to a separately staged BIDS dataset | Keep original inputs untouched when selecting a different anatomical acquisition |
| `RESAMPLING` | `0.3x0.3x0.3` | Explicit positive voxel sizes in mm; `template` reads the selected template's spatial spacing; `inputs_defined` delegates to RABIES | Prefer a justified analysis resolution near acquired EPI resolution; native atlas spacing is usually unnecessarily fine |
| `FAST_COMMONSPACE` | `false` | `--commonspace_reg masking=true,brain_extraction=false,template_registration=SyN,fast_commonspace=true`: each anatomical → SIGMA directly, no unbiased template | Quick tests, very many scans, or if the unbiased template fails |
| `ROBUST` | `false` | `--bold_robust_inho_cor apply=true` (and `--anat_robust_inho_cor apply=true` for `raw`), plus `--bold2anat_coreg masking=true,brain_extraction=false` | Brain masks in `bold_inho_cor/` (or `anat_inho_cor/`) fail for several scans. About 2× slower |
| `ANAT_INHO_COR` | from source | Overrides `--anat_inho_cor` | E.g. `method=Affine,...` if SyN masking fails on raw anatomicals |
| `BOLD2ANAT_COREG` | RABIES default (`registration=SyN`, no masking) | Overrides `--bold2anat_coreg` | E.g. `masking=true,brain_extraction=true` (the old robust script), or `registration=Rigid` |

The old `02_preproc_fastc.sh` = `FAST_COMMONSPACE=true`; the old `fmri_woProc/02_preproc_robust.sh` ≈ `ANAT_SOURCE=raw ROBUST=true BOLD2ANAT_COREG=masking=true,brain_extraction=true`. The settings of every run are saved to `analysis/fmri/logs/02_preproc_settings.txt` – copy them into your methods.

For rats, the reference is SIGMA InVivo and the CSF mask is reused as `--vascular_mask` (no rat vascular mask exists; it only feeds QC and optional regressors). For mice, `SPECIES=mouse` uses the installed RABIES DSURQE reference, tissue masks and vascular mask instead. Choose `RESAMPLING` for the acquisition; the rat default is not automatically changed for mice.

### Choosing output voxel size

- **Explicit analysis grid (existing default):** `RESAMPLING=0.3x0.3x0.3`. This is independent of the atlas's native voxel size.
- **Atlas-native spacing:** `RESAMPLING=template` reads SIGMA for rats or bundled DSURQE for mice using ANTs `PrintHeader` inside the container. Spatial spacing is retained without decimal rounding (DSURQE is approximately `0.04x0.04x0.04` mm). This matches voxel spacing, not necessarily the atlas's exact grid origin or dimensions. It can greatly increase runtime and disk use without recovering finer acquired detail.
- **RABIES automatic spacing:** `RESAMPLING=inputs_defined` uses the functional input spacing, not the anatomical atlas spacing. RABIES 0.6.1 requires consistent input spacing; otherwise it stops and asks for explicit dimensions. This is RABIES' own default for both species, but this wrapper's unchanged `0.3x0.3x0.3` default overrides it.

For example:

```bash
sbatch --export=ALL,RESAMPLING=inputs_defined neuropsytox_utils/code/fmri_wProc/02_preproc.sh <project>
sbatch --export=ALL,SPECIES=mouse,RESAMPLING=template neuropsytox_utils/code/fmri_wProc/02_preproc.sh <project>
```

The requested and resolved resampling settings and exclusions are saved in `logs/02_preproc_settings.txt`. Template spacing is separate from `--anatomical_resampling`, which controls registration-target resolution.

For example, mouse FLASH T1 inputs must retain the `T1w` suffix (do not rename T1 images as T2w):

```bash
sbatch --export=ALL,ANAT_SOURCE=raw,SPECIES=mouse,STC=false,TR=1.0,RESAMPLING=0.25x0.25x0.25,FMRI_INPUT_DIR=/scratch/$USER/<project>/analysis/fmri/data \
  neuropsytox_utils/code/fmri_wProc/02_preproc.sh <project>
```

This example disables slice-timing correction because the acquisition order is unconfirmed. Inspect anatomical masking, EPI-to-T1 registration and template alignment carefully before confound correction. The later rat-specific atlas/ROI scripts are not automatically converted to mouse analysis by this option. adolescence_thc (mouse) uses its own copies of steps 01–03 with these settings fixed: see `adolescence_thc/code/project_params.sh`.

For adolescence_thc, the confirmed acquisition order is `0,2,...,24,1,3,...,25` (26 slices). Its project settings now enable STC with `STC_PATTERN=alt+z`, `STC_AXIS=Y`, and `STC_TZERO=0`. RABIES 0.6.1 accepts `--apply_STC --stc_axis Y --args_3dTshift="-Fourier -tpattern alt+z -tzero 0"`, not `--slice_timing_args` JSON. The original LSA slice axis becomes Y after RABIES reorients to RAS; the `z` in AFNI's pattern refers to the temporary axis used internally by RABIES. `-tzero 0` aligns slices to the start of the TR, rather than AFNI's default mean slice time. This pattern assumes increasing stored slice indices match the reported acquisition order and evenly spaced slice acquisitions.

**Mouse leaving the field of view crashes RABIES.** If `gen_ref` or `motion_correction` fails with `All samples map outside moving image buffer`, the BOLD contains volumes without the brain. Plot the mean brain signal per volume, then crop the scan to its stable part or exclude it, and rerun 02. adolescence_thc's `01_stage.py` does this screen automatically before RABIES.

**Cluster export warning:** some sites wrap `sbatch` with `--export=NONE`. In that case, environment assignments before `sbatch` are lost. Use an explicit `--export=ALL,...` as above (or `--export=ALL` after setting the variables) and verify the printed settings in the job log before trusting the run.

The log is `rabies/preproc_out/rabies_preprocess.log`; check the end says the workflow finished without failed nodes.

## 2. Check – where are the QC images?

`<project>/analysis/fmri/rabies/preproc_out/preprocess_QC_report/`

| Folder | What it shows | What "good" looks like |
|---|---|---|
| `template_files/` | SIGMA template with the brain, WM, CSF and vascular masks | Masks sit on the right tissue. Check once per project. |
| `anat_inho_cor/` | (`raw` source only) each anatomical before/after bias correction + brain mask | Even intensity across the brain; mask follows the brain edge |
| `anat_robust_inho_cor_template.*`, `bold_robust_inho_cor_template.*` | (`ROBUST=true`) the temporary templates used for robust masking | Sharp, brain-shaped average; registered to SIGMA |
| `bold_inho_cor/` | Mean EPI before/after bias-field correction + automatic brain mask | Even intensity, mask covers the brain only. |
| `commonspace_reg_wf.Anat2Unbiased/` | Each anatomical → unbiased template (02_preproc.sh) | Edges match; no stretched or rotated brains. |
| `commonspace_reg_wf.Unbiased2Atlas/` | Unbiased template → SIGMA | Ventricles, corpus callosum, cortex edges match the atlas. |
| `commonspace_reg_wf.Anat2Atlas/` | (fast_commonspace) each anatomical → SIGMA | Same as above, for each scan. |
| `EPI2Anat/` | Each mean EPI → its own anatomical | Brain outline and ventricles of the EPI match the anatomical. Some distortion at the edges is normal for EPI. |
| `temporal_features/` | Per-scan motion, FD, temporal SD and SNR maps | Low FD, temporal SD highest in brain (not at the edges or outside). |

Motion files: `preproc_out/motion_datasink/FD_csv/` (one CSV per scan; columns are `Mean,Max` or `Timepoint,MeanFD,MaxFD` depending on the RABIES version).

Quick FD table for the whole project (run in `preproc_out/motion_datasink`, after `module load cobralab`):

```r
fs <- list.files("FD_csv", "_FD_file.csv$", recursive = TRUE, full.names = TRUE)
d <- do.call(rbind, lapply(fs, function(f) { x <- read.csv(f)
  fd <- if ("MeanFD" %in% names(x)) x$MeanFD else x$Mean
  data.frame(scan = sub("_bold.*_FD_file.csv", "", basename(f)),
             meanFD = mean(fd), pct_over_0.05 = mean(fd > 0.05) * 100) }))
d[order(-d$meanFD), ]                 # worst scans first
write.csv(d, "FD_summary.csv", row.names = FALSE)
```

## 3. How to check registration fast

1. Open all PNGs of a folder as a slideshow (e.g. `eog` or copy the folder locally). Look only at **edges and ventricles**.
2. Mark every scan as **OK / borderline / fail** in a spreadsheet (`preproc_QC.csv` with columns `scan, step, rating, comment`).
3. Look at failing scans in 3D: `preproc_out/bold_datasink/commonspace_bold/...` over the SIGMA template (fsleyes).

## 4. Decide – what to change

Fix problems **in this order**: inhomogeneity/masking → anatomical-to-template → EPI-to-anatomical. Each step feeds the next one.

| You see | Likely cause | Change in `02_preproc.sh` | Why |
|---|---|---|---|
| Everything is rotated/flipped | Orientation in headers | Go back to page 00 and fix step 01 | Registration can't fix orientation. |
| (`raw`) `anat_inho_cor/` masks cut the brain or include muscle | SyN masking of the raw T2w failed | First `ROBUST=true`; if still bad, `ANAT_INHO_COR=method=Affine,otsu_thresh=2,multiotsu=true`; if many fail, switch the project to `dbm` (run the DBM pipeline) | The robust template is a better masking target; the DBM pipeline has its own QC'd bias correction. |
| (`raw`) dark band / bright surface remains after correction | Coil bias too strong | Keep `multiotsu=true`; try `otsu_thresh=3` in `ANAT_INHO_COR` | More Otsu classes separate the gradient better. |
| (`dbm`) anatomical looks wrong (cropped, wrong contrast) | DBM preprocessing problem | Fix it in the sMRI pipeline (`*_pp_lsq6.jpg`), rerun 01 for that subject | 01 copies the DBM file as is. |
| Brain mask in `bold_inho_cor/` cuts brain or includes muscle in a few scans | Bias field / masking failed | Rerun with `ROBUST=true` | Builds a temporary template of all scans as a more robust masking target. |
| Strong intensity drop (e.g. ventral brain dark) remains | Strong coil inhomogeneity | Add `multiotsu=true` to `--bold_inho_cor` | Corrects low intensities first, then higher ones. |
| Lots of non-brain tissue (muscle, eyes) drives registration | Large FOV | Add `--bold_autobox` (and/or `--anat_autobox`) | Crops extra tissue automatically. |
| Many masks still fail | Template-to-EPI masking registration fails | `--bold_inho_cor` default is `method=Rigid`; if you raised it, step back down (`SyN → Affine → Rigid`). If Rigid fails too, use `method=no_reg` and tune `otsu_thresh` | Less flexible registration is more robust for low-contrast EPI. |
| Anatomical → template misregistered, especially at brain edges | Non-brain tissue / poor masks | `--commonspace_reg masking=true,brain_extraction=true,keep_mask_after_extract=true` (only if masks are good) | Constrains edges to the brain. |
| Cerebellum or olfactory bulb missing from the FOV and surrounding tissue stretched | Incomplete coverage | `brain_extraction=true,keep_mask_after_extract=true` in `--commonspace_reg` | Stops SyN from stretching tissue into missing areas. |
| Unbiased template is blurry or distorted | Some very bad anatomicals | Exclude those scans or use `FAST_COMMONSPACE=true` | A few bad scans corrupt the average template. |
| EPI → anatomical bad for many scans | EPI distortion / low contrast | Try `--bold2anat_coreg registration=Rigid` (or `Affine`) and check `masking=true` | Simpler transforms are more stable for EPI. |
| EPI → anatomical bad for one scan only | Bad acquisition | Exclude the scan (page 00 §5) | Do not tune the whole pipeline for one scan. |
| Large FD (mean FD > ~0.1 mm or many spikes > 0.05 mm) | Animal motion / anaesthesia | Nothing in 02. Handle with censoring in 03 or exclude | Motion is corrected in 03, not here. |
| Temporal SD bright at brain edges / outside brain | Motion or ghosting | Keep in mind for 03 (motion regressors, AROMA) | Edge variance is a motion signature. |
| Temporal SD bright in ventricles / around vessels | Physiological noise | Keep in mind for 03 (CSF regressor, aCompCor) | – |

After any change: rerun 02 in a **new output folder** (or delete `preproc_out`), and repeat the QC.

## 5. Exclusion rule (write it down before looking at group results)

A scan is excluded at this step if:

- registration fails after the fixes above, **or**
- the EPI does not cover the brain regions of your hypothesis, **or**
- motion is extreme (decide the number before seeing group results, e.g. mean FD > 0.1 mm, or < 480 volumes would remain after censoring).

Add the scan to the exclusion file with the reason.

## 6. Checklist before moving on

- [ ] All registrations rated OK (or the scan is excluded with a reason).
- [ ] `FD_summary.csv` saved; motion does not differ strongly between groups (check with a quick t-test/LMM on mean FD; if it differs, add mean FD as a covariate in step 06).
- [ ] TR, STC and resampling settings written in the project README.
