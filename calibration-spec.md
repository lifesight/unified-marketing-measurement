# Calibration Specification

Technical specification for the calibration layer of UMM — the mechanism that keeps causal MMM, incrementality testing, and causal attribution aligned into a single consistent measurement system.

For methodology context, see [docs/methodology.md](../docs/methodology.md). For terminology, see [docs/glossary.md](../docs/glossary.md).

---

## Overview

Calibration is what separates UMM from three measurement tools running in parallel. Each method has strengths and blind spots:

- MMM is continuous and covers all channels but is slow to update and cannot run controlled experiments
- Incrementality tests are experimentally rigorous but episodic and resource-constrained
- Attribution is granular and campaign-level but susceptible to selection bias in path data

Calibration is the process by which these methods correct each other's blind spots — experimental results update MMM priors, MMM outputs regularize attribution weights, and the system as a whole converges on estimates of incremental channel impact that no single method could produce alone.

This document specifies the calibration protocols, data flows, update triggers, drift detection rules, and audit procedures.

---

## Calibration architecture

### Data flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    CALIBRATION LAYER                            │
│                                                                 │
│  ┌──────────────┐     prior updates      ┌──────────────────┐  │
│  │  Geo-lift    │ ─────────────────────► │   Causal MMM     │  │
│  │  tests       │                        │                  │  │
│  │              │ ◄─────────────────────  │                  │  │
│  └──────────────┘  experiment roadmap    └────────┬─────────┘  │
│                                                   │             │
│                                     contribution  │             │
│                                     shares        │             │
│                                                   ▼             │
│  ┌──────────────┐  experimental iROAS  ┌──────────────────┐    │
│  │ Holdout      │ ─────────────────────► Causal           │    │
│  │ tests        │                      │ Attribution      │    │
│  └──────────────┘                      └──────────────────┘    │
│                                                                 │
│                              ▼                                  │
│               Unified output: one calibrated iROAS              │
│               per channel, continuously updated                 │
└─────────────────────────────────────────────────────────────────┘
```

### Calibration relationships

| From | To | What is transferred | Trigger |
|---|---|---|---|
| Geo-lift tests | MMM | Informative prior updates on channel parameters | After each completed test |
| MMM | Experiment roadmap | Channel uncertainty scores for test prioritization | Weekly (post model refit) |
| Geo-lift tests | Attribution | Experimental iROAS weights | After each completed test |
| MMM | Attribution | Channel contribution shares for regularization | Monthly |
| Holdout tests | Attribution | User-level incrementality estimates | After each completed test |
| Attribution | MMM | Conversion path coverage rates (data quality signal) | Monthly |

---

## MMM calibration from experiments

### Prior update protocol

When a geo-lift or holdout test completes, the experimental iROAS estimate is incorporated into the MMM as an informative prior update on the affected channel's parameters. This is the primary mechanism by which experimental ground truth corrects the MMM.

**Step 1: Translate experimental result to model parameter constraint**

The experimental iROAS for channel c implies a constraint on the model's contribution coefficient `β_c` and saturation parameters. The relationship is:

```
iROAS_model(c) ≈ β_c · f_sat'(spend_c_bar) / price_mean
```

Where:
- `f_sat'(spend_c_bar)` — derivative of the saturation function at mean spend
- `price_mean` — mean outcome per unit (revenue per conversion)

Given the experimental iROAS estimate and its confidence interval, solve for the implied range of `β_c`.

**Step 2: Construct the informative prior**

The updated prior for `β_c` is a truncated normal centered on the value implied by the experimental iROAS, with standard deviation derived from the experimental confidence interval width:

```
β_c | experimental ~ TruncatedNormal(μ_exp, σ_exp, lower=0)

μ_exp = implied_β_from_iROAS(iROAS_experimental)
σ_exp = implied_β_from_iROAS(iROAS_CI_width) / 3.29  (mapping 80% CI to 1.29σ)
```

**Step 3: Blend with existing prior**

If a prior from earlier experiments exists, blend with the new result using precision weighting:

```
precision_new = 1 / σ_exp²
precision_existing = 1 / σ_existing²

μ_blended = (precision_new · μ_exp + precision_existing · μ_existing) / (precision_new + precision_existing)
σ_blended = 1 / √(precision_new + precision_existing)
```

More recent experiments receive equal weight to older experiments unless explicitly discounted (see Staleness rules below).

**Step 4: Apply prior in next model refit**

The blended prior replaces the generic channel-type prior for channel c in the next scheduled weekly model refit. Document the prior update in the calibration log (see Audit section).

### Staleness rules

Experimental priors decay in influence over time as market conditions change:

| Time since experiment | Prior weight decay |
|---|---|
| 0–6 months | Full weight (no decay) |
| 6–12 months | 75% weight |
| 12–18 months | 50% weight |
| 18–24 months | 25% weight |
| > 24 months | Revert to channel-type prior; schedule re-test |

Apply decay by inflating the prior standard deviation:

```
σ_decayed = σ_original / √(decay_weight)
```

This widens the prior, giving the MMM more freedom to fit the data when experimental evidence is old.

### Multi-geography calibration

When experimental data is available for some geographies but not others:

- Apply experimental priors to geographies where the test ran
- For untested geographies, use a weighted blend: 70% channel-type prior + 30% experimental prior from closest comparable geography
- Flag untested geographies in outputs with an elevated uncertainty marker

---

## Experiment roadmap calibration from MMM

### Uncertainty scoring

After each weekly MMM refit, compute an uncertainty score for each channel:

```
uncertainty_score(c) = (P90_iROAS(c) - P10_iROAS(c)) / median_iROAS(c)
```

This is the normalized width of the 80% credible interval — a dimensionless measure of how uncertain the model is about channel c's incremental efficiency.

### Priority score

Combine uncertainty with spend share and time since last test to generate a test priority score:

```
priority_score(c) = uncertainty_score(c)
                  · log(1 + spend_share(c) · 100)
                  · log(1 + days_since_last_test(c) / 30)
```

The logarithmic scaling prevents any single factor from dominating. Channels with high uncertainty, large spend, and no recent tests rank highest.

**Priority score interpretation:**

| Priority score | Recommendation |
|---|---|
| > 3.0 | Test immediately — high uncertainty on a material channel |
| 1.5–3.0 | Queue for next available test slot |
| 0.5–1.5 | Monitor; test within 6 months |
| < 0.5 | Low priority; test annually or if spend changes materially |

The experiment roadmap is updated weekly with the current priority scores. The measurement team reviews and schedules tests quarterly.

---

## Attribution calibration from MMM and experiments

### Calibration update cadence

| Trigger | Action |
|---|---|
| Completed geo-lift or holdout test | Immediately update experimental iROAS weights for affected channel |
| Monthly MMM refit | Update MMM contribution shares used in attribution mixing |
| Quarterly | Review mixing weights; audit calibration adjustment factors |
| Attribution coverage rate drops >5pp | Increase MMM weight; flag path data issue |

### Mixing weight adjustment rules

The baseline mixing weights (defined in [attribution-spec.md](attribution-spec.md)) are adjusted under the following conditions:

**Increase experimental weight (γ) when:**
- A test completed within the last 6 months for the channel
- The test was well-powered (MDE < observed lift) and had clean results
- Multiple consistent experiments are available

**Increase MMM weight (β) when:**
- No experimental data is available for the channel
- The channel is offline (TV, OOH, radio) where path data cannot capture it
- Path coverage rate is below 50%

**Increase Shapley weight (α) when:**
- MMM uncertainty score for the channel is very high (model is unreliable)
- Both MMM and experimental data are stale (>18 months)
- This is a new channel with no MMM or experimental history

**Never set any weight to zero** unless the data source is completely unavailable for that channel. Even weak evidence from a stale experiment is better than ignoring it entirely.

### Channel-specific calibration rules

Some channels require special handling in attribution calibration:

**Branded paid search:**
- Typically low incrementality (captures existing intent rather than generating new demand)
- Experimental evidence frequently shows iROAS of 0.5–2.0 despite high reported ROAS (5–15)
- Attribution calibration should apply strong downward adjustment when experimental evidence confirms low incrementality
- Do not suppress entirely — branded search does drive some incremental volume

**TV and OOH:**
- Cannot be represented in path data at the touchpoint level
- Attribution calibration applies channel-level credit based entirely on MMM contributions and experimental results
- Distribute channel credit across the post-exposure window (typically 2–4 weeks for TV) weighted by the time-decay profile from the MMM adstock estimate

**Retargeting:**
- High selection bias — retargeting reaches users already close to converting
- Apply maximum downward calibration adjustment when experimental evidence is available
- Without experimental evidence, apply a default 40% discount to raw Shapley retargeting credit as a conservative bias correction
- Schedule incrementality test for retargeting within 90 days if none exists

**Affiliate:**
- Incrementality varies widely — some affiliate placements are genuinely incremental (content affiliates driving new demand); others capture existing intent (coupon sites at checkout)
- Segment affiliate spend by type before attribution if possible; apply different calibration adjustments per affiliate type
- If unsegmented, apply a default 30% discount to raw Shapley affiliate credit

---

## Drift detection

### What calibration drift means

Calibration drift occurs when the three methods diverge beyond acceptable thresholds — MMM estimates, experimental results, and attribution outputs that were once consistent begin to contradict each other. Drift indicates that one or more methods has moved away from ground truth and requires intervention.

### Drift detection rules

Run drift detection checks weekly (automated) and monthly (analyst review).

**Check 1: MMM vs. experimental iROAS divergence**

For channels with experimental data less than 12 months old:

```
divergence(c) = |iROAS_mmm(c) - iROAS_experimental(c)| / iROAS_experimental(c)
```

| Divergence | Status | Action |
|---|---|---|
| < 20% | In tolerance | Log; no action required |
| 20–40% | Yellow flag | Investigate cause; update priors at next refit |
| > 40% | Red flag | Halt using affected channel's MMM output for budget decisions; convene model review |

**Check 2: Attribution vs. MMM channel share divergence**

Compare calibrated attribution channel shares to MMM channel contribution shares over the same period:

```
share_divergence(c) = |attribution_share(c) - mmm_share(c)|
```

Flag if any channel's shares diverge by >15 percentage points. Attribution shares cannot be consistently higher than MMM shares for lower-funnel channels (retargeting, branded search) — this indicates calibration is not working.

**Check 3: Attribution coverage rate**

```
coverage_rate = conversions_with_matched_path / total_conversions
```

Flag if coverage rate falls below 60% or drops >5 percentage points month-over-month. Low coverage means attribution is modeling an increasingly unrepresentative sample of conversions.

**Check 4: Portfolio iROAS consistency**

Compare portfolio-level iROAS across methods:

```
portfolio_iROAS_mmm = Σ_c iRevenue_mmm(c) / total_spend
portfolio_iROAS_attribution = Σ_c iRevenue_attribution(c) / total_spend
portfolio_iROAS_experimental = weighted_mean(iROAS_experimental, by=spend_share)
```

Portfolio-level iROAS from the three methods should agree within 25%. Larger divergence indicates systematic bias in one or more methods.

### Drift response protocol

**Yellow flag response:**
1. Identify which check triggered the flag and which channel(s) are affected
2. Review data inputs for the flagged channel: spend data, revenue data, covariate completeness
3. Check for external events during the flagging period that could explain the divergence
4. Update MMM priors with experimental evidence if available
5. Re-run model and recheck; document in calibration log
6. If flag persists after two consecutive weekly checks, escalate to Red flag protocol

**Red flag response:**
1. Immediately suspend use of flagged channel's output for budget decisions
2. Convene measurement review within 5 business days
3. Systematic audit: data pipeline integrity, model specification, experimental data validity
4. Do not re-enable flagged output until root cause is identified and corrected
5. If root cause cannot be identified within 30 days, revert to MMM-only for affected channel with widened uncertainty intervals

---

## Calibration cadence

| Activity | Frequency | Owner | Output |
|---|---|---|---|
| MMM weekly refit | Weekly | Automated | Updated channel contributions and uncertainty scores |
| Experiment priority scoring | Weekly (post-refit) | Automated | Updated priority scores for experiment calendar |
| Attribution recalibration (scheduled) | Monthly | Automated | Updated mixing weights and calibration factors |
| Attribution recalibration (event-triggered) | After each test | Automated | Immediate update for affected channel |
| Drift detection checks | Weekly (automated) + Monthly (analyst) | Automated + Analyst | Drift flags and resolution actions |
| Calibration audit | Quarterly | Analyst | Full audit report (see Audit section) |
| Mixing weight review | Quarterly | Analyst | Updated mixing weight rationale per channel |
| Full system review | Bi-annually | Measurement team | Methodology update recommendations |

---

## Calibration audit

### Quarterly audit contents

The quarterly calibration audit documents the current state of the calibration system and flags any issues requiring attention:

**1. Experiment coverage summary**
- Channels tested in the past 12 months
- Channels with stale or no experimental data
- Experiments in progress and queued

**2. Prior update log**
- All MMM prior updates applied in the quarter: channel, test date, experimental iROAS, prior before and after

**3. Attribution mixing weights**
- Current mixing weights (α, β, γ) per channel
- Rationale for any non-standard weights
- Channels where mixing weights changed during the quarter

**4. Calibration adjustment factors**
- Calibrated vs. raw Shapley attribution per channel
- Trend in adjustment factors over time (are they stable or drifting?)

**5. Drift flags**
- All yellow and red flags in the quarter
- Resolution actions taken
- Any unresolved flags

**6. Coverage and data quality**
- Path coverage rate trend
- Data pipeline issues and resolutions

**7. Recommendations**
- Channels requiring experimental testing
- Model specification changes to consider
- Data quality improvements to prioritize

The quarterly audit report is stored in the calibration log and reviewed by the measurement team lead.

### Calibration log schema

Every calibration action is logged with the following fields:

```json
{
  "log_id": "string (UUID)",
  "timestamp": "ISO 8601 datetime",
  "action_type": "prior_update | mixing_weight_update | drift_flag | drift_resolution | scheduled_recalibration",
  "channel": "string (channel identifier)",
  "trigger": "experiment_completion | scheduled | drift_detection | manual",
  "experiment_id": "string (if triggered by experiment)",
  "before": {
    "description": "State before the action",
    "values": {}
  },
  "after": {
    "description": "State after the action",
    "values": {}
  },
  "analyst": "string (name or 'automated')",
  "notes": "string"
}
```

The calibration log is append-only. No log entries are deleted or modified after creation.

---

## Limitations

### What calibration cannot fix

- **Fundamental data problems**: Miscoded spend, wrong revenue source, missing channels. Calibration aligns methods; it cannot correct inputs that are wrong at the source.
- **Complete absence of experimental data**: Calibration with only MMM and attribution produces better outputs than either alone, but it cannot substitute for experimental ground truth. Channels with no experimental evidence carry irreducible uncertainty.
- **Severe path coverage loss**: If attribution path coverage falls below 40%, calibration cannot reliably connect path-level data to channel-level experimental evidence. MMM becomes the primary measurement layer.
- **Rapid market changes**: If the media landscape, competitive environment, or consumer behavior changes rapidly, historical experimental priors become stale faster than the staleness rules account for. Use calibration outputs with additional caution during periods of rapid change.

---

## References

- Brodersen, K.H. et al. (2015). Inferring causal impact using Bayesian structural time-series models. *Annals of Applied Statistics.*
- Gelman, A. & Hill, J. (2007). *Data Analysis Using Regression and Multilevel/Hierarchical Models.* Cambridge University Press.
- Leeflang, P. et al. (2015). Challenges and solutions for marketing in a digital era. *European Management Journal.*
