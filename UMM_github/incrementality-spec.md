# Incrementality Testing Specification

Technical specification for the incrementality testing component of UMM. This document defines experiment design standards, geo-lift test methodology, holdout test methodology, analysis methods, and integration with the calibration loop.

For methodology context, see [docs/methodology.md](https://github.com/lifesight/unified-marketing-measurement/blob/main/methodology.md). For terminology, see [docs/glossary.md](https://github.com/lifesight/unified-marketing-measurement/blob/main/glossary.md).

---

## Overview

Incrementality tests are the experimental ground truth layer of UMM. While MMM estimates channel contributions from observational data, incrementality tests create controlled conditions — a group that sees a campaign and a group that does not — and measure the difference directly. This experimental evidence is used to validate and calibrate the MMM, and to produce directly measured iROAS for channels where it matters most.

The primary design in UMM is the **geo-lift test**: a geographically randomized controlled experiment. Holdout tests are used where geo-lift is infeasible.

---

## When to run tests

### Priority criteria

Run incrementality tests when one or more of the following apply:

1. **MMM uncertainty is high**: The posterior credible interval for a channel's iROAS is wide (CI width > 50% of the median). The MMM experiment prioritization queue flags these channels automatically.
2. **Spend level is large**: Channels representing >15% of total budget warrant experimental validation regardless of MMM confidence.
3. **Prior test results are stale**: If a channel was last tested >12 months ago and the media mix or creative strategy has changed materially, re-test.
4. **New channel launch**: Any channel added to the mix should be tested within the first 60–90 days.
5. **Platform ROAS and MMM iROAS diverge by >50%**: A large divergence between what a platform reports and what the MMM estimates is a signal that either the MMM or the platform reporting is wrong. A test resolves the question.

### When not to test

- During major promotional events (BFCM, Prime Day, major product launches) — external volatility makes it impossible to isolate media effects
- When a channel is being ramped up or ramped down significantly — the treatment effect is confounded by the spend change
- When the brand is in crisis communications mode — unrelated events will dominate revenue signal

---

## Geo-lift test design

### Step 1: Define the test question

Every test should have one clearly defined question:

- "What is the incremental iROAS of YouTube in the US, at the current spend level?"
- "Does increasing paid social spend by 30% in Germany generate positive incremental return?"
- "Is branded paid search incremental, or does it capture organic intent?"

Clarity on the question determines the experimental design. A vague question produces ambiguous results.

### Step 2: Market identification and eligibility

Identify candidate markets (DMA, city, state/province, country, or region — depending on campaign geography).

**Eligibility requirements:**

| Requirement | Threshold | Rationale |
|---|---|---|
| Minimum weekly revenue | >$10K per market | Below this, noise dominates signal |
| Data history | ≥26 weeks of pre-test data | Required for synthetic control |
| No planned disruptions | During test window | Promotions, distribution changes invalidate results |
| Geographic isolation | Low cross-border spillover | Spillover contaminates control markets |

**Ineligible markets:**
- Markets with planned major promotions or distribution changes during the test window
- Markets with <26 weeks of clean revenue history
- Markets where the channel is not geo-targetable (some programmatic, national TV)

### Step 3: Market matching and synthetic control construction

Match treatment and control markets to minimize pre-test differences.

**Matching criteria** (in order of priority):
1. Revenue trend correlation (last 26 weeks)
2. Revenue volume (within 3x of treatment market)
3. Seasonality pattern similarity
4. Demographic similarity (where available)

**Synthetic control construction:**

For each treatment market, construct a synthetic control as a weighted combination of control markets:

```
Synthetic_control(t) = Σ_j w_j · Revenue_j(t),  for t in pre-test period
```

Solve for weights `w_j` (non-negative, sum to 1) by minimizing the sum of squared differences between the treatment market and the synthetic control in the pre-test period:

```
min Σ_{t=pre-test} [Revenue_treatment(t) - Σ_j w_j · Revenue_j(t)]²
```

Subject to: `w_j ≥ 0` for all j, `Σ_j w_j = 1`

**Fit quality threshold**: The synthetic control must explain ≥80% of pre-test revenue variance (R² ≥ 0.80). If not achieved, expand the candidate control market pool or reconsider market selection.

### Step 4: Power analysis

Before finalizing the design, run a power analysis to confirm the test can detect a meaningful effect.

**Required inputs:**
- Weekly revenue variance per market (from pre-test data)
- Number of treatment and control markets
- Planned test duration (weeks)
- Significance threshold (α = 0.10 is standard for marketing measurement; 0.05 for high-stakes decisions)
- Power target (1-β = 0.80 minimum; 0.90 preferred)

**Minimum detectable effect (MDE):**

```
MDE = z_{α/2} + z_β) · σ_market / √(n_treatment · T)
```

Where:
- `σ_market` — standard deviation of weekly revenue in control markets
- `n_treatment` — number of treatment markets
- `T` — test duration in weeks
- `z_{α/2}` — critical value for significance level (1.645 for α=0.10)
- `z_β` — critical value for power (0.842 for 80% power)

**Decision rule**: If MDE > expected true lift (based on MMM estimate or platform benchmark), the test is underpowered. Options: add more markets, extend the test, or accept lower power with explicit documentation.

### Step 5: Blackout and setup

**Pre-test blackout**: 2–4 weeks before the test begins, exclude test markets from any campaign targeting changes or budget adjustments. This prevents pre-test contamination that would bias the synthetic control.

**Platform setup**: Configure geo-targeting to include only treatment markets. Verify targeting before launch — a misdirected test cannot be corrected after the fact.

**Tracking verification**: Confirm revenue attribution is working in all treatment markets. Run a 1-week hold before launching to verify data quality.

**Documentation**: Record test parameters before launch in the experiment calendar:
- Test hypothesis
- Treatment and control market lists
- Expected MDE
- Significance threshold
- Planned start and end dates
- Campaign spend during test

### Step 6: Test monitoring

During the test:

- **Do not adjust** treatment or control market targeting, spend, or creative mid-test. Changes invalidate the synthetic control.
- **Monitor data pipelines** daily for the first week to catch tracking failures early.
- **Flag external events** (competitor launches, news events, weather anomalies) that could contaminate results. Extend the test if a major contaminating event occurs during the first half.
- **Do not peek** — running significance tests mid-test inflates Type I error. Check results only at the planned end date (or use sequential testing methods explicitly).

### Step 7: Analysis

**Primary analysis — synthetic control difference:**

```
Incremental_revenue(t) = Revenue_treatment(t) - Synthetic_control(t),  for t in test period
```

Sum across the test period for total incremental revenue. Compute iROAS:

```
iROAS = Σ Incremental_revenue(t) / Campaign_spend_in_treatment_markets
```

**Statistical inference:**

Use permutation testing to compute p-values:

1. Randomly reassign treatment/control labels across all markets 1,000–10,000 times
2. Compute the synthetic control gap for each permutation
3. The p-value is the fraction of permutations where the gap exceeds the observed gap

This non-parametric approach is robust to non-normal residuals and small market counts.

**Confidence intervals:**

Bootstrap the synthetic control weights 1,000 times (sampling from the control market pool with replacement). Report the 10th and 90th percentile of the resulting iROAS estimates as the 80% confidence interval.

### Step 8: Reporting

Every completed test report must include:

| Section | Contents |
|---|---|
| Test summary | Question, hypothesis, dates, markets, spend |
| Pre-test fit | Synthetic control R², market matching quality |
| Primary result | iROAS point estimate, 80% CI, p-value |
| Lift chart | Weekly treatment vs synthetic control revenue |
| Significance assessment | Pass/fail against pre-specified α |
| Post-period analysis | Did lift persist after test? (if post-period data available) |
| MMM calibration delta | How much does this update the MMM's channel estimate? |
| Next steps | Recommended action, follow-on test if warranted |

---

## Holdout test design

Used when geo-lift is infeasible — primarily for email/CRM, loyalty, retargeting, and always-on digital campaigns targeting identified users.

### User-level holdout

**Assignment**: Randomly assign eligible users to treatment (receives campaign) or holdout (does not receive campaign) groups at the start of the test. Assignment must be done at the user level before any campaign exposure.

**Holdout size**: Minimum 10% holdout group; 20% preferred for higher-spend campaigns. Larger holdouts increase statistical power but increase opportunity cost.

**Stratified assignment**: Stratify by relevant user segments (loyalty tier, purchase recency, predicted LTV) to ensure holdout group is representative.

**Analysis:**

```
Incremental_conversions = (Conversion_rate_treatment - Conversion_rate_holdout) · N_treatment
iROAS = Incremental_revenue / Campaign_spend_on_treatment_group
```

Use a two-proportion z-test for significance (or chi-squared for conversion count data).

### Ghost bid holdout (where platform-supported)

Some ad platforms support ghost bid holdouts natively — the platform identifies users who would have seen an ad, wins the auction, but serves a blank or PSA. This is the cleanest user-level holdout design where available.

Where supported, ghost bid holdouts are preferred over post-exposure holdouts because they control for ad server selection bias.

---

## Experiment calendar management

### Structure

The experiment calendar tracks:

| Status | Definition |
|---|---|
| Active | Currently running — do not modify |
| Queued | Designed and approved — awaiting scheduling |
| In design | Being designed — not yet approved |
| Completed | Finished — results integrated into MMM |
| Cancelled | Cancelled before or during test — document reason |

### Prioritization

New experiments are prioritized using the MMM uncertainty score:

```
priority_score(c) = uncertainty_score(c) · spend_share(c) · days_since_last_test(c)
```

Channels with high uncertainty, large spend, and no recent tests rank highest. The experiment calendar is reviewed quarterly and updated as MMM outputs change.

### Minimum test cadence

| Brand scale | Minimum tests per year |
|---|---|
| < $5M annual media spend | 2–4 |
| $5M–$50M | 4–8 |
| > $50M | 8–12+ |

At least one test should be active at all times.

---

## Integration with UMM calibration

### MMM prior update protocol

After each completed geo-lift test, update the MMM channel priors using the experimental iROAS estimate. The update is applied before the next weekly model refit.

**Prior update approach:**

The experimental iROAS constrains the expected contribution coefficient for the channel. Translate the experimental result into a constraint on the model parameters:

```
Expected_iROAS_from_model = β_c · E[f_sat(adstock_c)] / E[spend_c]
```

Set an informative prior on `β_c` such that the implied model iROAS is consistent with the experimental estimate, weighted by the experimental confidence interval width. Narrow CIs produce stronger priors; wide CIs produce weaker priors.

**Divergence handling:**

If the experimental result diverges >40% from the MMM estimate, convene a model review before updating priors:
- Check data quality for the test period in both MMM and experiment data
- Check for contaminating events during the test
- Check synthetic control fit quality
- If all checks pass, the experiment supersedes the MMM — update priors and flag for calibration review

### Attribution weight update

Pass experimental iROAS estimates to the attribution calibration process after each completed test. See [attribution-spec.md](https://github.com/lifesight/unified-marketing-measurement/blob/main/attribution-spec.md) for the calibration weighting formula.

---

## Common failure modes

| Failure | Cause | Prevention |
|---|---|---|
| Synthetic control fit < 80% R² | Too few control markets; unusual treatment market | Expand control pool; reconsider market selection |
| Contamination from spillover | Geographic spillover between treatment/control | Use DMA or country-level markets; check spillover pre-test |
| Mid-test spend changes | Campaign budget adjustments during test | Lock spend for test markets before test begins |
| Revenue tracking failure | Data pipeline issues in specific markets | Verify tracking 1 week before launch |
| Underpowered test | MDE larger than true effect | Run power analysis before launch; extend test or add markets |
| Peeking | Testing significance before planned end date | Pre-register analysis date; use sequential testing if needed |
| Contaminating events | Competitor moves, weather, news during test | Flag events; extend test or discount results in report |

---

## References

- Abadie, A., Diamond, A., & Hainmueller, J. (2010). Synthetic Control Methods for Comparative Case Studies. *Journal of the American Statistical Association.*
- Gordon, B., Zettelmeyer, F., Bhargava, N., & Chapsky, D. (2019). A Comparison of Approaches to Advertising Measurement. *Marketing Science.*
- Deng, A., Xu, Y., Kohavi, R., & Walker, T. (2013). Improving the Sensitivity of Online Controlled Experiments by Utilizing Pre-Experiment Data. *ACM WSDM.*
