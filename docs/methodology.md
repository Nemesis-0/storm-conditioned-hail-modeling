# Methodology

This document records the scientific definitions and evaluation logic used in the storm-conditioned hail modeling workflow.

The goal is to make the assumptions of the analysis explicit and to distinguish the **storm definition**, **hail definition**, **sample frame**, **forecast timing**, and **model comparison** from one another.

---

## 1. Scientific framing

Let:

- S denote storm occurrence;
- H denote hail occurrence;
- X denote environmental and radar predictors.

The central modeling question is whether hail probability is better represented directly as

```math
P(H=1\mid X)
```
or hierarchically as

```math
P(S=1\mid X)
\times
P(H=1\mid S=1,X).
```
The hierarchy is not assumed to be superior.

The scientific purpose is to determine where predictive information enters the storm-to-hail process.

---

## 2. Why storm occurrence is defined independently of hail

A negative hail label does not necessarily imply a non-hail storm.

For example, a location with no nearby NOAA hail report may represent:

- a storm that produced no hail;
- clear or weakly convective conditions;
- a location outside the storm;
- an unreported hail event;
- or insufficient observational coverage.

Therefore, storm occurrence is constructed from MRMS radar **before** NOAA hail reports are overlaid.

The intended ordering is:

```text
MRMS radar
    ↓
storm state S
    ↓
NOAA hail overlay
    ↓
joint storm–hail state
```

This allows explicit identification of:

```text
S = 1, H = 1   hail-producing storm
S = 1, H = 0   observed storm without matched hail
S = 0, H = 1   hail report not captured by the storm proxy
S = 0, H = 0   neither storm nor hail
```

The S=1, H=0 category is particularly important for estimating hail probability conditional on storm occurrence.

---

## 3. MRMS storm proxy

Storm occurrence is defined using NOAA MRMS composite reflectivity.

Product:

```text
MergedReflectivityQCComposite_00.50
```

The storm rule is fixed before model evaluation.

### 3.1 Missing-value handling

Both of the following MRMS sentinel values are treated as unavailable:

```text
-99
-999
```

They are converted to missing values before spatial summaries are calculated.

---

## 4. Spatial grid

MRMS native pixels are summarized onto nominal:

```text
0.25° × 0.25°
```

coarse grid cells.

Each nominal coarse cell corresponds to:

```text
25 × 25 = 625
```

fine MRMS pixels.

The full nominal cell size of 625 pixels is used as the denominator when calculating high-reflectivity spatial support.

This prevents missing fine pixels from artificially increasing the apparent fraction of a grid cell exceeding the reflectivity threshold.

---

## 5. Fine-pixel coverage requirement

For a coarse cell to be usable within an individual radar scan, at least:

```math
80\%
```
of its nominal 625 fine pixels must contain valid MRMS values.

Equivalently:

```math
\frac{N_{\text{valid fine pixels}}}{625} \ge 0.80.
```
Scans failing this criterion are not counted as usable scans for that grid cell.

---

## 6. Reflectivity threshold

Convective reflectivity support is defined using:

```math
Z \ge 35\ \text{dBZ}.
```
The 35-dBZ value is treated as a **literature-informed operational proxy**, not as a universal physical boundary between storm and non-storm conditions.

Threshold sensitivity was examined during storm-proxy development rather than selected to maximize downstream model performance.

---

## 7. Spatial-support criterion

For each usable scan and coarse grid cell, define:

```math
A_{35}
=
\frac{
N(\text{fine pixels with } Z\ge35\text{ dBZ})
}{
625
}.
```
A scan provides strong convective spatial support when:

```math
A_{35}\ge0.10.
```
Thus, at least 10% of the full nominal coarse cell must exceed 35 dBZ.

---

## 8. Temporal persistence

A coarse cell is classified as storm-positive within a radar window only when strong convective spatial support persists across at least:

```text
5 scans
```

within that window.

The storm proxy therefore requires both:

- sufficient radar intensity and spatial extent;
- persistence through time.

It is not based on a single extreme reflectivity pixel or a single isolated scan.

---

## 9. Radar-window coverage

A grid cell must also have sufficient temporal radar coverage.

At least:

```math
80\%
```
of the expected scans in the window must be usable.

For the final development windows, 15 scans are expected.

The minimum usable-scan requirement is therefore:

```math
12/15 = 80\%.
```
Cells failing radar-coverage requirements are excluded from model evaluation rather than assigned automatically to S=0.

---

## 10. Final storm rule

The frozen strict storm definition therefore requires:

1. MRMS composite reflectivity;
2. both `-99` and `-999` treated as missing;
3. at least 80% valid fine-pixel coverage within a scan-cell;
4. reflectivity threshold of at least 35 dBZ;
5. high-reflectivity support calculated using the full 625-pixel denominator;
6. at least 10% of the coarse cell above 35 dBZ;
7. persistence for at least 5 scans;
8. at least 80% usable scans within the radar window.

This rule is fixed independently of model performance.

---

## 11. Hail labeling

NOAA hail observations are converted to UTC before temporal matching.

Hail labels are overlaid only after the MRMS storm state has been defined.

For the temporally aligned experiment, a grid cell receives:

```math
H^+=1
```
when at least one qualifying hail report occurs within the corresponding spatial cell and strictly inside the future target interval:

```math
(t_0,\ t_0+30\text{ min}].
```
The future interval excludes the forecast-origin instant itself and includes the ending timestamp.

---

## 12. Explicit storm-proxy misses

Hail-positive cells are never silently removed because the radar storm proxy fails.

The analysis explicitly tracks:

```math
H=1,S=0.
```
This quantity is needed to determine whether conditioning on the radar-defined storm state structurally excludes genuine hail observations.

In the final eligible Notebook 05 development panel:

```text
H=1, S=1 : 17
H=1, S=0 : 0
```

among radar-coverage-eligible hail-positive cells.

Eight additional hail-positive cells were excluded because radar coverage was insufficient.

Therefore, the result is:

> 17 of 17 eligible hail-positive cells were captured by the storm proxy.

It is **not** evidence that the storm rule universally captures all hail events.

---

## 13. Retrospective storm-first sample frame

Notebook 03 constructs a retrospective storm-first sample frame.

The sequence is:

```text
MRMS radar
    ↓
independent storm candidate
    ↓
NOAA hail overlay
```

For the three primary development periods, the strict storm-first sample contains:

```text
504 total grid-hours
169 coverage-eligible grid-hours
123 storm-positive grid-hours
```

with the contingency structure:

```text
S=1, H=1 : 13
S=1, H=0 : 110
S=0, H=1 : 0
S=0, H=0 : 46
```

This retrospective analysis establishes the feasibility of identifying genuine storm-without-hail examples.

It is not itself an operational forecast experiment.

---

## 14. Retrospective conditioning diagnostic

Notebook 04 evaluates a retrospective ERA5-only comparison between:

```text
Direct:
P(H=1 | X)
```

and

```text
Conditioned:
P(H=1 | S=1, X)
```

under the storm-first sample frame.

This notebook is treated as a methodological diagnostic.

Its sensitivity analyses show that ERA5-only storm conditioning does not produce a robust universal advantage.

That mixed result motivates the temporally aligned future-window experiment rather than being discarded.

---

## 15. Temporally aligned predictive formulation

Notebook 05 changes the question from retrospective conditioning to a temporally ordered future-target formulation.

Define:

- X⁻: predictors whose valid times do not extend beyond t₀;
- S⁺: storm occurrence in the future target window;
- H⁺: hail occurrence in the same future target window.

The two competing formulations are:

### Direct

```math
P(H^+=1\mid X^-)
```
### Hierarchical

```math
P(S^+=1\mid X^-)
\times
P(H^+=1\mid S^+=1,X^-).
```
The comparison therefore asks whether explicitly representing future storm occurrence provides useful predictive structure for future hail probability.

---

## 16. Forecast-origin regimes

Two pre-specified timing regimes are evaluated.

### 16.1 30/30 regime

Pre-origin predictor window:

```math
[\text{start},\ \text{start}+30\text{ min})
```
Forecast origin:

```math
t_0=\text{start}+30\text{ min}
```
Future target window:

```math
(t_0,\ t_0+30\text{ min}].
```
---

### 16.2 45/30 regime

Pre-origin predictor window:

```math
[\text{start}+15\text{ min},\ \text{start}+45\text{ min})
```
Forecast origin:

```math
t_0=\text{start}+45\text{ min}
```
Future target window:

```math
(t_0,\ t_0+30\text{ min}].
```
The larger point-estimate gains observed under 45/30 are treated descriptively.

The current sample is too small to establish a definitive timing effect.

---

## 17. Cross-midnight radar handling

Radar retrieval must include every UTC calendar date touched by a requested interval.

This matters especially for periods near midnight.

For example, the 45/30 future target window for the June 13 development period extends into:

```text
2024-06-14
```

A previous implementation queried only the start date and therefore truncated the future radar window.

The corrected implementation enumerates all UTC dates touched by the requested interval before identifying MRMS scans.

All final Notebook 05 results were recomputed after this correction.

---

## 18. Environmental predictors

The frozen ERA5 predictor set contains six variables:

```text
cape
t2m
d2m
t_500
shear_850_500
shear_850_300
```

These represent:

- convective available potential energy;
- 2-m temperature;
- 2-m dew point;
- 500-hPa temperature;
- 850–500-hPa wind shear;
- 850–300-hPa wind shear.

ERA5 is aligned by valid time.

Because ERA5 is retrospective reanalysis, valid-time alignment does not imply operational real-time availability.

The current experiment should therefore be interpreted as a retrospective development study.

---

## 19. Pre-origin radar predictors

Three radar predictors are calculated using only data before t₀:

```text
pre_cmax_dbz
pre_n_area5
pre_n_area10
```

They represent:

- maximum pre-origin composite reflectivity;
- number of scans with at least 5% high-reflectivity area;
- number of scans with at least 10% high-reflectivity area.

No future-window MRMS information is used as a model predictor.

Future MRMS is used only to define the future storm target S⁺.

---

## 20. Frozen predictor set

The final same-X comparison therefore uses nine predictors:

```text
cape
t2m
d2m
t_500
shear_850_500
shear_850_300
pre_cmax_dbz
pre_n_area5
pre_n_area10
```

The identical predictor matrix is used for Direct, Stage 1, and Stage 2 models.

This prevents the hierarchy from receiving additional information unavailable to the Direct model.

---

## 21. Development panel

Notebook 05 uses nine event-enriched 2024 periods.

### Primary

```text
active_20240418
active_20240508
active_20240526
```

### Expanded

```text
expand_active_20240210_07
expand_active_20240502_21
expand_active_20240520_01
expand_active_20240523_10
expand_active_20240609_07
expand_active_20240613_23
```

The final eligible panel contains:

```text
486 rows
297 future storm-positive rows
17 future hail-positive rows
```

split as:

```text
30/30:
232 rows
148 storms
8 hail positives

45/30:
254 rows
149 storms
9 hail positives
```

These periods are intentionally event-enriched and are not representative of long-term hail climatology.

---

## 22. Modeling

All primary comparisons use the same simple modeling pipeline:

```text
SimpleImputer(strategy="median")
StandardScaler()
LogisticRegression(
    C=1,
    penalty="l2",
    solver="liblinear",
    max_iter=5000,
    random_state=20260913
)
```

No large hyperparameter search is performed.

The goal is to test the factorization itself under a controlled same-X comparison rather than optimize model complexity.

---

## 23. Direct model

The Direct model estimates:

```math
\hat p_D
=
P(H^+=1\mid X^-).
```
It is trained using all eligible training rows.

---

## 24. Stage 1

Stage 1 estimates:

```math
\hat p_S
=
P(S^+=1\mid X^-).
```
It is trained using all eligible training rows.

---

## 25. Stage 2

Stage 2 estimates:

```math
\hat p_{H\mid S}
=
P(H^+=1\mid S^+=1,X^-).
```
Training is restricted to rows satisfying:

```math
S^+=1.
```
At prediction time, the fitted Stage-2 model produces conditional hail probabilities for all held-out rows so that the soft hierarchical product can be calculated.

---

## 26. Hierarchical prediction

The final hierarchical probability is:

```math
\hat p_H
=
\hat p_S
\times
\hat p_{H\mid S}.
```
This is a **soft probabilistic hierarchy**.

A held-out row is not assigned zero hail probability simply because Stage 1 predicts storm probability below an arbitrary classification threshold.

No hard Stage-1 gate is used.

---

## 27. Validation

Primary validation uses leave-one-period-out cross-validation.

For each fold:

1. one development period is held out;
2. all models are fitted on the remaining periods;
3. predictions are generated for the held-out period;
4. predictions are pooled across held-out folds for final metrics.

This reduces direct within-period leakage.

However, a period is not necessarily equivalent to a fully independent meteorological event.

Therefore, leave-one-period-out results should not be interpreted as large-sample independent-event validation.

---

## 28. Primary evaluation metrics

The primary evaluation metrics are:

- PR-AUC;
- Brier score;
- calibration behavior.

ROC-AUC and log loss are also reported.

The emphasis on PR-AUC reflects the rarity of hail-positive observations.

Probability scores are important because the scientific target is probabilistic hail risk rather than thresholded classification alone.

---

## 29. LOPO prevalence baseline

For each held-out period, a no-feature baseline predicts the positive-class prevalence estimated from the remaining training periods.

This is called the:

```text
LOPO prevalence baseline
```

rather than climatology.

It is not a long-term climatological estimate.

Because each held-out fold can receive a different constant probability, pooled ROC-AUC for this baseline may differ substantially from 0.5.

This does not imply that a constant predictor has intrinsic ranking skill within an individual fold.

---

## 30. Stage-specific analysis

The hierarchy is evaluated not only as a final probability product but also through separate Stage-1 and Stage-2 diagnostics.

This distinguishes:

```text
Can the model predict future storm occurrence?
```

from:

```text
Among storms, can the model distinguish hail from non-hail?
```

The current development results show substantially stronger skill in Stage 1 than Stage 2.

Stage 2 remains difficult because of the very small number of hail-positive storms.

---

## 31. Component ablation

The hierarchy is additionally decomposed into:

- prevalence-baseline product;
- Direct;
- Stage-1-only product;
- Stage-2-only product;
- full hierarchical product.

The current results do not support the claim that the hierarchy gain is driven exclusively by Stage 1.

Stage-1-only predictions improve some probability scores but reduce discrimination relative to Direct.

Stage-2-only predictions provide limited or inconsistent discrimination improvement.

The full multiplicative hierarchy produces the strongest pooled discrimination in both timing regimes.

The result is therefore interpreted as evidence of **complementary information across stages**, not as proof of a specific interaction mechanism.

---

## 32. Calibration-in-the-large

Calibration is first assessed through mean predicted probability versus observed prevalence.

The hierarchy shows less average overprediction than the Direct model in both timing regimes.

This is described as:

```text
better calibration-in-the-large than Direct
```

rather than as fully calibrated probability estimation.

Sparse reliability-bin summaries are treated as descriptive only.

No stable calibration slope is claimed from the current sample.

---

## 33. Period-cluster bootstrap

A period-cluster bootstrap is used to examine sensitivity of Direct-versus-Hierarchical metric differences to period composition.

The resampling unit is the development period rather than the individual grid row.

This better respects within-period dependence than row-wise resampling.

However, the bootstrap is interpreted as:

> sensitivity to the composition of the nine development periods

rather than as:

- a formal hypothesis test;
- a population confidence interval;
- or a climatological uncertainty estimate.

---

## 34. Current interpretation

Within the corrected event-enriched development panel, the full hierarchy outperforms the same-X Direct model in pooled:

- PR-AUC;
- ROC-AUC;
- Brier score;
- log loss

under both 30/30 and 45/30 timing regimes.

This is evidence that explicitly representing future storm occurrence may provide useful predictive structure.

It is not yet evidence that the hierarchy will outperform Direct prediction across the broader climatological population.

---

## 35. Main limitations

The current analysis is limited by:

1. only nine event-enriched periods;
2. only 17 radar-coverage-eligible hail-positive cells;
3. eight hail-positive cells excluded because of inadequate radar coverage;
4. leave-one-period-out rather than large independent-event validation;
5. retrospective ERA5 reanalysis;
6. limited Stage-2 positive sample size;
7. lack of representative broad-population calibration;
8. no external-year confirmation in the final corrected experiment.

These limitations constrain the strength of scientific conclusions.

---

## 36. Planned final population-level comparison

The next major experiment should use a broad panel constructed independently of hail occurrence.

At each forecast origin t₀:

```text
X^-  = information available no later than t0
S+   = radar-defined storm occurrence in (t0, t0+30 min]
H+   = hail occurrence in (t0, t0+30 min]
```

The final comparison should evaluate:

```math
P(H^+\mid X^-)
```
against:

```math
P(S^+\mid X^-)
P(H^+\mid S^+,X^-)
```
on the same representative held-out population.

Stage 2 may still be developed using storm-enriched data, but final evaluation and calibration should use naturally occurring storms from the representative panel.

---

## 37. Scientific stopping rule

The project does not assume that adding model complexity must improve results.

The intended progression is:

```text
ERA5 baseline
    ↓
ERA5 + pre-origin radar structure
    ↓
additional data sources only if scientifically justified
```

If Stage 2 remains weak after adequate sample size, label quality, timing alignment, and pre-origin radar features are established, that negative result is scientifically informative.

Likewise, if both stages are individually predictive but the hierarchical product underperforms Direct prediction on a representative held-out population, the hierarchy should be rejected rather than further tuned solely to make it win.

The final scientific question remains:

> Where does predictive information enter the hail-generation chain?