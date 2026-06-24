# Causal MMM Specification

Technical specification for the causal Marketing Mix Modeling component of UMM. This document defines the model structure, estimation approach, required inputs, validation criteria, and integration points with the rest of the UMM framework.

For methodology context, see [docs/methodology.md](https://github.com/lifesight/unified-marketing-measurement/blob/main/methodology.md). For terminology, see [docs/glossary.md](https://github.com/lifesight/unified-marketing-measurement/blob/main/glossary.md).

---

## Overview

The causal MMM estimates the incremental contribution of each marketing channel to a business outcome (typically revenue or conversions) using historical observational data. It uses a Bayesian structural time-series framework with carefully specified priors to estimate incremental lift rather than fit-maximizing correlation.

The model is the long-run measurement layer of UMM: it estimates channel contributions continuously across the full historical dataset, identifies where spend is over- or under-performing, and generates budget response curves used for optimization. It is calibrated against incrementality test results to stay grounded in experimental ground truth.

---

## Model structure

### Outcome equation

```
Y(t) = α(t) + Σ_c β_c · f_sat(adstock_c(t)) + Σ_k γ_k · X_k(t) + ε(t)
```

Where:
- `Y(t)` — outcome variable (revenue, conversions) at time t
- `α(t)` — time-varying baseline (trend + seasonality component)
- `β_c` — channel c contribution coefficient
- `f_sat(·)` — saturation transformation (Hill function)
- `adstock_c(t)` — adstock-transformed spend for channel c at time t
- `γ_k` — coefficient for control variable k
- `X_k(t)` — control variable k (seasonality, promotions, external factors)
- `ε(t)` — residual error, modeled as normally distributed

### Baseline component

The time-varying baseline α(t) is modeled as a local linear trend:

```
α(t) = μ(t) + δ(t)
μ(t) = μ(t-1) + δ(t-1) + η_μ(t),   η_μ ~ Normal(0, σ_μ)
δ(t) = δ(t-1) + η_δ(t),              η_δ ~ Normal(0, σ_δ)
```

This captures slow-moving changes in baseline demand (brand equity, distribution, competitive dynamics) without attributing them to media.

### Seasonality

Seasonal components are modeled using Fourier terms:

```
season(t) = Σ_j [a_j · sin(2πjt/P) + b_j · cos(2πjt/P)]
```

Where P is the seasonal period (52 for weekly data with annual seasonality) and j indexes harmonic terms. Typically 2–4 harmonic pairs are sufficient.

For businesses with strong weekly patterns, a day-of-week component is added using indicator variables.

---

## Transformations

### Adstock

```
adstock_c(t) = spend_c(t) + λ_c · adstock_c(t-1)
```

`λ_c` is the decay rate for channel c. Estimated from data with channel-type priors (see Priors section). Geometric decay is the default; delayed-peak adstock (where effects build before decaying) is available for channels with known delayed response patterns.

**Delayed-peak adstock** (for TV, brand campaigns):

```
adstock_c(t) = spend_c(t) * w_0 + spend_c(t-1) * w_1 + ... + spend_c(t-L) * w_L
```

Where weights w follow a bell-curve shape peaking at the lag of maximum effect.

### Saturation (Hill function)

```
f_sat(x; K, n) = x^n / (x^n + K^n)
```

Parameters:
- `K` — half-saturation point: the spend level at which 50% of maximum effect is achieved
- `n` — shape parameter: controls curve steepness (higher n = sharper transition from low to high saturation)

The Hill function is bounded [0, 1] and monotonically increasing, suitable for modeling diminishing returns. The product `β_c · f_sat(adstock_c(t))` gives the channel's contribution to revenue at time t.

**Budget response curve** from the saturation function:

```
iROAS(spend) = β_c · [f_sat(spend + Δ) - f_sat(spend)] / Δ
```

Where Δ is a small increment. This gives the marginal incremental return at each spend level — the basis for budget optimization.

---

## Priors

Priors are specified per channel type. Values below represent the default prior distributions; they are updated over time using completed incrementality test results.

### Adstock decay (λ)

| Channel type | Prior distribution | Mode | 90% CI |
|---|---|---|---|
| Paid search (brand) | Beta(2, 10) | 0.1 | [0.02, 0.35] |
| Paid search (non-brand) | Beta(3, 10) | 0.2 | [0.05, 0.45] |
| Paid social | Beta(4, 8) | 0.33 | [0.1, 0.6] |
| Display / programmatic | Beta(4, 8) | 0.33 | [0.1, 0.6] |
| Online video | Beta(5, 7) | 0.42 | [0.15, 0.7] |
| TV (national) | Beta(7, 5) | 0.6 | [0.3, 0.85] |
| TV (local/cable) | Beta(6, 5) | 0.55 | [0.25, 0.8] |
| OOH | Beta(6, 5) | 0.55 | [0.25, 0.8] |
| Radio | Beta(5, 6) | 0.45 | [0.2, 0.72] |
| Email / CRM | Beta(2, 8) | 0.2 | [0.04, 0.45] |
| Affiliate | Beta(2, 10) | 0.15 | [0.03, 0.38] |

### Saturation shape (n)

Default `n ~ HalfNormal(1)`, mode 0, most mass between 0.5 and 3. Channels with known sharp saturation (direct response) may use tighter priors toward n=2–3.

### Channel contribution coefficient (β)

`β_c ~ HalfNormal(σ_β)` where `σ_β` is calibrated to the scale of the outcome variable. Positive constraint enforced — advertising cannot have negative returns in the model (separate spend efficiency analysis handles situations where reduction in spend improves outcomes).

### Baseline variance

`σ_μ ~ HalfNormal(0.05)`, `σ_δ ~ HalfNormal(0.01)` — weakly informative, allowing baseline to drift slowly.

### Control variable coefficients

`γ_k ~ Normal(0, 1)` for standardized control variables.

---

## Estimation

### Sampling

Hamiltonian Monte Carlo via Stan (recommended) or PyMC. Configuration:

```
chains: 4
warmup: 1000
samples: 2000 (per chain, post-warmup)
target_accept: 0.95
max_treedepth: 12
```

Total posterior samples: 8,000 (after warmup discard). Thinning: not required with HMC.

### Convergence diagnostics

All parameters must pass convergence checks before outputs are used:

| Diagnostic | Threshold | Action if failed |
|---|---|---|
| R-hat | < 1.05 for all parameters | Increase warmup, check model specification |
| Bulk ESS | > 400 per chain | Increase samples |
| Tail ESS | > 400 per chain | Increase samples or reparameterize |
| MCMC divergences | < 1% of samples | Increase `target_accept`, reparameterize |
| Energy Bayesian fraction of missing information (E-BFMI) | > 0.2 | Model reparameterization required |

Failed convergence invalidates model outputs. Do not report results from non-converged models.

### Posterior summary

Report per parameter:
- Median (point estimate)
- Mean
- 10th, 25th, 75th, 90th percentile
- 80% credible interval (10th–90th)
- 95% credible interval

Standard reporting uses median ± 80% CI.

---

## Required inputs

### Spend data

- Granularity: Weekly (minimum), daily (preferred for digital channels)
- History: 2+ years (104+ weekly observations) strongly preferred; absolute minimum 78 weeks
- Format: See [schemas/input-schema.json](https://github.com/lifesight/unified-marketing-measurement/blob/main/input-schema.json)
- Required channels: All channels with material spend (>2% of total budget)
- Missing values: Impute zeros for weeks with no spend; flag anomalous zero-spend weeks for review

### Outcome data

- Granularity: Matching spend granularity (weekly or daily)
- Metric: Revenue (preferred), conversions, or other primary KPI
- Geography: Must match spend geography (national if spend is national; regional if spend is regional)
- Adjustment: Remove revenue attributable to non-marketing factors where identifiable (e.g., large B2B deals not driven by marketing)

### Control variables

Required where applicable:

| Variable | Source | Notes |
|---|---|---|
| Holiday indicators | Calendar | Country-specific; include lead-up periods for major holidays |
| Promotional events | Business records | Price promotions, major launches, distribution changes |
| Competitor activity | Paid data or estimation | Competitor spend, product launches (where available) |
| Macroeconomic | Public data | Consumer confidence, unemployment, category spend index |
| Seasonality | Fourier terms (derived) | Auto-generated from time index |
| COVID/anomaly indicators | Business records | Flag anomalous periods; model separately |

### Data quality requirements

Before model fitting:
- [ ] No unexplained step changes in revenue (>30% week-over-week) without corresponding covariate
- [ ] Spend data and revenue data from the same source or reconciled
- [ ] No channels with zero spend throughout (exclude or impute)
- [ ] Holiday and promotional calendars complete for the full history
- [ ] Spend data matches finance records within 5%

---

## Validation

### Holdout validation

Hold out the final 10–20% of the time series (minimum 13 weeks). Fit the model on the remaining data. Measure prediction accuracy on the holdout:

| Metric | Target | Action if missed |
|---|---|---|
| MAPE (holdout) | < 10% | Review covariate specification |
| WAPE (holdout) | < 8% | Review baseline model |
| R² (holdout) | > 0.85 | Review model structure |
| Directional accuracy | > 90% of weeks | Review adstock specification |

### Experimental calibration check

After each completed geo-lift test, compare:
- MMM estimated iROAS for the tested channel (posterior median, 80% CI)
- Experimentally measured iROAS from the geo-lift test

| Divergence | Status | Action |
|---|---|---|
| < 20% | Within tolerance | Log result; update priors |
| 20–40% | Yellow flag | Investigate; update priors; schedule recheck |
| > 40% | Red flag | Halt reporting; model review required |

Experimental iROAS is incorporated as an informative prior update:

```
Prior_updated(λ_c) ∝ Prior_original(λ_c) · Likelihood(experimental_iROAS | λ_c)
```

### Decomposition sanity check

Channel contributions + baseline + control variable effects should sum to total revenue within ±5%. Large residuals indicate missing covariates or structural model misspecification.

### Multicollinearity check

Compute variance inflation factors (VIF) for all spend channels after adstock transformation. VIF > 10 indicates problematic collinearity — consider channel aggregation, stronger priors, or external data sources.

---

## Outputs

### Channel-level outputs

Per channel, per time period:

| Output | Definition |
|---|---|
| Contribution ($) | Estimated incremental revenue from this channel |
| Contribution (%) | Share of total marketing-driven revenue |
| iROAS (current spend) | Marginal incremental return at current spend level |
| iROAS (optimal spend) | iROAS at the budget-optimal spend level |
| Adstock decay (λ) | Posterior median and 80% CI |
| Saturation status | Distance from half-saturation point (under/at/over-saturated) |

### Aggregate outputs

| Output | Definition |
|---|---|
| Revenue decomposition | Baseline + each channel + control variables |
| Total marketing-driven revenue | Sum of all channel contributions |
| Total iROAS (portfolio) | Total incremental revenue / total spend |
| Optimal budget allocation | Spend per channel maximizing iRevenue at given total budget |
| Budget response curves | iROAS as a function of spend, per channel |

### Uncertainty reporting

All outputs include 80% credible intervals. Outputs with CI width > 50% of the point estimate should be flagged as high-uncertainty and not used as the primary basis for budget decisions without experimental validation.

---

## Integration with UMM

### Feeding the experiment roadmap

After each model fit, compute posterior uncertainty by channel:

```
uncertainty_score(c) = (P90_iROAS(c) - P10_iROAS(c)) / median_iROAS(c)
```

Channels with the highest uncertainty scores are prioritized for geo-lift testing. This ensures experimental resources are directed where the model is least confident.

### Receiving experimental calibration

After each completed geo-lift test, update the channel's adstock and saturation priors using the experimental iROAS estimate. The update follows a Bayesian updating rule — the experimental result narrows the posterior for that channel's parameters, improving future model fits.

### Feeding attribution calibration

MMM channel contribution shares (posterior medians) are passed to the attribution model as regularization weights. This prevents the attribution model from assigning dramatically different credit than the MMM estimates over the same period — a key mechanism for catching and correcting attribution model bias.

---

## Implementation notes

### Software

Recommended: Stan (via CmdStan 2.32+) or PyMC (5.x). Both implement HMC and provide the diagnostic outputs required for convergence validation.

Reference implementations are available in `examples/notebooks/`.

### Computational requirements

A typical weekly MMM (2 years of data, 6–8 channels) requires approximately 10–20 minutes of sampling on a modern laptop. Parallel chain execution (4 chains) reduces wall time by ~4x with multiple cores.

### Refit cadence

Refit the model weekly as new data arrives. Use a rolling window (keep all historical data or set a maximum window of 4–5 years to avoid over-weighting old patterns). Log model version, data version, and convergence diagnostics with each fit.

---

## References

- Jin, Y. et al. (2017). Bayesian methods for media mix modeling with carryover and shape effects. *Google Research.*
- Gelman, A. et al. (2013). *Bayesian Data Analysis* (3rd ed.). CRC Press.
- Asmussen, N. et al. (2021). Robyn: Automated Marketing Mix Modeling. Meta Open Source.
- Vehtari, A. et al. (2021). Rank-normalization, folding, and localization: An improved Rhat for assessing convergence of MCMC. *Bayesian Analysis.*
