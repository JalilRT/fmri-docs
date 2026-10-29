# 04 – Group statistics (05 → 06 → 07, and NBR)

[← 03 Analysis QC](03_Analysis_QC.md) | [Home](Home.md) | Next: [05 Figures →](05_Figures.md)

Goal: test your hypothesis on the **clean, QC-approved** maps, with a model that matches the design, and with **correct control of false positives**.

> Decide the model(s) **before** looking at results, and write them down (pre-registration style). Changing models after seeing results inflates false positives.

---

## 1. Step 05 – collect the maps and build the analysis mask

```bash
bash neuropsytox_utils/code/fmri_wProc/05_r2z_transform_SB_DR.sh <project> dual_reg
bash neuropsytox_utils/code/fmri_wProc/05_r2z_transform_SB_DR.sh <project> seed 0.3
```

| Type | What 05 does | Output |
|---|---|---|
| `dual_reg` | Copies the DR maps **unchanged**. DR maps are regression coefficients (not correlations), so **no Fisher z-transform** is applied. | `rabies/Dual_regression_maps/<sub_ses>/` |
| `seed` | Applies Fisher z (atanh) to each seed correlation map. Then runs a **one-sample test per seed** (all scans pooled, cluster-corrected) to show the mean network. | `rabies/Seed_based_zscored_<size>mm/<sub_ses>/`, `Mean_<seed>/` |
| both | Creates `analysis_mask.nii.gz` = group ICA brain mask on the maps' grid (fallback: SIGMA InVivo brain mask) | in the output folder |

The seed one-sample maps are descriptive (they check the network is there); they are not the group comparison.

## 2. The phenotype table – `analysis/fmri/data/RID.csv`

One row **per scan**, with at least `RID` (e.g. `sub-001`) and `Session` (e.g. `ses-T1`), plus every variable used in the models:

```
RID,Session,Group,Sex,Age,Batch,meanFD
sub-001,ses-T1,CTRL,male,57,b1,0.021
sub-001,ses-T2,CTRL,male,90,b1,0.025
```

- Use text labels for groups (`CTRL`, `MOR`), not 0/1, otherwise 3dLMEr treats them as numbers.
- Add `meanFD` (from `FD_summary.csv`, page 01) if motion differed between groups.
- The scripts report rows **dropped** because a map or a phenotype is missing – read that message.

## 3. The model file(s) – `analysis/fmri/data/Mod1.txt, Mod2.txt, …`

Start from `code/fmri_wProc/resources/model_template.txt`. Each `Mod*.txt` = one model; 06 runs all of them. Lines starting with `#` are ignored. Do not add `-prefix`, `-mask`, `-resid` or `-dataTable` (06 adds them). The subject column is called `Subj` (06 renames `RID`).

### Choose the model for your design

| Design | `-model` | Notes |
|---|---|---|
| 2+ groups, one session | `'Group+Sex+(1\|Subj)'` | With one scan per animal the random effect is not identifiable – use `3dttest++`/`3dMVM`, or keep LMEr only if some animals have repeats. |
| Groups × sessions (longitudinal) | `'Group*Session+Sex+(1\|Subj)'` | The `Group:Session` interaction = "does the change over time differ between groups?" Usually the main hypothesis. |
| Groups × continuous time/age | `'Group*Age+Sex+(1\|Subj)'` + `-qVars 'Age'` | Centre Age (see below). |
| Continuous predictor (e.g. dominance score) | `'Score*Session+(1\|Subj)'` + `-qVars 'Score'` | |
| Few batches (< 5) | `+Batch` as a **fixed** effect | Random effects need ~5+ levels. |

### Centering continuous variables (`-qVarCenters`)

- Without `-qVarCenters`, 3dLMEr centres at the **sample mean** → group effects are tested at the average age. This is usually what you want.
- `-qVarCenters 0` tests group effects at **Age = 0** (birth) – almost never meaningful. Some older project files (`Dominance_project`, `sudmex_alcohol_rat` `data/Mod1.txt`) still use it: **fix them before rerunning**.

### Contrasts

- `-gltCode name 'Group : 1*MOR -1*CTRL'` → pairwise comparison (gives an effect brick and a `Z` brick).
- Interaction contrast example: `-gltCode MORxT 'Group : 1*MOR -1*CTRL Session : 1*ses-T2 -1*ses-T1'`.
- Only write the contrasts you need: every contrast is another test.

## 4. Step 06 – run the voxel-wise models

```bash
DR_COMPONENTS="3 5 9" bash neuropsytox_utils/code/fmri_wProc/06_3dLMEr.sh <project> dual_reg   # BOLD components only (0-based)
bash neuropsytox_utils/code/fmri_wProc/06_3dLMEr.sh <project> seed 0.3
```

- One Slurm job per model × component (or seed). Output: `Dual_regression/<Mod>/<Mod>_<comp>.nii.gz` (+ `_resid.nii.gz`) or `Seed_based/<Mod>/...`.
- Existing results are **skipped**. To rerun a model after changing it, delete (or rename) that `Dual_regression/<Mod>/` folder first.
- `DR_COMPONENTS` = the BOLD components from `ICA_classification.csv`. Without it, all ~30 components (most are noise) are modelled and you get many more tests.

What is inside each result file:

| Brick label | Meaning | Thresholded by 07 as |
|---|---|---|
| `<term> Chi-sq` | Omnibus test of a model term (e.g. `Group`, `Group:Session`) | 1-sided χ² (p = PTHR) |
| `<contrast>` | Effect size (difference in connectivity) | not thresholded (used for masking/plots) |
| `<contrast> Z` | Z-statistic of the contrast | 2-sided Z (p = PTHR) |

## 5. Step 07 – correct for multiple comparisons

```bash
sbatch neuropsytox_utils/code/fmri_wProc/07_multiple_comparison_correction.sh <project> dual_reg [mod_number]
sbatch neuropsytox_utils/code/fmri_wProc/07_multiple_comparison_correction.sh <project> seed [mod_number] 0.3
PTHR=0.005 sbatch ...   # change the voxel-wise threshold (default 0.001)
ALPHA=0.0167 sbatch ... # cluster-level FWE alpha per map (default 0.05), e.g. Bonferroni over 3 maps
```

How it works (plain words):

1. **Smoothness**: `3dFWHMx -acf` measures how smooth the **model residuals** are (that's why 06 saves `_resid`).
2. **Minimum cluster size**: `3dClustSim` simulates noise with that smoothness inside the analysis mask and finds the cluster size that random noise reaches only 5% of the time (at voxel p < PTHR).
3. **Clusters**: `3dClusterize` keeps clusters larger than that size (bi-sided for Z, 1-sided for χ²).
4. **FDR**: voxel-wise FDR q-values are also written, as an alternative.

Outputs per model/component in `Dual_regression/Mult_comparison/<Mod>/Component_<c>/` (seed: `Seed_based/Mult_comparison_<size>mm/`):

| File | Content |
|---|---|
| `acf_estimates/thresholds_used.txt` | ACF, PTHR and cluster sizes used → **report these numbers in the paper** |
| `cluster_corrected/*_clustered.nii.gz` | Cluster map (each cluster has an ID) |
| `cluster_corrected/cluster_statistics_summary.csv` | Size, peak, coordinates per cluster |
| `effect_masked/*_effect_masked.nii.gz` | Effect size inside significant clusters |
| `FDR_corrected/*_FDR_qvals.nii.gz` | Voxel-wise FDR q-values |

### Rules for cluster correction

| Rule | Why |
|---|---|
| Use PTHR = 0.001 (default). Use 0.005 only if justified and reported. | Lenient thresholds (p < 0.01, 0.05) give inflated false positives with cluster correction (Eklund et al. 2016). |
| A significant cluster means "there is an effect *somewhere* in this cluster", not at every voxel. | Report peak and extent; don't over-interpret the shape. |
| Interpret `Chi-sq` (omnibus) first, then the contrasts. | Avoids testing many contrasts without an overall effect. |
| Big clusters crossing many regions → describe with the atlas, report the peak. | |

### Many maps = many tests (important)

07 corrects **within** each map. If you test several components/seeds/contrasts, the chance of at least one false positive grows. 07 prints the total number of maps tested. Choose and report one strategy:

| Strategy | When |
|---|---|
| Bonferroni across maps: run 07 with `ALPHA=0.05/<number of maps>` (e.g. 3 BOLD components → `ALPHA=0.0167 sbatch 07_...`) | Confirmatory claims |
| Pre-register a single primary map/contrast; the rest are exploratory | Hypothesis-driven studies |
| Report everything as exploratory | Pilot studies |

## 6. NBR – network-based statistics on FC matrices

```bash
bash neuropsytox_utils/code/fmri_wProc/06_NBR.sh <project> [nperm] [--atlas functional59|anatomical]
bash neuropsytox_utils/code/fmri_wProc/09_explore_plot_results.sh <project> [alpha]
```

`Models.txt` (in `analysis/fmri/data/`): first column `Model`, other columns are filters (`none` or a list of kept values):

```
Model Dominance Aggression Sex
Aggression*Session+Sex+(1|RID) none "non_aggressive,aggressive" "female,male"
```

Every model must have `(1|RID)`. See `code/fmri_wProc/NBR/README.md` for details.

### ⚠️ Which NBR terms are valid?

The NBR package shuffles data **only within each animal** (it swaps an animal's own sessions). This is correct for terms that change **within** animals, but it can never change a **between-animal** label (group, sex). So:

| Term type | Examples | p-value valid? |
|---|---|---|
| Within-subject | `Session`, `Age` | ✅ |
| Interaction with a within-subject variable | `Group:Session`, `Subgroup:Age` | ✅ |
| Between-subject only | `Group`, `Sex`, `Subgroup`, `dominance_score`, `Sex:Group` | ❌ **not valid** |

`09_explore_plot_results.sh` now flags this: `summary_all_models.csv` has a `permutation_valid` column, and no figure is made for invalid terms. For between-group main effects use the voxel-wise analyses (3dLMEr), or an NBR model with between-subject permutation (see [Code review and plan](Code_Review_and_Plan.md)).

## 7. Checklist

- [ ] Models written before seeing results
- [ ] `DR_COMPONENTS` = BOLD components only.
- [ ] `thresholds_used.txt` values and the multiple-map strategy reported.
- [ ] NBR: only within-subject terms/interactions interpreted.
