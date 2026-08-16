# Unsupervised Detection of Building Destruction from Public SAR Imagery

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.8+](https://img.shields.io/badge/python-3.8%2B-blue.svg)](https://www.python.org/)
[![arXiv](https://img.shields.io/badge/PNAS%20Nexus-2025-red)](https://doi.org/10.1093/pnasnexus/pgaf367)

A robust, unsupervised method to detect building destruction in conflict zones using freely available **Sentinel‑1 SAR** imagery.  
This repository provides the complete reproduction code for the paper:

> **Daniel Racek et al.**  
> *Unsupervised detection of building destruction during war from publicly available radar satellite imagery*  
> *PNAS Nexus (2025)*  [DOI: 10.1093/pnasnexus/pgaf367](https://doi.org/10.1093/pnasnexus/pgaf367)

---

## Overview

Armed conflicts cause widespread building destruction. Monitoring this destruction is vital for humanitarian aid, human rights documentation, and academic research. Traditional methods rely on expensive high-resolution optical images and require labelled training data, which is rarely available in war zones.

Our approach instead uses **SAR interferometric coherence** from the European Space Agency’s **Sentinel‑1** satellites—open‑access, all‑weather, day‑and‑night radar images.  
By statistically modelling the coherence time series of individual buildings, we can:

- Detect destruction **in near real‑time** (every 12 days)
- Work **without any labelled training data** (fully unsupervised)
- Provide continuous **p‑values** that quantify the evidence for destruction
- Aggregate pixel‑level evidence to the **building level** via the *harmonic mean p‑value* (HMP)

The method has been validated on three case studies: **Beirut (2020 explosion)**, **Mariupol (2022 Russian invasion)**, and **Gaza (2023‑2024 Israel–Hamas war)**.

---

## Key Features

- **Open data only**: Sentinel‑1 SLC images (freely available from Copernicus)
- **Near real‑time**: 12‑day update cycle, no need for post‑event image stacks
- **Statistically grounded**: Robust quantile regression + Q<sub>n</sub> scale estimator + one‑sided p‑values
- **Pixel → building aggregation**: Weighted HMP that handles spatial dependence
- **Economic loss module**: Damage grade assignment, reconstruction cost Monte Carlo, and insurance portfolio loss simulation
- **Full performance evaluation**: PR/ROC curves, 99:1 negative sampling, 100‑iteration bootstrap

---

## Methodology at a Glance

1. **Coherence maps:** Compute InSAR coherence between two Sentinel‑1 MLC images 12 days apart (Eq. 1 of paper).
2. **First differences:** For each 10‑m pixel, take the first differences of its coherence time series.
3. **Median regression:** Fit a quadratic trend using quantile regression (τ = 0.5) to the first differences.
4. **Residuals:** Compute the residuals from the trend.
5. **Robust scale (Q<sub>n</sub>):** Estimate the pixel‑wise standard deviation using Rousseeuw & Croux’s Q<sub>n</sub> (correct finite‑sample correction).
6. **p‑value:** Calculate a one‑sided (left‑tail) p‑value assuming a normal distribution with the Q<sub>n</sub> scale.
7. **Building‑level HMP:** For each building and time period, combine pixel p‑values with building‑coverage weights using the *weighted harmonic mean p‑value* (Wilson 2019).
8. **Classification:** Apply a threshold (optimized on Mariupol) to flag likely destruction.

*The algorithm is fully unsupervised; the threshold used is transported from the Mariupol case study.*


## Data Preparation

The raw Sentinel‑1 imagery is **not included** due to size, but all intermediate datasets (coherence maps, building footprints, ground truth) are publicly available:

- **OSF repository** (paper’s data): [https://osf.io/kw5g9/](https://osf.io/kw5g9/)
- **Beirut**: Pre‑processed coherence maps (`Coherence_Maps_Beirut`) and ground truth shapefiles (ZKI) – see `beirut.py` for paths.
- **Mariupol**: `.rds` files with coherence, regressions, building footprints and UNOSAT labels.  
- **Gaza**: `.rds` files, boundary shapefile, background image, and UNOSAT buffer files.

*The `mariupol.py` script includes a fallback binary parser for `.rds` if `pyreadr` cannot handle geometry columns.*

Place the data according to the paths defined at the top of each script, or modify the `BASE_PATH` / `beirut_path` variables.  
The scripts expect a directory structure consistent with the OSF repository.

---

## Usage

### Beirut Harbour Explosion

```bash
python beirut.ipynb
```
- Loads 22 coherence maps (11‑Jul‑2020 to 26‑Dec‑2020)
- Maps pixel‑building overlaps
- Fits median regression (λ = 10) and computes Q<sub>n</sub> + p‑values
- Evaluates performance (pixel + building level)
- **Steps 8‑9**: Damage grading (EMS‑98), reconstruction cost estimate with Monte Carlo, insurance loss simulation for a synthetic portfolio
- Outputs: figures, CSV insurance report, summary statistics

### Mariupol (Zhovtnevyi District)

```bash
python mariupol.ipynb
```
- Uses pre‑computed regression residuals (`.rds`) and UNOSAT damage labels
- Recomputes Q<sub>n</sub> with corrected finite‑sample factors (matching R’s `robustbase::Qn`)
- Evaluates pixel‑ and building‑level performance (99:1 sampling, 100 iterations)
- Produces PR/ROC curves, p‑value ridge plots, spatial classification maps
- Saves combined destruction map with UNOSAT ground truth overlay

### Gaza Strip

```bash
python gaza.ipynb
```
- Loads pre‑processed RDS files (coherence, p‑values, pixel alignments, UNOSAT buffers)
- Recomputes `prob_res_qn` using the **corrected** Q<sub>n</sub> implementation to ensure exact match with the paper’s R code
- Computes building‑level HMP for 27 time points (172,916 buildings)
- Compares cumulative destruction percentages with UNOSAT Table 2
- Runs 100‑iteration building‑level PR/ROC evaluation
- Generates Figure 4 p‑value maps, time‑series trends, ridge density plots, and satellite overlay
- Exports all figures and results to `/kaggle/working/outputs`

---

## Important Implementation Corrections

When reproducing the R code in Python, we identified and fixed two critical discrepancies in earlier notebook versions:

1. **Q<sub>n</sub> scale estimator**  
   The original Python code used `np.percentile(diffs, 25)`, which is **not** Rousseeuw & Croux’s Q<sub>n</sub>.  
   The corrected version (`qn_scale()`) computes the exact k‑th order statistic of all pairwise differences, with the proper finite‑sample correction factors (including small‑n adjustments). This now matches `robustbase::Qn` in R exactly.

2. **Harmonic mean p‑value**  
   The original code clipped p‑values to `[1e-300, 1.0]`, causing zeros to bias the result.  
   The corrected version (`harmonic_mean_pvalue()`) drops zeros and NaNs, exactly replicating R’s `combine_p_values_harmonic_mean()`.

Both corrections are implemented in `gaza.py` and `mariupol.py`; `beirut.py` uses the original (slightly less accurate) implementation but the methodology remains valid.

---

## Case Study Highlights

### Beirut
- Single‑day event: explosion on 2020‑08‑04.
- Pixel‑level AUPRC (weighted): **0.657** (SD 0.010)
- Building‑level AUPRC: **0.905** (SD 0.058)
- False positive rate in other periods: **0.166%** of all buildings.

### Mariupol
- Destruction spread over several weeks (Feb–May 2022).
- Pixel‑level AUPRC: **0.550** (SD 0.020)
- Building‑level AUPRC: **0.650** (SD 0.035)
- Using the Beirut‑optimised threshold gives F1 = 0.534 (pixels) and 0.558 (buildings).

### Gaza
- War started 2023‑10‑07; analysis until 2024‑04‑01.
- Pre‑war false positive rate: mean **0.021%** (SD 0.010), matching the paper exactly.
- Building‑level AUPRC: **0.904** (SD 0.04) in our replicated evaluation.
- Cumulative destruction trends closely follow UNOSAT’s composite estimates (Table 2).

---

## License

This project is licensed under the **MIT License** — see the [LICENSE](LICENSE) file for details.

---
