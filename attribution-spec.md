# Causal Attribution Specification

Technical specification for the causal attribution component of UMM. This document defines the attribution methodology, calibration process, data inputs, output formats, and integration with MMM and incrementality testing.

For methodology context, see [docs/methodology.md](../docs/methodology.md). For terminology, see [docs/glossary.md](../docs/glossary.md).

---

## Overview

Attribution assigns fractional credit for conversions to the marketing touchpoints that contributed to them. Causal attribution differs from rules-based and standard data-driven attribution in one fundamental way: it estimates **incremental** contribution — the share of conversions that would not have occurred without each touchpoint — rather than assigning credit based on path position, frequency, or co-occurrence with conversion.

In UMM, attribution is the campaign-level and touchpoint-level measurement layer. MMM operates at the weekly channel level; attribution operates at the impression and click level, making it the right tool for creative testing, audience analysis, and campaign optimization. Calibration connects the two: attribution weights are regularized toward MMM channel contribution estimates and updated with experimentally measured iROAS, preventing the path-level model from drifting into correlation-grade outputs.

---

## The attribution problem

### Why standard attribution fails

Standard attribution — including rules-based (last touch, first touch, linear) and uncalibrated data-driven attribution — shares a common flaw: it assigns credit to touchpoints based on their presence in conversion paths, not their causal impact.

Consider a user who:
1. Has high purchase intent from a TV campaign running in their market
2. Is subsequently served a retargeting ad
3. Clicks a branded paid search ad
4. Converts

Last-touch attribution gives 100% credit to branded search. Linear attribution splits credit equally across all three. Neither asks: would this user have converted without each touchpoint?

The retargeting ad found a user already close to converting. The branded search ad captured intent that already existed. The TV campaign — invisible in the path — may have been the causal driver. Standard attribution systematically under-credits upper-funnel channels and over-credits lower-funnel, always-on channels that skew toward high-intent users.

### The incrementality bias in path data

Path data is not a random sample of advertising exposures. It reflects the users the ad platform chose to show ads to — users the algorithm predicted were likely to convert. This selection bias means that even sophisticated data-driven attribution models trained on path data learn to predict conversion, not to estimate causal impact. They reproduce the platform's targeting logic, not the true effect of the advertising.

Calibration against experimental ground truth corrects for this bias.

---

## Model structure

### Stage 1: Base path model

The base path model estimates the probability that each touchpoint contributed to conversion, conditioned on the observed path. This uses a Shapley value decomposition over the conversion path:

```
φ_i = Σ_{S ⊆ N\{i}} [|S|!(|N|-|S|-1)! / |N|!] · [v(S ∪ {i}) - v(S)]
```

Where:
- `φ_i` — Shapley value (credit) for touchpoint i
- `N` — set of all touchpoints in the path
- `S` — subset of touchpoints not including i
- `v(S)` — conversion probability given only the touchpoints in subset S (estimated from path data)
- `v(S ∪ {i})` — conversion probability given subset S plus touchpoint i

Shapley values satisfy desirable axioms: efficiency (credits sum to total conversions), symmetry (identical touchpoints get identical credit), and null player (touchpoints that never affect conversion get zero credit).

The conversion probability function `v(S)` is estimated using a gradient-boosted classifier trained on observed path data, with features including:
- Touchpoint presence/absence by channel and position
- Time between touchpoints
- Recency of most recent touchpoint
- Path length
- Device and session features (where available)

### Stage 2: Calibration adjustment

The base Shapley model captures correlation patterns in path data — not causal impact. Stage 2 applies a calibration adjustment that aligns channel-level attribution totals with experimental and MMM-based estimates of true incremental impact.

**Calibration inputs:**

| Input | Source | Frequency |
|---|---|---|
| Channel iROAS estimates | Completed geo-lift and holdout tests | After each test |
| Channel contribution shares | MMM posterior medians | Weekly (rolling) |
| Platform-reported conversions | Ad platform APIs | Daily |

**Calibration weight for channel c:**

```
w_calibrated(c) = α · w_shapley(c)
               + β · w_mmm(c)
               + γ · w_experimental(c)
```

Where:
- `w_shapley(c)` — channel c's share of total Shapley credit from the base model
- `w_mmm(c)` — channel c's contribution share from the MMM posterior median
- `w_experimental(c)` — channel c's iROAS-implied contribution share from geo-lift results
- `α + β + γ = 1` — mixing weights

**Mixing weight assignment:**

Mixing weights are assigned based on data availability and recency:

| Condition | α (Shapley) | β (MMM) | γ (Experimental) |
|---|---|---|---|
| No experimental data available | 0.3 | 0.7 | 0.0 |
| Experimental data >12 months old | 0.3 | 0.5 | 0.2 |
| Experimental data 6–12 months old | 0.25 | 0.4 | 0.35 |
| Experimental data <6 months old | 0.2 | 0.3 | 0.5 |
| Multiple recent experiments | 0.15 | 0.25 | 0.6 |

When multiple experiments exist for a channel (different time periods or geographies), experimental weight `w_experimental(c)` is the precision-weighted average of available results:

```
w_experimental(c) = Σ_k [precision_k · iROAS_k] / Σ_k precision_k
```

Where `precision_k = 1 / CI_width_k²` and `CI_width_k` is the 80% confidence interval width of experiment k.

### Stage 3: Touchpoint-level redistribution

After calibrating channel-level totals, redistribute credits within each channel to individual touchpoints based on position, recency, and frequency features from the base path model:

```
credit(touchpoint_i in channel c) = w_calibrated(c) · [φ_i / Σ_{j in c} φ_j]
```

This preserves the within-channel relative ordering from the Shapley model (which touchpoints within paid social matter most) while correcting the cross-channel totals for incrementality.

---

## Conversion types and attribution windows

### Conversion types

UMM attribution supports multiple conversion types simultaneously. Each conversion type is attributed independently:

| Conversion type | Typical attribution window | Notes |
|---|---|---|
| Direct purchase (e-commerce) | 7–14 day click, 1 day view | Shorter windows for high-frequency categories |
| Lead / form submission | 14–30 day click, 7 day view | Longer consideration cycles |
| App install | 7 day click, 1 day view | Per MMP standard windows |
| In-store visit | 14 day click, 3 day view | Requires offline visit data |
| Brand search lift | 7 day click, 3 day view | Proxy for brand impact |

### Attribution window calibration

Attribution windows should align to experimentally measured time-to-effect. If a geo-lift test shows that paid social lift is concentrated in days 1–5 post-exposure, a 7-day attribution window is appropriate. If TV lift persists for 3–4 weeks, a 28-day window is more appropriate.

Review attribution windows after each completed geo-lift test. Misaligned windows are a common source of over- or under-attribution to specific channels.

### View-through attribution

View-through attribution (crediting impressions that were seen but not clicked) is supported but should be used cautiously. View-through credit should only be applied to channels where:

1. An incrementality test has confirmed the channel drives incremental conversions (not just co-occurrence with converting users)
2. The platform provides verified impression data (not modeled reach)

Default: view-through attribution is **off** for display and social retargeting until experimental confirmation. It is **on** by default for TV and OOH, where click-through is structurally impossible.

---

## Data inputs

### Path data

| Field | Required | Description |
|---|---|---|
| User / device ID | Yes | Consistent identifier across touchpoints (hashed; no PII) |
| Touchpoint timestamp | Yes | Datetime of each impression or click |
| Channel | Yes | From the channel taxonomy in [schemas/channel-taxonomy.json](../schemas/channel-taxonomy.json) |
| Campaign ID | Yes | Platform-native campaign identifier |
| Ad format | Recommended | Video, display, search, social feed, etc. |
| Conversion flag | Yes | 1 if a conversion event occurred, 0 otherwise |
| Conversion timestamp | Yes (if converted) | Datetime of conversion event |
| Conversion value | Yes (if revenue) | Revenue value of the conversion |
| Platform | Yes | Ad platform (Google, Meta, TikTok, etc.) |

### Identity resolution

Cross-device and cross-platform identity resolution is required before path construction. Without it, the same user's touchpoints across desktop, mobile web, and app appear as separate paths, understating path length and distorting channel sequencing.

Minimum identity resolution requirements:
- First-party login-based matching (where available)
- Probabilistic cross-device graph (where login not available)
- Documented match rate: report the % of conversions with fully resolved cross-device paths

Privacy constraints:
- No use of third-party cookies or device fingerprinting
- All user identifiers must be hashed before entering the attribution pipeline
- Comply with the applicable privacy regulations for each market (GDPR, CCPA, etc.)

### Offline conversion data

Where in-store purchases or offline leads are the primary conversion event:
- Import offline conversions via CRM match or point-of-sale data
- Match offline conversions to online paths using hashed email, phone, or loyalty ID
- Report offline match rate; unmatched offline conversions are attributed to baseline or MMM

---

## Outputs

### Channel-level outputs

| Output | Definition | Reporting cadence |
|---|---|---|
| Attributed conversions (calibrated) | Total conversions credited to each channel, calibrated | Weekly |
| Attributed revenue (calibrated) | Total revenue credited to each channel, calibrated | Weekly |
| iROAS (calibrated) | Attributed revenue / spend, calibrated | Weekly |
| Calibration adjustment factor | Ratio of calibrated to raw Shapley attribution per channel | Monthly |
| Attribution window sensitivity | How attributed conversions change across window lengths | Quarterly |

### Campaign-level outputs

| Output | Definition |
|---|---|
| Campaign incrementality score | Calibrated attributed conversions / raw Shapley attributed conversions (ratio closer to 1 = more incremental) |
| Campaign iROAS | Calibrated attributed revenue / campaign spend |
| Audience incrementality index | Incrementality score by audience segment (where experimental data supports) |
| Creative contribution | Relative Shapley credit by creative variant within a campaign |

### Touchpoint-level outputs

| Output | Definition |
|---|---|
| Touchpoint credit | Fractional calibrated credit per impression/click |
| Position contribution | Credit by path position (first, mid, last) per channel |
| Time-decay profile | How credit is distributed across days since first touchpoint |

---

## Validation

### Calibration convergence check

After each calibration update, verify that channel-level calibrated attribution totals are consistent with MMM and experimental inputs:

| Check | Threshold | Action if failed |
|---|---|---|
| Calibrated iROAS vs MMM iROAS | Within 20% | Review mixing weights; check data alignment |
| Calibrated iROAS vs experimental iROAS | Within 20% | Check experimental data recency; review mixing weights |
| Total calibrated conversions vs platform-reported | Within 10% | Review path data completeness; check attribution window |
| Attribution sum (all channels) = total conversions | Within 1% | Shapley efficiency property — investigate if violated |

### Incrementality ratio monitoring

Track the **incrementality ratio** per channel monthly:

```
incrementality_ratio(c) = calibrated_attribution(c) / raw_shapley_attribution(c)
```

- Ratio < 0.7: channel is being heavily downweighted by calibration — review experimental data; consider whether the channel is genuinely low-incrementality or whether the test was underpowered
- Ratio 0.7–1.3: within normal calibration range
- Ratio > 1.3: channel is being upweighted — typically upper-funnel channels (TV, OOH) that are under-represented in path data

Document and review any channel with a ratio outside 0.7–1.3.

### Holdout validation

Quarterly, run a user-level holdout test for one digital channel (CRM or retargeting, where holdouts are feasible). Compare:
- Incrementality measured by holdout test
- Calibrated attribution iROAS for the same channel and period

This serves as an independent validation of the calibration process. If the holdout result differs from calibrated attribution by >30%, recalibrate.

---

## Limitations

### What calibrated attribution cannot do

- **Estimate baseline conversions**: Attribution credits touchpoints; it cannot estimate how many conversions would have occurred with zero marketing. That is MMM's role.
- **Attribute offline-only channels at the touchpoint level**: TV, OOH, and radio cannot be matched to individual conversion paths. Their contributions are estimated by MMM and incorporated into attribution via calibration weights — they receive calibrated channel-level credit but not touchpoint-level credit.
- **Fully correct for selection bias in targeting**: Calibration reduces selection bias; it does not eliminate it. Channels with extreme targeting (retargeting only high-intent users, branded search capturing in-market users) will still show some correlation-driven credit even after calibration.
- **Operate without path data**: Attribution requires impression and click data at the user or device level. Categories with long purchase cycles, heavy offline behavior, or low digital path coverage will have noisier attribution outputs. MMM is the more reliable measurement layer in these cases.

### Privacy and signal loss

Signal loss from cookie deprecation, iOS ATT, and increasing use of ad blockers reduces path coverage over time. Attribution outputs should be interpreted in the context of their path coverage rate — the percentage of conversions with at least one matched digital touchpoint. Coverage rates below 50% indicate that attribution outputs are unreliable as a primary measurement layer; MMM should be weighted more heavily.

Monitor path coverage rate monthly. Flag any month-over-month decline > 5 percentage points.

---

## Integration with UMM

### Receiving calibration from MMM and experiments

The attribution model is recalibrated:
- Monthly (scheduled): using updated MMM channel contribution shares
- After each completed geo-lift or holdout test: incorporating the new experimental iROAS estimate

Calibration updates are logged with timestamps, mixing weights used, and the resulting channel adjustment factors — enabling audit of how attribution has evolved over time.

### Feeding campaign optimization

Calibrated attribution outputs feed campaign-level optimization:
- **Budget allocation**: Calibrated iROAS by channel informs the budget recommendation alongside MMM budget curves
- **Audience suppression**: Users with high baseline conversion probability (would convert without the ad) can be identified and suppressed from retargeting audiences — reducing wasted spend on non-incremental users
- **Creative testing**: Within-channel Shapley credit by creative variant identifies which creatives drive more path contribution, informing creative rotation

### Reporting hierarchy

In UMM, measurement outputs are reported in a defined hierarchy of reliability:

| Method | Reliability | Best for |
|---|---|---|
| Geo-lift test | Highest (experimental ground truth) | Channel-level iROAS validation |
| Calibrated MMM | High (calibrated observational) | Portfolio allocation, long-run trends |
| Calibrated attribution | Medium (calibrated path model) | Campaign and creative optimization |
| Raw Shapley attribution | Low (correlation-grade) | Path analysis only; not for budget decisions |
| Platform-reported ROAS | Not recommended | No use in UMM decision-making |

Never use platform-reported ROAS or uncalibrated attribution as inputs to budget decisions.

---

## References

- Shapley, L.S. (1953). A value for n-person games. *Contributions to the Theory of Games.*
- Dalessandro, B. et al. (2012). Causally Motivated Attribution for Online Advertising. *ACM ADKDD.*
- Gordon, B. et al. (2019). A Comparison of Approaches to Advertising Measurement. *Marketing Science.*
- Lai, L. et al. (2022). Towards Calibrated and Efficient Attribution for Large-Scale Advertising. *KDD.*
