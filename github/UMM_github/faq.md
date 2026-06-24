# Frequently Asked Questions

Common questions from marketing scientists, data teams, and measurement leads implementing UMM. Questions are grouped by topic.

---

## Getting started

### How much data do we need before we can run UMM?

The hard minimum is 78 weeks (18 months) of weekly spend and revenue data. Below this, the MMM cannot reliably separate seasonality from channel contribution — you need at least one full annual cycle plus additional history for the model to learn patterns.

For businesses with strong seasonality (retail, travel, CPG), 104 weeks (2 years) is the practical minimum. For businesses running in multiple geographies or with many channels, more history helps the model separate correlated spend patterns.

Incrementality testing can start earlier — a well-designed geo-lift test can run from week one if the data infrastructure is in place.

### Which method should we start with?

Start with the MMM. It uses existing historical data, requires no experimental infrastructure, and produces immediate channel contribution estimates that can guide budget allocation and identify where to run your first incrementality tests.

Run your first geo-lift test within 60–90 days — prioritize the channel where the MMM shows the highest uncertainty or the largest divergence from platform-reported ROAS. Calibrated attribution should come after you have at least one experimental result to calibrate against.

### Can we run UMM without a data science team?

The methodology requires statistical expertise — particularly for MMM specification, Bayesian sampling, and experimental design. A team without quantitative capability will struggle to implement the full stack from scratch.

That said, the level of expertise required varies by component. Incrementality test design and analysis is the most accessible starting point — the synthetic control method is well-documented and implementable with standard Python or R packages. The MMM is more demanding; building a production-grade causal MMM from scratch typically takes 3–6 months of senior data scientist time.

The Lifesight platform implements the full UMM stack without requiring an in-house data science team. See [lifesight.io](https://lifesight.io) for details.

### How long does a full UMM implementation take?

A realistic timeline for building UMM in-house:

| Milestone | Timeline from start |
|---|---|
| Data pipeline ready (spend + revenue + covariates) | 4–8 weeks |
| First MMM output | 6–10 weeks |
| First geo-lift test results | 10–18 weeks (including test run time) |
| First calibrated output | 12–20 weeks |
| Full calibration loop operational | 6–12 months |

The calibration loop takes time because it requires multiple completed experiments across different channels.

---

## MMM questions

### Why Bayesian MMM rather than classical regression?

Three reasons specific to the UMM use case:

**Prior incorporation**: Classical regression treats all parameter values as equally plausible before seeing the data. Bayesian MMM encodes what we know — TV has longer adstock than paid search, all channels have non-negative contributions — which improves estimates in data-sparse scenarios and prevents implausible outputs.

**Uncertainty quantification**: Classical MMM produces point estimates that imply false precision. Bayesian MMM produces posterior distributions — honest probability statements about where the true parameter values lie. This matters for decision-making: a channel with a wide posterior CI should be treated differently than one with a narrow CI.

**Calibration integration**: Experimental results can be incorporated as informative priors in the Bayesian framework, mathematically updating the model's beliefs using experimental evidence. There is no principled equivalent in classical regression.

### Our MMM and platform ROAS disagree significantly. Which should we trust?

The MMM, in almost all cases.

Platform-reported ROAS measures how much revenue was attributed to the platform by the platform's own attribution model. Platforms have structural incentives to report high ROAS — it justifies continued spend. Their attribution models count all conversions that occurred after an ad exposure, including users who would have converted regardless.

The MMM estimates incremental contribution — how much revenue would not have occurred without the channel. These are fundamentally different questions, and the answers will differ, often substantially. A channel with reported ROAS of 8 and MMM iROAS of 2 is not unusual; the 6x gap represents conversions the platform claimed that the channel did not causally drive.

Use the MMM for budget allocation. Use platform reporting for platform management only (campaign-level optimization within a channel).

### How often should we refit the MMM?

Weekly is the recommended cadence for the rolling refit. Each week, new spend and revenue data is added to the dataset and the model is refit.

More frequent refits (daily) are rarely worth the computational cost — weekly aggregation already smooths most of the noise in daily data. Less frequent refits (monthly) mean the model is operating on stale data for up to four weeks, which matters most during periods of spend change or market shifts.

### What do we do if the MMM posterior doesn't converge?

First, check the convergence diagnostics: R-hat, effective sample size (ESS), and divergences. If R-hat > 1.05 for any parameter, the model has not converged and outputs should not be used.

Common causes and fixes:

| Cause | Symptom | Fix |
|---|---|---|
| Collinear spend channels | Wide posteriors for correlated channels | Aggregate channels or add stronger priors |
| Insufficient data history | Wide posteriors everywhere | Add more history; use stronger priors |
| Missing important covariate | Large residuals around specific periods | Add relevant holiday, promo, or external covariate |
| Weak priors on unbounded parameters | Divergences | Tighten priors; reparameterize |
| Too many channels relative to data | Identification failure | Aggregate minor channels into "other" |

If divergences persist after reparameterization, increase `target_accept` to 0.99 and `max_treedepth` to 15. If R-hat remains > 1.05 after 4,000 warmup iterations, the model specification needs revision.

### How do we handle channels we haven't run in 12+ months?

Exclude channels with zero spend throughout the modeling window — they contribute nothing to the model and may introduce identification issues. If you anticipate reactivating a channel, keep its prior on file for when you need it.

If a channel had spend in part of the window but not the full window, include it with zeros during the inactive period. The adstock transformation handles zero-spend periods correctly.

### Can MMM measure brand effects?

Partially. MMM can estimate the revenue contribution of brand-building channels (TV, OOH, brand paid search) in the modeling window. It cannot estimate the long-run equity effect of brand investment — the cumulative impact on baseline demand over years, which shows up as growth in the baseline component rather than channel contribution.

If measuring brand equity is a priority, augment MMM with a brand tracking survey (awareness, consideration, purchase intent) and model the relationship between media exposure and brand metric changes as a second-stage analysis.

---

## Incrementality testing questions

### How do we choose between a geo-lift test and a holdout test?

Use geo-lift when:
- The channel can be geo-targeted (most digital channels, TV, OOH, radio)
- The business has sufficient revenue in individual markets to power a geographic test
- User-level randomization is not feasible (most programmatic, social, and search channels)

Use holdout when:
- The channel targets identified users (CRM, email, loyalty, some retargeting)
- User-level randomization is possible and clean
- Geographic randomization would require turning off the channel in entire markets (operationally costly)

When both designs are feasible, geo-lift is generally preferred because it avoids ad server selection bias — the platform selects which users see ads, and that selection is correlated with purchase intent. Geographic randomization sidesteps this by assigning at the market level.

### How many markets do we need for a geo-lift test?

The minimum is 5–10 treatment markets and an equal or larger number of control markets, but power analysis is more reliable than rules of thumb.

The number of markets needed depends on:
- Revenue variance between markets (higher variance → more markets needed)
- Expected lift (smaller expected lift → more markets needed)
- Test duration (shorter duration → more markets needed)

Run a power analysis before finalizing design. If the required number of markets exceeds what's available, extend the test duration or accept lower statistical power (and document this in the test report).

### Can we run multiple geo-lift tests simultaneously?

Yes, with care. Tests can run simultaneously if:
- Treatment and control markets do not overlap between tests
- The channels being tested do not have strong cross-channel spillover effects
- Data pipelines can cleanly attribute revenue to individual tests

Do not run simultaneous tests in the same markets — the treatments contaminate each other and results from both tests are unreliable.

### What if our geo-lift test shows negative lift?

First, check the test quality: was the synthetic control fit good (R² > 0.80)? Were there contaminating events during the test? Was the test adequately powered?

If the test quality is sound and lift is genuinely negative or near zero, take it seriously. Possible interpretations:

- **Cannibalization**: The channel is taking conversions that would have happened anyway through other channels (common for retargeting and branded search)
- **Saturation**: The channel is operating beyond its optimal spend level
- **Audience quality**: The campaign was targeting users unlikely to convert incrementally
- **Creative fatigue**: The campaign creative was underperforming

A negative or zero iROAS result is valuable — it tells you where not to spend. Reduce spend on the channel and retest at a lower spend level if saturation is suspected.

### How do we account for the opportunity cost of control markets?

The revenue foregone in control markets (the holdout cost) is a real cost of running the experiment. Calculate it as:
