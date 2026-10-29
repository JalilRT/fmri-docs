# 00 – Setup, BIDS and acquisition parameters

[← Home](Home.md) | Next: [01 Preprocessing QC →](01_Preprocessing_QC.md)

Goal: make sure the input data are **correct, complete and correctly described** before RABIES touches them. Most "strange" results later come from mistakes here.

---

## 1. Requirements

| Item | Where | Notes |
|---|---|---|
| BIDS data | `/scratch/$USER/<project>/Bids/` | `sub-XXX/ses-YY/{anat,func}` |
| sMRI (DBM) preprocessing done – **only for `ANAT_SOURCE=dbm`** | `<project>/analysis/smri/DBM/preproc/` | `01_arrange.sh` takes the anatomical `*_pp_lsq6.nii.gz` from there (see §3) |
| Containers | `neuropsytox_utils/containers` (`rabies.sif`, `afni.sif`, `afni_3dlmer.sif`) | |
| Atlas | SIGMA v1.2.1 (InVivo template, masks, functional59 parcellation) | used by 02, 04, 08 |
| Modules | `module load cobralab` (R, FSL, minc, ANTs helpers) and `module load afni` | |
| Phenotype table | `<project>/analysis/fmri/data/RID.csv` | Needed from step 06 on |

If you run scripts from a copy of the code that is not in `/scratch/$USER/neuropsytox_utils/code/fmri_wProc`, set `export FMRI_WPROC_DIR=/path/to/fmri_wProc` before `sbatch` (under Slurm the script can't find its own folder).

## 2. Check every BIDS header (5 minutes that save weeks)

```bash
module load afni
for f in /scratch/$USER/<project>/Bids/sub-*/ses-*/func/*_bold.nii.gz; do
  echo "$f  $(3dinfo -orient -ad3 -nt -tr -is_oblique $f)"
done
```

| Check | Expected | If not |
|---|---|---|
| Same voxel size and number of volumes for all scans | identical | Find out why (different protocol?). Exclude or analyse separately. |
| TR (`-tr`) | Same as the scanner protocol (e.g. 1.0 s) | If the header TR is wrong, fix the header (`3drefit -TR`) **or** set `TR=` when running 02 and 03. |
| Number of volumes | ≥ what you need after censoring (03 requires `minimum_timepoint=480`) | Shorter scans will be dropped by 03 → lower the minimum or exclude. |
| Orientation | Same for all scans | Run `01_arrange.sh` (see below). |

Also open the JSON sidecar:

| Field | Why it matters |
|---|---|
| `RepetitionTime` | Must match the TR given to RABIES. 02 now warns if they differ. Bruker exports sometimes store it in **ms** or as a **list** (Dominance project: `[1000.0]`). |
| `SliceTiming` | Needed to choose the slice-timing correction (STC) pattern. Often **missing or wrong** in Bruker → BIDS conversions (sudmex_alcohol JSON: interleaved values spanning only ~1.6 ms, which is impossible for a 1 s TR). |

## 3. Run step 01 (copy + anatomical source + orientation)

### 3a. Choose the anatomical image: `dbm` or `raw`

RABIES needs one anatomical image per session. You can give it two kinds of image:

| `ANAT_SOURCE` | Anatomical used | Bias-field (inhomogeneity) correction | Use it when |
|---|---|---|---|
| `dbm` | `smri/DBM/preproc/<sub>/<ses>/anat/*_pp_lsq6.nii.gz` from the sMRI/DBM pipeline | Already done by the DBM pipeline → 02 uses `--anat_inho_cor method=disable` | You already ran the DBM pipeline (recommended: one shared anatomical preprocessing for sMRI and fMRI, already QC'd) |
| `raw` | BIDS anatomical `<sub>_<ses>_${ANAT_SUFFIX}.nii.gz` (default suffix `run-02_T2w`). If it has 2+ volumes it is averaged with the same method as `smri/01_average.sh` (an existing sMRI `average.nii.gz` is reused) | Done by RABIES → 02 uses `--anat_inho_cor method=SyN,otsu_thresh=2,multiotsu=true` | No DBM run (yet), or the DBM preprocessing failed/looks bad for this project |

Rules:

- **One source per project.** 01 writes it to `analysis/fmri/anat_source.txt`; 01 and 02 stop if you try to mix sources. Mixing would make some animals' registration differ from others' for technical reasons only.
- Never apply bias correction twice on purpose: with `dbm`, keep `method=disable`. (It is not harmful, only slower – Dominance_project was run that way.)
- To switch source: delete `analysis/fmri/data` and `anat_source.txt`, rerun 01 for all subjects, then rerun 02 in a clean `preproc_out`.

### 3b. Obliquity and orientation options

| Variable | Default | Options | Meaning |
|---|---|---|---|
| `DEOBLIQUE` | `refit` | `refit`, `warp` | `refit` = `3drefit -deoblique` (rewrites the header, no resampling). `warp` = `3dWarp -oblique2card` (resamples to a cardinal grid; slight blurring) |
| `REORIENT` | `false` | `true`, `false` | `true` resamples the BOLD to `FUNC_ORIENT_FROM` (`LPS`) and relabels it `FUNC_ORIENT_TO` (`RSA`). This is a **header relabel that flips axes**, only for Bruker data whose BOLD header is wrong (animal in sphinx position). Use it only if the orientation QC below shows the BOLD flipped |
| `ANAT_SUFFIX` | `run-02_T2w` | any BIDS suffix | Only for `raw` |

The old `01_deoblique.sh` = `DEOBLIQUE=warp REORIENT=true`. The old `01_arrange.sh` = the defaults.

### 3c. Run

```bash
# DBM anatomicals (loop over the subjects that have DBM preprocessing)
for s in $(ls -d /scratch/$USER/<project>/analysis/smri/DBM/preproc/sub-*); do
  ANAT_SOURCE=dbm sbatch neuropsytox_utils/code/fmri_wProc/01_arrange.sh <project> $(basename $s)
done

# raw BIDS anatomicals
for s in $(ls -d /scratch/$USER/<project>/Bids/sub-*); do
  ANAT_SOURCE=raw sbatch neuropsytox_utils/code/fmri_wProc/01_arrange.sh <project> $(basename $s)
done
```

Submit **one subject first**, check its orientation (below), then submit the rest with the same settings. Every run appends a line to `analysis/fmri/logs/01_arrange_orientation.tsv` (`subject, session, source, anat orientation, BOLD orientation, DEOBLIQUE, REORIENT`). All rows should have the same settings, and the same orientation within each column.

What existing projects used (from their files and RABIES logs, 2026-10-16):

| Project | Anatomical in RABIES | 02 inhomogeneity options | BOLD orientation | Equivalent new settings |
|---|---|---|---|---|
| Dominance_project | DBM `pp_lsq6` | `anat_inho_cor=SyN` + robust (old `fmri_woProc/02_preproc_robust.sh`) | RSA | `ANAT_SOURCE=dbm` + 02 with `ROBUST=true ANAT_INHO_COR=method=SyN,otsu_thresh=2,multiotsu=true` |
| sudmex_crv_mor, sudmex_mor_T2 | DBM `pp_lsq6` (from the 02 log; mor_T2 `data/` is now empty) | `disable` | RSA | `ANAT_SOURCE=dbm` |
| sudmex_alcohol_rat | DBM `pp_lsq6` (files identical) | not recorded in the log | RIP (not reoriented) | `ANAT_SOURCE=dbm`, defaults |
| adolescence_thc | **mouse**, raw FLASH T1 (RABIES bias correction), DSURQE template | `SyN,multiotsu=true`, STC `alt+z`, axis `Y`, `tzero=0` | LPI anat / LSA BOLD (nibabel) | project-specific scripts in `adolescence_thc/code/` (settings in `project_params.sh`) |

For projects whose source is known, create the marker once so 02 can run: `echo dbm > /scratch/$USER/<project>/analysis/fmri/anat_source.txt`.

### QC after step 01 (mandatory)

Follow RABIES *How to check orientation*: open one anatomical image and one BOLD image of a few subjects **over the SIGMA template** (e.g. `fsleyes` or `itksnap`):

- Anterior/posterior, left/right and superior/inferior must point the same way as in the template.
- Anatomical and functional images must be in the same orientation.

| You see | Change | Why |
|---|---|---|
| Brain upside down / flipped vs template | Fix orientation in 01 (`3drefit -orient`, `fslswapdim`) and rerun 01 | RABIES registration cannot fix orientation errors; no parameter tuning will help. |
| Only BOLD flipped | Rerun 01 for the whole project with `REORIENT=true` (or the opposite of what you used) | The BOLD header orientation is wrong; the anatomical is fine. |
| `raw` anatomical has a different orientation code (e.g. IAL) than the BOLD (RSA) | Nothing, **if** both overlay correctly on the template | The orientation code only describes how voxels are stored; what matters is that the header maps them to the right anatomy. |
| Oblique flag still 1 | Rerun deoblique | Oblique headers can cause registration failures. |

## 4. Acquisition settings used by step 02 (decide them now)

The 02 scripts read these settings from **environment variables**, so you don't have to edit the script:

| Variable | Default | What to set |
|---|---|---|
| `TR` | `1.0` | Your TR in seconds. |
| `STC` | `true` | `false` if you don't know the slice order, or the TR is short (≤ 1 s) and you prefer not to risk a wrong correction. |
| `STC_PATTERN` | `seq-z` | Slice order: `seq+z` (bottom→top), `seq-z` (top→bottom), `alt+z`, `alt-z` (interleaved). Read it from the Bruker `method` file (`PVM_ObjOrderList` / `PVM_SliceGeoObj`). |
| `STC_AXIS` | `Y` | The axis along which slices were acquired **after RAS reorientation**. Coronal rat slices are usually `Y`. Axial = `Z`, sagittal = `X`. |
| `RESAMPLING` | `0.3x0.3x0.3` | Commonspace voxel size in mm. Use about the native EPI resolution (smaller = huge files, no real gain). |

Example:

```bash
TR=1.0 STC=true STC_PATTERN=alt+z STC_AXIS=Y RESAMPLING=0.3x0.3x0.3 \
  sbatch neuropsytox_utils/code/fmri_wProc/02_preproc.sh <project>
```

### How to decide about slice-timing correction

| Situation | Decision |
|---|---|
| You know the slice order (method file) and TR ≥ 1.5 s | `STC=true` with the right pattern and axis. |
| You know the order and TR ≈ 1 s | Either is acceptable; keep it consistent across all projects you will compare. |
| You **don't know** the order, or `SliceTiming` is implausible | `STC=false`. A wrong STC is worse than no STC. |
| Projects to be pooled/compared | Use **exactly the same** STC choice. |

Write your choice in the project README together with the TR and resampling.

## 5. Exclusion lists (used by 03 and 04)

You can pass exclusions as `"sub-001_ses-T1,sub-042_ses-T1"` (the script finds the files) or as a `.txt` file. **The `.txt` must contain one full path per line** of the BOLD file in `analysis/fmri/data/`, e.g.

```
/scratch/<user>/<project>/analysis/fmri/data/sub-001/ses-T1/func/sub-001_ses-T1_task-rest_bold.nii.gz
```

Keep **one** exclusion file per project (`analysis/fmri/data/excluded_scans.txt`) and write the **reason** for each exclusion next to it (e.g. in `excluded_scans_reasons.csv`). Reviewers ask for this.
