# 05 – Publication figures

[← 04 Group stats](04_Group_Stats.md) | [Home](Home.md) | Next: [06 Troubleshooting →](06_Troubleshooting.md)

Goal: figures that are **honest** (show only corrected results, with the threshold stated) and **reproducible** (made by a script, not by hand).

---

## 1. Voxel-wise results – `08_plotClusterMaps.sh`

```bash
sbatch neuropsytox_utils/code/fmri_wProc/08_plotClusterMaps.sh <project>
```

- Input: the 07 outputs in `Dual_regression/Mult_comparison/`.
- Underlay: SIGMA InVivo template resampled onto the statistics grid (`analysis/fmri/figures_underlay/SIGMA_InVivo_template_on_stats_grid.nii.gz`), so statistics and anatomy line up.
- Output, per model/component, in `.../Component_<c>/fig_cluster_maps/`:
  - `multiple/` – slice series of the corrected Z map with cluster outlines
  - `component_multiple/` – the same with the group ICA component as reference (shows where the effect sits relative to the network)

Currently 08 is written for dual regression; for seed-based results use the same R functions with `Seed_based/Mult_comparison_<size>mm` (see plan).

### Checklist for each voxel-wise figure

| Item | Example caption text |
|---|---|
| Statistic and threshold | "Z maps, voxel-wise p < 0.001, cluster-corrected FWE p < 0.05 (AFNI 3dClustSim, ACF; minimum k = 23 voxels, NN1, bi-sided)" – numbers from `thresholds_used.txt` |
| Colour bar | Show both positive (red) and negative (blue) |
| Slice positions | Bregma coordinates or SIGMA mm |
| Orientation | L/R marked |
| Template | "SIGMA InVivo template (Barrière et al., 2019)" |
| N | Animals and scans per group, after exclusions |

### Show effect sizes, not only maps

For each significant cluster, extract the mean connectivity per scan and plot it by group/session (box/violin + individual points + lines linking sessions of the same animal):

```bash
# mean DR value in cluster 1 for every scan (run inside the AFNI container or with module afni)
for f in rabies/Dual_regression_maps/sub-*/components/*_DR_maps_0003.nii.gz; do
  echo "$f,$(3dmaskave -quiet -mask 'cluster_corrected/MvC_Z_clustered.nii.gz<1..1>' $f)"
done > cluster1_values.csv
```

Then join with `RID.csv` in R and plot with ggplot. This shows the **direction** and **size** of the effect, and lets readers see outliers.

## 2. NBR results – `09_explore_plot_results.sh`

```bash
bash neuropsytox_utils/code/fmri_wProc/09_explore_plot_results.sh <project> [alpha] [--atlas functional59]
```

Output in `analysis/fmri/NBR/<atlas>/plots/`:

| File | Content |
|---|---|
| `summary_all_models.csv` | All models × terms × components: size, strength, FWE p, **`permutation_valid`** |
| `Mod<id>_<term>_component<c>.png` | Network graph of each significant component (only valid terms) |
| `Mod<id>_Tier1_network.png` | System-level network (anatomical mode) |

Caption: number of permutations, cluster-forming threshold, FWE p, and that permutations were within-animal.

## 3. Supplementary material reviewers expect (RABIES recommendation)

- The **spatiotemporal diagnosis** of every scan included (PNG from `figure_temporal_diagnosis/` and `figure_spatial_diagnosis/`).
- The **group statistical report** and **distribution plot** (`analysis_QC/`).
- The **ICA components** and their classification (`melodic_IC.nii.gz` + `ICA_classification.csv`).
- The exclusion list with reasons, and the number of scans excluded at each step.
- The confound-correction options (copy the `03_confound.sh` RABIES call), TR, STC choice, smoothing.

## 4. Methods paragraph template

> Data were preprocessed with RABIES v0.x (Desrosiers-Grégoire et al., 2024): [STC: yes/no, pattern], head-motion correction, [unbiased template / direct] registration to the SIGMA template, resampled to 0.3 mm isotropic. Confound correction included frame censoring (FD > 0.05 mm; scans with < 480 remaining volumes excluded), ICA-AROMA (dim = 10), regression of [WM, CSF, 6 motion parameters], linear detrending, band-pass filtering (0.01–0.1 Hz) and removal of the first and last 30 s. Data quality was assessed with the RABIES data diagnosis using study-specific group-ICA priors (30 components; N classified as networks); scans with Dice < x or confound correlation > y were excluded. Dual regression [and seed-based connectivity] maps were analysed with linear mixed-effects models (AFNI 3dLMEr; model: …), and corrected for multiple comparisons with cluster-size thresholding (voxel-wise p < 0.001; FWE p < 0.05 from 3dClustSim with the ACF model estimated from model residuals).
