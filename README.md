# Storm-Conditioned Hail Modeling

A methodological study of whether short-term hail prediction benefits from explicitly separating **future storm occurrence** from **hail production within storms**.

The project develops a storm-first sample frame using MRMS radar, overlays NOAA hail observations independently, and compares direct and hierarchical probabilistic models under matched predictors, forecast horizons, validation splits, and evaluation samples.

## Research question

Let

- $X^-$ denote predictors whose valid times do not extend beyond a forecast origin $t_0$;
- $S^+$ denote radar-defined storm occurrence in a future target window;
- $H^+$ denote observed hail occurrence in the same future window.

The direct formulation is

$$
P(H^+=1 \mid X^-),
$$

while the hierarchical formulation is

$$
P(S^+=1 \mid X^-)
\times
P(H^+=1 \mid S^+=1, X^-).
$$

The central question is not simply whether the hierarchy wins.

The broader objective is to determine **where predictive information enters the storm-to-hail chain**:

1. through predicting whether a storm will occur;
2. through predicting hail conditional on storm occurrence;
3. or through the combination of both stages.

## Why a storm-first sample frame?

A direct hail classifier can easily acquire a misleading negative class if “non-hail” examples are defined only by the absence of nearby hail reports.

This repository therefore separates two questions:

1. **Was a storm present?**
2. **Did that storm produce hail?**

Storm occurrence is defined from MRMS composite reflectivity independently of NOAA hail reports. Hail labels are overlaid only after the radar-based storm state has been constructed.

This makes it possible to distinguish

$$
S=1,H=1
$$

from the scientifically important

$$
S=1,H=0
$$

case rather than treating arbitrary non-report locations as equivalent to non-hail storms.

## Study design

The final development experiment is a **temporally ordered retrospective pilot**, not an operational forecasting validation.

Two pre-specified forecast-origin regimes are evaluated:

- **30/30:** 30-minute predictor lookback, forecast origin at +30 min, followed by a 30-minute future target window;
- **45/30:** 30-minute predictor lookback, forecast origin at +45 min, followed by a 30-minute future target window.

The predictor set is frozen at nine variables:

**ERA5 environmental predictors**

- CAPE
- 2-m temperature
- 2-m dew point
- 500-hPa temperature
- 850–500-hPa shear
- 850–300-hPa shear

**Pre-$t_0$ MRMS radar predictors**

- maximum composite reflectivity
- number of scans with at least 5% high-reflectivity area
- number of scans with at least 10% high-reflectivity area

The direct and hierarchical formulations use:

- the same predictors;
- the same held-out rows;
- the same logistic-regression learner;
- leave-one-period-out validation;
- no post-$t_0$ predictor information.

ERA5 fields are aligned by valid time but are retrospective reanalysis products, so this experiment should not be interpreted as demonstrating operational real-time forecast performance.

## Corrected development results

The final temporally aligned panel contains nine event-enriched 2024 periods.

After radar-coverage filtering:

| Regime | Eligible rows | Future storms | Future hail |
|---|---:|---:|---:|
| 30/30 | 232 | 148 | 8 |
| 45/30 | 254 | 149 | 9 |
| **Total** | **486** | **297** | **17** |

Among the 17 radar-coverage-eligible hail-positive grid cells,

$$
H^+=1,S^+=1: 17,
\qquad
H^+=1,S^+=0: 0.
$$

Thus, within the eligible development population, the independently constructed MRMS storm proxy captured all observed hail-positive cells.

Eight additional hail-positive grid cells were excluded because radar coverage was inadequate, so the 17/17 result must not be interpreted as universal hail capture.

### Direct versus hierarchical prediction

Pooled leave-one-period-out results are:

| Regime | Model | PR-AUC | ROC-AUC | Brier | Log loss |
|---|---|---:|---:|---:|---:|
| 30/30 | Direct | 0.0485 | 0.5848 | 0.04431 | 0.18218 |
| 30/30 | Hierarchical | **0.0593** | **0.6602** | **0.03929** | **0.16463** |
| 45/30 | Direct | 0.0519 | 0.5977 | 0.04087 | 0.17334 |
| 45/30 | Hierarchical | **0.0931** | **0.6830** | **0.03525** | **0.15273** |

The corrected hierarchical formulation has better pooled PR-AUC, ROC-AUC, Brier score, and log loss than the same-$X$ direct model in both timing regimes.

The discrimination gains are larger in point estimate under 45/30, but this should be treated descriptively rather than as a confirmed timing effect because the two regimes have different radar-coverage-eligible samples and only 8–9 hail-positive cells.

## Where does the hierarchy gain information?

The stage-specific diagnostics provide a more informative result than the aggregate model comparison alone.

### Stage 1 — future storm occurrence

$$
P(S^+=1 \mid X^-)
$$

shows substantial held-out predictive skill.

| Regime | PR-AUC | ROC-AUC |
|---|---:|---:|
| 30/30 | 0.768 | 0.767 |
| 45/30 | 0.801 | 0.771 |

### Stage 2 — hail conditional on storm

$$
P(H^+=1 \mid S^+=1,X^-)
$$

shows weaker and less stable predictive information.

Its ranking metrics exceed the no-feature training-prevalence baseline, but its Brier and log-loss performance does not, and only 8–9 positive hail cells are available for evaluation.

Component ablation shows that:

- Stage 1 alone does not reproduce the full hierarchy's discrimination;
- Stage 2 alone provides only limited additional discrimination;
- the full multiplicative hierarchy produces the strongest PR-AUC and ROC-AUC in both regimes.

The development results therefore suggest **complementary information across the storm-occurrence and conditional-hail stages**, rather than a hierarchy driven entirely by one component.

## Calibration

The observed hail prevalence is approximately 3.4–3.5% in both timing regimes.

Mean held-out predicted probabilities are:

| Regime | LOPO prevalence baseline | Direct | Hierarchical | Observed |
|---|---:|---:|---:|---:|
| 30/30 | 0.0342 | 0.0659 | 0.0516 | 0.0345 |
| 45/30 | 0.0340 | 0.0688 | 0.0528 | 0.0354 |

The direct model overpredicts hail occurrence on average.

The hierarchy reduces this mean overprediction and therefore shows better **calibration-in-the-large** than Direct, although it remains imperfect and does not uniformly outperform the conservative prevalence baseline on proper scoring rules.

Because hail positives are sparse, reliability-bin summaries are treated as descriptive rather than as evidence of a stable calibration curve.

## Repository roadmap

The notebooks are intended to be read in order.

### `01_summer_sample_frame_audit.ipynb`

Audits the earlier negative-sample construction and clarifies what the original negative label actually represented.

Key question:

> Were existing negatives truly storm-without-hail examples?

### `02_mrms_storm_proxy_feasibility.ipynb`

Evaluates whether MRMS composite reflectivity can provide a hail-independent operational storm proxy.

Focuses on:

- missing-value handling;
- spatial coverage;
- reflectivity support;
- scan persistence;
- threshold sensitivity.

### `03_storm_first_sample_construction.ipynb`

Constructs the first explicit storm-first retrospective sample frame:

$$
\text{MRMS storm state}
\rightarrow
\text{NOAA hail overlay}.
$$

This notebook establishes the core storm / hail contingency structure and produces explicit $S=1,H=0$ examples.

### `04_direct_vs_storm_conditioned_modeling.ipynb`

Tests retrospective ERA5-only direct versus storm-conditioned hail modeling.

The main result is intentionally mixed: ERA5-only conditioning does not show a robust advantage under compact-feature and placebo sensitivity analyses.

This motivates the temporally ordered experiment rather than forcing a positive hierarchy result.

### `05_temporally_aligned_hierarchical_modeling.ipynb`

Implements the corrected future-window experiment:

$$
X^-
\rightarrow
S^+
\rightarrow
H^+.
$$

It includes:

- cross-midnight MRMS window correction;
- corrected full-cell radar support;
- future storm and hail construction;
- same-$X$ Direct versus Hierarchical comparison;
- leave-one-period-out evaluation;
- period-cluster bootstrap;
- stage-specific diagnostics;
- component ablation;
- calibration-in-the-large analysis.

This notebook contains the current primary development result.

## Outputs

Derived analysis tables are stored under:

```text
outputs/tables/
```

They include:

- storm-first sample summaries;
- hail-capture audits;
- retrospective comparison diagnostics;
- held-out prediction tables;
- direct-versus-hierarchical metrics;
- period-cluster bootstrap summaries;
- stage-specific skill tables;
- hierarchy component ablations;
- calibration summaries.

Raw meteorological datasets are not distributed in this repository.

## Data

The study uses:

- **NOAA hail observations**
- **NOAA MRMS composite reflectivity**
- **ERA5 reanalysis**

Local raw-data layout and acquisition notes are documented in:

```text
data/README.md
```

The public repository contains code and derived research artifacts rather than raw MRMS or ERA5 files.

## Reproducibility

Install the Python dependencies with:

```bash
pip install -r requirements.txt
```

The notebooks expect the local data structure described in `data/README.md`.

Notebook 05 has been verified from a clean kernel using the complete local development inputs before its final outputs were frozen.

The analysis intentionally uses simple logistic models rather than extensive algorithm or hyperparameter search. The purpose is to test the scientific factorization under a controlled same-$X$ comparison, not to maximize leaderboard performance.

## Limitations

The current results are development evidence rather than climatological validation.

Important limitations include:

- only nine event-enriched periods;
- only 17 radar-coverage-eligible hail-positive grid cells;
- eight additional hail-positive cells excluded because of insufficient radar coverage;
- leave-one-period-out rather than a large independent-event evaluation;
- ERA5 reanalysis rather than operational real-time environmental predictors;
- limited conditional-hail sample size for Stage 2;
- no representative broad-population calibration analysis yet.

The next major scientific step is to evaluate both formulations on the same broad, hail-independent representative panel while preserving the storm-first sample construction and pre-$t_0$ information constraints.

## Research scope and acknowledgment

This repository documents a methodological research branch developed within the **BIRLIUR undergraduate research project at the University of Illinois Urbana-Champaign**.

Earlier team work informed the meteorological data-source review, preliminary sample construction, and broader hail-modeling context. This repository focuses on the subsequent sample-frame audit, hail-independent storm construction, storm-first reformulation, and controlled direct-versus-hierarchical experiments documented in Notebooks 01–05.