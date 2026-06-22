# Glossary

Complete terminology reference for Unified Marketing Measurement. Terms are defined as they are used in this framework — some have broader definitions in other contexts.

---

## A

**Adstock**
A transformation applied to spend data that models the carryover effect of advertising — the idea that advertising exposure in week T continues to influence behavior in weeks T+1, T+2, and so on. Represented as a geometric decay:

```
adstock(t) = spend(t) + λ · adstock(t-1)
```

Where `λ` is the decay rate (0 = no carryover, 1 = permanent effect). Decay rates vary by channel and are estimated from data, with priors informed by media type.

**Attribution**
The process of assigning credit for a conversion (purchase, sign-up, etc.) to the marketing touchpoints that preceded it. See also: *causal attribution*, *rules-based attribution*, *data-driven attribution*.

**Attribution window**
The lookback period over which touchpoints are considered when attributing a conversion. Common windows: 7-day click, 28-day click, 1-day view. Shorter windows exclude upper-funnel influence; longer windows inflate attribution counts. UMM uses calibrated windows aligned to experimentally measured time-to-effect.

---

## B

**Baseline revenue**
The revenue that would occur with zero marketing spend — driven by brand equity, organic demand, distribution, and other non-media factors. Estimating baseline accurately is one of the most consequential modeling choices in MMM: a high baseline estimate compresses channel contribution estimates, and vice versa.

**Bayesian inference**
A statistical framework that combines prior beliefs (encoded as probability distributions) with observed data to produce posterior distributions over model parameters. UMM uses Bayesian inference for MMM because it enables prior incorporation, honest uncertainty quantification, and principled integration of experimental results as informative priors.

**Budget response curve**
A channel-level curve showing expected incremental revenue at each spend level. Derived from the saturation transformation in MMM. The shape of the curve tells you: where diminishing returns begin, the spend level at which iROAS equals 1 (breakeven), and the optimal spend given a total budget constraint. Used directly in budget optimization.

---

## C

**Calibration**
The process of aligning the outputs of multiple measurement methods — MMM, incrementality testing, and attribution — so they produce consistent estimates of channel impact. In UMM, calibration works in both directions: experimental results update MMM priors; MMM outputs regularize attribution weights. See [calibration-spec.md](https://github.com/lifesight/unified-marketing-measurement/blob/main/calibration-spec.md).

**Calibration drift**
When MMM channel contribution estimates diverge from experimentally measured iROAS beyond acceptable thresholds. Drift indicates the model has moved away from ground truth and requires prior updates, covariate review, or re-specification. Yellow flag: >20% divergence. Red flag: >40% divergence.

**Causal attribution**
Attribution that estimates the incremental contribution of touchpoints — what would not have happened without the exposure — rather than assigning credit based on path position or frequency. Calibrated using incrementality test results and MMM channel contributions.

**Causal MMM**
A variant of Marketing Mix Modeling that uses Bayesian structural time-series models with carefully specified priors to estimate incremental channel contributions, rather than maximizing fit to observed data (which captures correlation, not causation). The priors encode domain knowledge about channel response functions; experimental results update those priors over time.

**Confidence interval / Credible interval**
A range of values within which the true parameter is estimated to lie with a given probability. In frequentist statistics: confidence interval. In Bayesian statistics: credible interval (more directly interpretable as "there is an X% probability the true value is in this range"). UMM reports 80% credible intervals as the standard uncertainty range for decision-making.

**Control market / Control group**
In a geo-lift test, the markets that do not receive the treatment (the campaign being tested). Their revenue trajectory during the test period provides the counterfactual — what would have happened in treatment markets without the campaign. See also: *synthetic control*.

**Conversion**
A business outcome used as the measurement target — purchase, subscription, lead, in-store visit, or other defined action. UMM can be applied to any conversion metric; the methodology does not assume e-commerce.

**Counterfactual**
What would have happened in the absence of a marketing intervention. Estimating the counterfactual is the central challenge in causal measurement. Geo-lift tests estimate the counterfactual through control markets; MMM estimates it through the modeled baseline.

---

## D

**Data-driven attribution (DDA)**
Attribution that uses machine learning to weight touchpoints based on observed path data, rather than fixed rules. Standard DDA (as implemented by Google, Meta, and others) estimates correlation, not causation — it does not adjust for the fact that users who were already likely to convert appear in many paths regardless of whether the touchpoints caused the conversion. UMM's causal attribution layer calibrates DDA outputs to correct for this bias.

**Decay rate (λ)**
The parameter in the adstock transformation that controls how quickly advertising effects diminish over time. A decay rate of 0.8 means 80% of the adstock effect carries over to the next period; a rate of 0.2 means 20% carries over. Estimated per channel with priors informed by media type.

**Difference-in-differences (DiD)**
A method for estimating causal effects by comparing the change in outcomes in a treatment group versus a control group over time. Used in geo-lift analysis as an alternative to synthetic control when treatment/control market groups are large enough to average out pre-existing differences.

**Diminishing returns**
The empirical pattern in which each additional dollar of spend on a channel produces less incremental revenue than the previous dollar. Modeled in MMM through the saturation transformation. The point at which marginal iROAS falls below 1 (spending $1 to generate <$1 of incremental revenue) defines the theoretical overspend threshold.

---

## E

**Experiment calendar**
A planned schedule of incrementality tests across channels and campaigns, prioritized by MMM uncertainty estimates. Maintains at least one active test at all times; queues future tests; integrates completed test results into the calibration loop.

**External factors / covariates**
Variables included in the MMM that are not marketing spend but influence revenue — macroeconomic indicators, competitor activity, distribution changes, weather, pricing, and so on. Failure to include relevant external factors causes their effects to be misattributed to media channels, inflating contribution estimates.

---

## F

**Flighting**
The practice of concentrating advertising spend in defined periods rather than running continuously. Flighting creates natural variation in spend that improves MMM identifiability — the model can more precisely estimate channel contribution when spend levels vary over time. Continuously flat spend makes MMM estimation harder.

---

## G

**Geo-lift test**
A geographically randomized controlled experiment that measures the incremental impact of a marketing channel or campaign. Treatment markets receive the campaign; control markets do not. The revenue difference between matched treatment and control markets estimates the causal, incremental impact. The primary experimental design in UMM. See [incrementality-spec.md](../specs/incrementality-spec.md).

**Ghost bid / holdout bid**
A technique used by some ad platforms to create user-level control groups: the platform identifies users who would have been served an ad, runs an auction, wins, but serves a blank (ghost) ad rather than a real one. Allows measurement of ad impact at the user level without geographic holdouts. Less common than geo-lift due to platform support constraints.

---

## H

**Half-saturation point (K)**
In the Hill saturation function, the spend level at which a channel achieves 50% of its maximum possible effect. A key parameter in the budget response curve — spend below K is in the steep portion of the curve (high marginal iROAS); spend above K is in the flat portion (diminishing returns). Estimated per channel with priors informed by data.

**Hamiltonian Monte Carlo (HMC)**
A Markov chain Monte Carlo (MCMC) algorithm used for Bayesian posterior sampling. More efficient than standard MCMC for high-dimensional continuous parameter spaces, which makes it well-suited to MMM. Implemented in Stan (via CmdStan or RStan) and PyMC.

**Holdout test**
An incrementality test design where a randomly selected subset of users or accounts is excluded from a campaign for a defined period. Used when geo-level control is infeasible. Most applicable to CRM campaigns, email, loyalty programs, and retargeting where user-level assignment is possible.

**Holdout validation**
A model validation technique where a portion of the historical data is withheld during model fitting and used to evaluate out-of-sample prediction accuracy. In UMM, holdout validation checks whether the MMM can accurately predict revenue in unseen time periods — the primary test of whether the model has learned genuine patterns rather than overfitting.

---

## I

**iRevenue**
Incremental revenue — the additional revenue that occurred as a direct causal result of a marketing investment, that would not have occurred without it. The top-line output of UMM. Distinguished from attributed revenue (which may include revenue that would have occurred anyway) and platform-reported revenue (which is almost always overstated due to correlation bias).

**iROAS (Incremental Return on Ad Spend)**
Incremental revenue generated per dollar of media spend. The primary efficiency metric in UMM.

```
iROAS = iRevenue / Spend
```

An iROAS of 3 means each dollar spent generated $3 of revenue that would not have occurred otherwise. iROAS < 1 means spending is destroying value on the margin. Distinguished from reported ROAS, which includes revenue that would have occurred without the ad.

**Identifiability**
The property of a statistical model where the data is sufficient to estimate the model parameters uniquely. In MMM, identifiability is a persistent challenge: if two channels always move together (collinearity), the model cannot separate their contributions. Solutions include: stronger priors, experimental data, channel-level spend variation, and longer data history.

**Incrementality**
The causal, marginal impact of a marketing action — what would not have happened without it. A channel is incremental if removing it would reduce total revenue. A channel with high reported ROAS but low incrementality is capturing existing intent rather than generating new demand.

**Incrementality testing**
The experimental practice of measuring true incremental impact through controlled experiments (geo-lift tests, holdout tests). The gold standard for estimating iROAS. Used in UMM both to measure channel performance directly and to calibrate MMM model priors. See [incrementality-spec.md](../specs/incrementality-spec.md).

**Informative prior**
A Bayesian prior distribution that encodes meaningful domain knowledge, as opposed to a weakly informative or flat prior. In UMM, informative priors encode knowledge about channel response functions (e.g., TV has higher adstock carryover than paid search) and are updated over time with experimental results.

---

## L

**Lift**
The incremental effect of a marketing action — typically expressed as a percentage increase over the counterfactual. A geo-lift test measures revenue lift: the percentage by which treatment market revenue exceeded what the synthetic control predicted.

---

## M

**Market matching**
The process of identifying control markets that most closely resemble treatment markets prior to a geo-lift test. Matching is done on pre-test revenue trajectories, seasonality patterns, volume, and demographic characteristics. Good matching minimizes the risk that post-test differences reflect pre-existing divergence rather than treatment effect.

**Marketing Mix Modeling (MMM)**
A statistical modeling approach that uses historical data on marketing spend and business outcomes to estimate each channel's contribution to those outcomes. In UMM, the causal variant uses Bayesian inference and is continuously calibrated against experimental results. See [mmm-spec.md](../specs/mmm-spec.md).

**Minimum detectable effect (MDE)**
The smallest true effect size that a given experimental design can reliably detect, given the planned duration, market variance, sample size, and significance threshold. A power analysis calculates MDE before a test runs — if the MDE is larger than the expected true effect, the test should be redesigned.

**Model prior**
In Bayesian statistics, a probability distribution representing beliefs about a parameter before observing data. In UMM MMM, priors encode domain knowledge about channel response functions and are updated using experimental results. Well-specified priors improve model accuracy in data-sparse situations and prevent implausible parameter estimates.

---

## O

**Opportunity cost**
In the context of incrementality testing, the revenue foregone in control markets during a geo-lift test (the control group does not receive the campaign). A real cost of experimental measurement that must be weighed against the value of the information gained.

---

## P

**Post-period analysis**
An analysis of the holdout period after a geo-lift test ends, measuring how quickly the treatment effect dissipates. If treatment markets continue to outperform control markets after the campaign stops, this indicates brand-building or purchase-cycle effects that persist beyond the campaign window.

**Power analysis**
A calculation determining the probability of detecting a true effect of a given size, given sample size, variance, and significance threshold. Run before every geo-lift test to ensure the design can answer the question it sets out to answer.

**Posterior distribution**
In Bayesian inference, the probability distribution over model parameters after observing data. Combines the prior distribution with the likelihood of the observed data. UMM reports posterior medians as point estimates and 80% credible intervals as uncertainty ranges.

---

## R

**Response curve**
See *budget response curve*.

**ROAS (Return on Ad Spend)**
Total revenue attributed to a channel divided by spend on that channel. As reported by ad platforms, ROAS includes revenue that would have occurred without the ad — making it an overestimate of true incremental efficiency. UMM replaces ROAS with iROAS as the primary efficiency metric.

**Rules-based attribution**
Attribution that assigns credit according to fixed rules: first touch (100% to first touchpoint), last touch (100% to last touchpoint), linear (equal credit to all touchpoints), time decay (more credit to touchpoints closer to conversion). All rules-based models are arbitrary and do not estimate causal impact.

---

## S

**Saturation**
The diminishing returns pattern in channel response: the first dollar of spend generates the most incremental revenue; each additional dollar generates less. Modeled in MMM through the saturation transformation (Hill function or Michaelis-Menten). The saturation curve is the foundation of budget optimization — it tells you where the next dollar generates the most and least incremental return.

**Spend decomposition**
The breakdown of total marketing spend by channel, showing how much is allocated to each channel and how that compares to the optimal allocation implied by budget response curves.

**Synthetic control**
A method for constructing a counterfactual by computing a weighted combination of control markets that most closely matches the treatment market's pre-test trajectory. More precise than simple difference-in-differences when the number of treatment markets is small. Used in UMM geo-lift tests by default. See [incrementality-spec.md](../specs/incrementality-spec.md).

---

## T

**Time-to-effect**
The lag between advertising exposure and measured impact on the outcome metric. Varies by channel and purchase cycle. Short-cycle categories (FMCG, app installs) may show effects within days; long-cycle categories (auto, financial services, B2B) may show effects over weeks or months. Time-to-effect informs adstock decay rate priors and experiment duration requirements.

**Treatment market / Treatment group**
In a geo-lift test, the markets that receive the campaign being tested. Their revenue during the test is compared to what the synthetic control predicts would have happened without the campaign. See also: *control market*.

---

## U

**Unified Marketing Measurement (UMM)**
The framework documented in this repository. A system that combines causal MMM, incrementality testing, and causal attribution into a single calibrated measurement stack, producing one consistent view of incremental channel impact across all marketing investment. See [methodology.md](methodology.md).

---

## V

**Variance**
In the context of geo-lift tests, the week-to-week variability in market revenue. High variance markets require longer tests or larger market sets to achieve adequate statistical power. Accounted for in power analysis before test design is finalized.

---

## W

**Weakly informative prior**
A Bayesian prior that rules out physically impossible or absurd parameter values while remaining broadly agnostic. Used in UMM when domain knowledge about a parameter is limited. Provides regularization without imposing strong beliefs.
