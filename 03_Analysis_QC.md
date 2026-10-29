# 03 – Analysis (group ICA, FC matrices, dual regression, seed-based) and its QC

[← 02 Confound correction QC](02_Confound_Correction_QC.md) | [Home](Home.md) | Next: [04 Group stats →](04_Group_Stats.md)

Goal: obtain **one connectivity map (or matrix) per scan** that truly represents brain networks, using **rat** references.

---

## 1. Group ICA + FC matrices – `04-1_analysis_ICA_FCmatrix.sh` (PASS 1)

```bash
sbatch neuropsytox_utils/code/fmri_wProc/04-1_analysis_ICA_FCmatrix.sh <project> [exclusions] [--atlas functional59|anatomical]
```

- Runs RABIES `analysis` with `--group_ica apply=true,dim=30` and `--FC_matrix` on the SIGMA parcellation (`functional59` by default).
- Output: `rabies/analysis_out_functional59/commonspace_analysis_datasink/`
  - `group_ICA_dir/group_melodic.ica/melodic_IC.nii.gz` → your **rat priors** (30 components)
  - `matrix_data_file/` → one FC matrix per scan (used by NBR)
  - `matrix_fig/` → one matrix image per scan

> Why two passes? RABIES' default priors for `--data_diagnosis` are **mouse** maps. They don't fit rat brains, so the diagnosis must use **your own** group ICA. (The old version of the script used the mouse maps; diagnosis results produced with it should not be trusted.)

### 1a. Classify the ICA components (this is a manual, important step)

Open `melodic_IC.nii.gz` over the SIGMA InVivo template (fsleyes: threshold e.g. |z| > 3; component index = volume index, **starting at 0**), or open `group_melodic.ica/report/00index.html`.

| Label as **network (BOLD)** if… | Label as **confound** if… |
|---|---|
| Mostly in grey matter | Ring at the brain edge (motion) |
| Bilateral and symmetric (most rat networks) | In ventricles / CSF |
| Matches a known rat network: somatosensory (barrel, limb), motor, cingulate/prefrontal (DMN-like), retrosplenial, visual, auditory, insular, striatum, hippocampus, thalamus | Along large vessels / sagittal sinus |
| Smooth, compact clusters | White matter only |
| | Stripes, slices or checkerboard patterns (scanner/ghosting) |
| | Salt-and-pepper noise all over |

Classify **conservatively**: not every component needs a label. Save the decision in `analysis/fmri/data/ICA_classification.csv`:

```
index,label,name,comment
0,confound,edge,motion ring
3,network,somatosensory,bilateral S1
...
```

You will reuse the same indices for: pass 2 (diagnosis), `DR_COMPONENTS` in step 06, and the paper's supplementary figure.

### 1b. QC the FC matrices

Open a few images in `matrix_fig/`, and the group average:

| You see | Meaning | Action |
|---|---|---|
| Strong homotopic (left–right same region) correlations, blocks of related regions | Good | – |
| Almost all positive and high correlations everywhere | Global signal/motion dominates | Back to page 02 (row 4 of the decision table) |
| Almost no structure | Network absent or too much cleaning | Check page 02; check registration (page 01) |
| One scan very different from the others | Outlier | Check its diagnosis report; exclude if spurious/absent |

## 2. Data diagnosis with rat priors – `04-1` PASS 2

```bash
sbatch neuropsytox_utils/code/fmri_wProc/04-1_analysis_ICA_FCmatrix.sh <project> [exclusions] \
   --prior_maps /scratch/$USER/<project>/analysis/fmri/rabies/analysis_out_functional59/commonspace_analysis_datasink/group_ICA_dir/group_melodic.ica/melodic_IC.nii.gz \
   --prior_bold_idx "3 5 9" --prior_confound_idx "0 1 7"
```

- Output: `rabies/analysis_diagnosis_out/data_diagnosis_datasink/` → read it with [page 02](02_Confound_Correction_QC.md) sections 3–5.
- After choosing your thresholds, rerun pass 2 with them to regenerate the group report on the kept scans:

```bash
SCAN_QC_THRESHOLDS='{DR:{Dice:[0.3,0.3,0.3],Conf:[0.25,0.25,0.25],Amp:false}}' \
  sbatch neuropsytox_utils/code/fmri_wProc/04-1_analysis_ICA_FCmatrix.sh <project> ... (same arguments)
```

The scans that pass are listed in `analysis_QC/analysis_QC_scanlist.txt`. Add the scans that fail to your exclusion file (with the reason "Dice < x / Conf > y"), and **use the same exclusion file for every analysis below** so that all analyses use the same scans.

> Changing the confound strategy changes the group ICA. If you change step 03, rerun 04-1 pass 1, re-check the classification, then pass 2.

## 3. Dual regression – `04-3_analysis_DR.sh`

```bash
sbatch neuropsytox_utils/code/fmri_wProc/04-3_analysis_DR.sh <project> [exclusions] [suffix] [prior melodic_IC.nii.gz]
```

- Uses your group ICA (`analysis_out_functional59`, else `analysis_out`), or the 4th argument.
- **All** components are used in the regression (noise components "absorb" noise from the networks – that's wanted). Only the **BOLD** components go to group statistics (`DR_COMPONENTS` in 06).
- Output: `rabies/analysis_DR_out/commonspace_analysis_datasink/dual_regression_nii/` → one 4D file per scan (`*_DR_maps.nii.gz`, one volume per component).

QC: open the DR map of a BOLD component for ~5 scans of each group. It should look like the group component. If many scans look like noise, go back to page 02.

## 4. Seed-based connectivity – `04-2_create_ROIs.sh` + `04-3_analysis_SeedBased.sh`

### 4a. Create the seeds

Write `analysis/fmri/data/ROIs.csv`:

```
ROI_model,x,y,z
lS1BF,-5.1,-2.3,3.9
rS1BF,5.1,-2.3,3.9
```

`x,y,z` = world coordinates (mm) **in the template you pass as 2nd argument** (e.g. your sMRI study template, or SIGMA InVivo).

```bash
sbatch neuropsytox_utils/code/fmri_wProc/04-2_create_ROIs.sh <project> /full/path/template.nii.gz 0.3   # radius in mm
```

The script draws spheres, registers your template to SIGMA InVivo (= RABIES commonspace) if needed, and resamples the seeds to the analysis grid. Output list: `rois_list.txt` (used by 04-3).

QC (mandatory): open every `*_sphere_bin.nii.gz` over the SIGMA template.

| You see | Action |
|---|---|
| Seed in the wrong region / outside the brain | Fix coordinates (check sign of x for left/right, and which template they belong to) |
| Seed partly in WM/CSF | Move it into grey matter or make it smaller |
| "empty ROI" warning | Seed too small for 0.3 mm voxels → increase radius (≥ 0.3 mm) |
| Left and right seeds not symmetric | Check coordinates |

### 4b. Run seed-based analysis

```bash
sbatch neuropsytox_utils/code/fmri_wProc/04-3_analysis_SeedBased.sh <project> 0.3 [exclusions]
```

Output: `rabies/analysis_SB_<size>mm_out/.../seed_correlation_maps/` → one Pearson r map per scan and seed.

QC: step 05 makes a **mean map per seed** (one-sample test). It should show the expected network (e.g. a S1 seed → bilateral S1/S2 and motor cortex). If it only shows the seed itself, or edges/ventricles, go back to page 02.

## 5. Which analysis for which question?

| Question | Analysis |
|---|---|
| Does connectivity **of a specific region** change? (hypothesis-driven) | Seed-based |
| Do **whole networks** change in strength/shape? (data-driven) | Dual regression |
| Which **connections between regions** change, and do they form a sub-network? | NBR (FC matrices) |

Using all three is fine, but report how many tests you ran (see [page 04](04_Group_Stats.md), "Many maps").

## 6. Checklist before moving on

- [ ] `ICA_classification.csv` saved.
- [ ] Data diagnosis run with **rat** priors; thresholds chosen; exclusion file updated.
- [ ] Same exclusion file used for DR, seed-based and FC/NBR.
- [ ] Seeds checked visually.
- [ ] Mean seed maps / DR maps look like networks.
