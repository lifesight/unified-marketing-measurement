# UMM Methodology

**Unified Marketing Measurement** is a framework for estimating the causal incremental impact of marketing investment across all channels — continuously, at scale, without requiring a dedicated data science team to run it.

This document is the canonical methodology specification. It covers the principles, each of the three core methods, how calibration works across them, how outputs are validated, and how the system evolves over time.

---

## Principles

### 1. Causation over correlation

Every methodology choice in UMM is made to measure **causal, incremental** impact — not correlation. Platform-reported ROAS, last-touch attribution, and uncalibrated MMM all measure correlation. They tell you which channels fire near conversions; they do not tell you which channels *cause* conversions.

The distinction matters: a brand running heavy TV and seeing strong online sales cannot conclude from correlation that TV is working. It might be. Or organic demand might be driving both. Only a methodology that isolates the counterfactual — what would have happened *without* the marketing — can answer the question.

UMM uses three methods, each designed around causal estimation, and calibrates them against each other to produce answers that are as close to ground truth as the available data allows.

### 2. Unified, not fragmented

Point solutions — MMM-only, incrementality-only, attribution-only — each solve part of the problem and introduce blind spots. A team running three point solutions gets three answers that conflict, with no principled way to reconcile them.

UMM treats the three methods as a single system. MMM informs experiment design. Experiments ground the MMM. Attribution is calibrated to both. The output is one number every team can act on.

### 3. Continuous, not quarterly

Traditional MMM is a quarterly or annual engagement. By the time results are ready, the marketing mix has changed. UMM is designed to update continuously as new data arrives — producing updated channel contributions, budget curves, and iROAS estimates on a rolling basis.

### 4. Actionable by default

Measurement that produces a report is only half the job. UMM outputs are structured as **decision inputs**: budget reallocation recommendations, experiment prioritization queues, and attribution weights that feed into campaign optimization — not just slide decks.

---

## Method 1: Causal MMM

### What it is

Marketing Mix Modeling is a statistical approach that uses historical data — spend by channel, revenue, and external factors — to estimate each channel's contribution to business outcomes. The "causal" variant uses a Bayesian structural time-series framework designed to estimate incremental lift rather than fit-maximizing correlation.

### Model structure

The core model expresses revenue (or other outcome KPIs) as a function of:

- **Paid media spend** by channel, transformed through adstock and saturation functions
- **Organic factors** — seasonality, trend, holidays, promotional events
- **External factors** — macroeconomic indicators, competitor activity, weather (where relevant)
- **Baseline** — the revenue that would occur with zero media spend

```
Revenue(t) = Baseline(t)
           + Σ [channel_contribution(spend(t), adstock, saturation)]
           + Organic(t)
           + External(t)
           + ε(t)
```

### Adstock transformation

Advertising effects decay over time. The adstock transformation models this:

```
adstock(t) = spend(t) + λ · adstock(t-1)
```

Where `λ` is the decay rate — estimated per channel, with priors informed by media type. TV typically has higher carryover than paid search. The decay rate is one of the key parameters calibrated using incrementality tests.

### Saturation transformation

Advertising exhibits diminishing returns. The saturation transformation (typically Hill or Michaelis-Menten) models the response curve:

```
saturation(x) = x^α / (x^α + K^α)
```

Where `K` is the half-saturation point and `α` controls the shape of the curve. These parameters determine the budget curve — the expected iROAS at each spend level — and are critical inputs to budget optimization.

### Bayesian estimation

UMM uses Bayesian inference (specifically, Hamiltonian Monte Carlo via Stan or PyMC) rather than frequentist regression for three reasons:

1. **Prior incorporation**: Domain knowledge about channel response — TV carryover is typically 4–8 weeks; paid search decays in days — can be encoded as priors, improving estimates in data-sparse scenarios.
2. **Uncertainty quantification**: Posterior distributions over parameters give honest confidence intervals on every estimate, rather than point estimates that imply false precision.
3. **Calibration integration**: Incrementality test results can be incorporated as informative priors, updating the model's channel contribution estimates with experimental evidence.

### Priors

Priors are set per media type based on published literature, platform benchmarks, and Lifesight's cross-client calibration database. Key priors:

| Parameter | TV | Paid Search | Paid Social | OOH |
|-----------|-----|-------------|-------------|-----|
| Adstock decay (λ) | 0.6–0.8 | 0.1–0.3 | 0.3–0.5 | 0.5–0.7 |
| Half-saturation point | High | Low | Medium | Medium |
| Peak contribution lag | 2–4 weeks | Same week | 1–2 weeks | 2–4 weeks |

Priors are updated over time using experimental evidence (see Calibration section).

### Validation

Before outputs are used for decision-making, the model is validated against:

- **Holdout validation**: A portion of time-series data is held out during fitting; model accuracy is measured on the holdout period. Target: >90% accuracy on held-out revenue.
- **Experimental calibration check**: MMM channel contribution estimates are compared against experimentally measured iROAS. Deviations >20% trigger model review and prior updates.
- **Decomposition sanity**: Channel contributions should sum to approximately total revenue (within baseline). Large unexplained residuals indicate missing variables.

---

## Method 2: Incrementality testing

### What it is

Incrementality tests are controlled experiments that directly measure the causal impact of a marketing channel or campaign. By creating a treatment group (exposed to the campaign) and a control group (not exposed), the test measures what would not have happened without the marketing.

### Geo-lift tests

The primary experimental design in UMM is the **geo-lift test**: a geographic randomized controlled trial where treatment and control markets are matched prior to the test.

**Why geo?** User-level randomization is increasingly difficult due to privacy constraints, cross-device behavior, and ecosystem fragmentation. Geographic randomization avoids these issues: a city or region either receives the treatment or it does not, with no contamination.

**Test design process:**

1. **Market selection**: Identify candidate markets with sufficient revenue signal. Filter for markets without confounding factors (planned promotions, distribution changes, unusual events).

2. **Market matching**: Use pre-test data to identify treatment/control pairs with similar baseline revenue trajectories. Matching criteria include: revenue trend, seasonality pattern, volume, demographic profile. Matched markets minimize the risk that post-test differences reflect pre-existing divergence rather than treatment effect.

3. **Power analysis**: Estimate the minimum detectable effect (MDE) given the planned test duration, market variance, and significance threshold. A test that cannot detect a 10% lift in the planned timeframe should be redesigned or extended.

4. **Test duration**: Minimum 4 weeks for most channels; 6–8 weeks for channels with longer purchase cycles (TV, OOH, brand campaigns). Shorter tests risk underpowering; longer tests risk drift.

5. **Blackout periods**: Establish blackout windows before and during the test to prevent contamination from other campaigns targeting the same markets.

**Synthetic control method:**

For tests where a small number of treatment markets are used, UMM uses the synthetic control method rather than simple difference-in-differences. A synthetic control constructs a weighted combination of control markets that most closely matches the treatment market's pre-test trajectory. This improves precision and reduces sensitivity to market selection.

```
Revenue_treatment(t) - Σ[w_j · Revenue_control_j(t)] = Incremental_impact(t)
```

Where `w_j` are the synthetic control weights estimated from pre-test data.

**Reading results:**

The primary output is **iROAS** (incremental ROAS):

```
iROAS = Incremental_revenue / Spend_in_treatment_markets
```

Secondary outputs include:
- Incremental revenue (absolute)
- Incremental conversions
- Cost per incremental conversion
- Time-to-effect (when in the test period was lift observed?)
- Persistence of effect post-test (holdout extension)

### Holdout tests

Where geo-lift tests are not feasible (always-on channels, national-only campaigns), UMM uses **holdout tests**: a randomly selected subset of users or accounts is excluded from a campaign for a defined period. The revenue difference between exposed and holdout groups estimates incremental impact.

Holdout tests are used primarily for:
- Email and CRM campaigns (user-level holdout is feasible)
- Loyalty programs
- Retargeting campaigns (where geo-level control is impractical)

### Experiment prioritization

Not every channel can be tested simultaneously. UMM uses MMM uncertainty estimates to prioritize the experiment roadmap: channels where the MMM posterior is widest (highest uncertainty) get tested first.

A continuous experiment calendar is maintained with:
- Active tests (currently running)
- Queued tests (designed, awaiting scheduling)
- Completed tests (results integrated into MMM priors)
- Channels pending test design

---

## Method 3: Causal attribution

### What it is

Attribution assigns fractional credit for conversions to the touchpoints that contributed to them. Causal attribution differs from rules-based attribution (first-touch, last-touch, linear) and uncalibrated data-driven attribution in that it estimates **incremental** contribution — not just presence in the path.

### The attribution problem

Standard attribution models have a fundamental bias: they assign credit based on touchpoint presence, not causal impact. A user who would have converted anyway, regardless of seeing an ad, contributes to the attributed conversions of every channel they touched. This inflates the apparent effectiveness of all channels, but especially retargeting and branded search — channels that skew toward users who were already close to converting.

### Calibrated attribution

UMM calibrates attribution using two inputs:

**1. Incrementality test results**: iROAS estimates from geo-lift and holdout tests provide ground truth on the incremental impact of specific channels. Attribution weights are adjusted so that each channel's attributed revenue aligns with its experimentally measured incremental revenue over the same period.

**2. MMM channel contributions**: MMM provides medium-term estimates of each channel's incremental contribution. Attribution weights are regularized toward MMM estimates, preventing short-term path data from over-indexing on in-path correlations.

The calibration process:

```
Calibrated_attribution_weight(channel) =
    α · Path_based_weight(channel)
  + β · MMM_contribution_share(channel)
  + γ · Experimental_iROAS_index(channel)
```

Where `α + β + γ = 1` and the mixing weights are estimated to minimize error against experimental holdouts.

### What calibrated attribution produces

- **Channel-level iROAS**: Incrementally adjusted return per dollar, at the channel level
- **Campaign-level incrementality scores**: Which campaigns drove true incremental conversions vs. captured existing intent
- **Audience-level lift**: Incrementality by audience segment, where experimental data supports it
- **Path contribution scores**: Fractional credit per touchpoint, calibrated to causal reality

### Limitations

Attribution remains a model. It is strongest for digital channels with good path coverage. It is weaker for:
- Offline channels (TV, OOH, radio) where touchpoints are not measurable at the user level
- Long purchase cycles where path data is incomplete
- Markets with low data density

For these cases, MMM and incrementality test results carry more weight in the unified output.

---

## Calibration: the unifying layer

Calibration is what makes UMM a unified system rather than three separate tools producing conflicting outputs.

### The calibration loop

```
MMM produces → Channel contribution estimates
                        │
                        ▼
Experiment roadmap ← Uncertainty ranking (which channels need testing?)
                        │
                        ▼
Geo-lift tests run → iROAS ground truth
                        │
                        ▼
MMM priors updated ← Experimental results ingested as informative priors
                        │
                        ▼
Attribution calibrated ← MMM contributions + experimental iROAS
                        │
                        ▼
Unified output: one iROAS per channel, continuously updated
```

### Calibration cadence

| Activity | Frequency |
|----------|-----------|
| MMM model refit | Weekly (rolling window) |
| Calibration check (MMM vs experiments) | After each completed test |
| Attribution weight update | Monthly (or post-test) |
| Experiment roadmap review | Quarterly |
| Full model audit | Bi-annually |

### Detecting calibration drift

The system monitors for calibration drift — when MMM estimates diverge from experimental results beyond acceptable thresholds:

- **Yellow flag**: MMM channel contribution differs from experimental iROAS by >20%
- **Red flag**: Divergence >40% — triggers immediate model review
- **Actions**: Prior update, covariate review, data quality audit, model re-specification if needed

---

## Outputs

### Standard UMM outputs

| Output | Definition | Decision use |
|--------|------------|-------------|
| iROAS by channel | Incremental revenue per $1 spent, calibrated | Budget allocation |
| iRevenue by channel | Total incremental revenue attributed, calibrated | Reporting to CFO |
| Budget response curves | Expected iROAS at different spend levels | Spend optimization |
| Optimal budget allocation | Spend distribution maximizing total iRevenue at budget | Budget planning |
| Experiment queue | Prioritized list of channels to test next | Measurement roadmap |
| Attribution weights | Calibrated fractional credit by touchpoint | Campaign-level reporting |
| Contribution decomposition | Revenue breakdown: baseline, paid, organic, external | Executive reporting |

### Confidence intervals

All UMM outputs include posterior credible intervals (Bayesian) or bootstrap confidence intervals (frequentist). Outputs should never be communicated as point estimates without their uncertainty range.

Standard reporting convention:
- **Point estimate**: Median of posterior distribution
- **Uncertainty range**: 80% credible interval (not 95% — 80% CIs are more decision-relevant and less susceptible to false precision)

---

## Implementation notes

### Data requirements

See [data-requirements.md](data-requirements.md) for full specifications. Minimum requirements:

- **Spend data**: Weekly channel-level spend, 2+ years of history preferred (minimum 104 data points)
- **Revenue/outcome data**: Weekly, aligned to spend data
- **External factors**: Relevant macroeconomic, seasonal, and business-specific covariates

### Time to first output

With adequate historical data:
- First MMM output: 2–4 weeks
- First geo-lift test results: 4–8 weeks (after test completion)
- First calibrated output: After first experimental results are available

### Ongoing maintenance

UMM is not a one-time project. It requires:
- Weekly data pipeline maintenance
- Monthly calibration checks
- Quarterly model reviews
- An ongoing experiment program (minimum 4–6 tests per year for a mid-size brand)

---

## Further reading

- [Causal MMM specification](../blob/main/mmm-spec.md)
- [Incrementality testing specification](../specs/incrementality-spec.md)
- [Causal attribution specification](../specs/attribution-spec.md)
- [Calibration specification](../specs/calibration-spec.md)
- [Data requirements](data-requirements.md)
- [Glossary](glossary.md)
- [Lifesight platform](https://lifesight.io) — production implementation of UMM
