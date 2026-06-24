Frequently Asked Questions
Common questions from marketing scientists, data teams, and measurement leads implementing UMM. Questions are grouped by topic.


Getting started
How much data do we need before we can run UMM?
The hard minimum is 78 weeks (18 months) of weekly spend and revenue data. Below this, the MMM cannot reliably separate seasonality from channel contribution — you need at least one full annual cycle plus additional history for the model to learn patterns.

For businesses with strong seasonality (retail, travel, CPG), 104 weeks (2 years) is the practical minimum. For businesses running in multiple geographies or with many channels, more history helps the model separate correlated spend patterns.

Incrementality testing can start earlier — a well-designed geo-lift test can run from week one if the data infrastructure is in place.
Which method should we start with?
Start with the MMM. It uses existing historical data, requires no experimental infrastructure, and produces immediate channel contribution estimates that can guide budget allocation and identify where to run your first incrementality tests.

Run your first geo-lift test within 60–90 days — prioritize the channel where the MMM shows the highest uncertainty or the largest divergence from platform-reported ROAS. Calibrated attribution should come after you have at least one experimental result to calibrate against.
Can we run UMM without a data science team?
The methodology requires statistical expertise — particularly for MMM specification, Bayesian sampling, and experimental design. A team without quantitative capability will struggle to implement the full stack from scratch.

That said, the level of expertise required varies by component. Incrementality test design and analysis is the most accessible starting point — the synthetic control method is well-documented and implementable with standard Python or R packages. The MMM is more demanding; building a production-grade causal MMM from scratch typically takes 3–6 months of senior data scientist time.

The Lifesight platform implements the full UMM stack without requiring an in-house data science team. See lifesight.io for details.
How long does a full UMM implementation take?
A realistic timeline for building UMM in-house:

Milestone
Timeline from start
Data pipeline ready (spend + revenue + covariates)
4–8 weeks
First MMM output
6–10 weeks
First geo-lift test results
10–18 weeks (including test run time)
First calibrated output
12–20 weeks
Full calibration loop operational
6–12 months


The calibration loop takes time because it requires multiple completed experiments across different channels.


MMM questions
Why Bayesian MMM rather than classical regression?
Three reasons specific to the UMM use case:

Prior incorporation: Classical regression treats all parameter values as equally plausible before seeing the data. Bayesian MMM encodes what we know — TV has longer adstock than paid search, all channels have non-negative contributions — which improves estimates in data-sparse scenarios and prevents implausible outputs.

Uncertainty quantification: Classical MMM produces point estimates that imply false precision. Bayesian MMM produces posterior distributions — honest probability statements about where the true parameter values lie. This matters for decision-making: a channel with a wide posterior CI should be treated differently than one with a narrow CI.

Calibration integration: Experimental results can be incorporated as informative priors in the Bayesian framework, mathematically updating the model's beliefs using experimental evidence. There is no principled equivalent in classical regression.
Our MMM and platform ROAS disagree significantly. Which should we trust?
The MMM, in almost all cases.

Platform-reported ROAS measures how much revenue was attributed to the platform by the platform's own attribution model. Platforms have structural incentives to report high ROAS — it justifies continued spend. Their attribution models count all conversions that occurred after an ad exposure, including users who would have converted regardless.

The MMM estimates incremental contribution — how much revenue would not have occurred without the channel. These are fundamentally different questions, and the answers will differ, often substantially. A channel with reported ROAS of 8 and MMM iROAS of 2 is not unusual; the 6x gap represents conversions the platform claimed that the channel did not causally drive.

Use the MMM for budget allocation. Use platform reporting for platform management only (campaign-level optimization within a channel).
How often should we refit the MMM?
Weekly is the recommended cadence for the rolling refit. Each week, new spend and revenue data is added to the dataset and the model is refit.

More frequent refits (daily) are rarely worth the computational cost — weekly aggregation already smooths most of the noise in daily data. Less frequent refits (monthly) mean the model is operating on stale data for up to four weeks, which matters most during periods of spend change or market shifts.
What do we do if the MMM posterior doesn't converge?
First, check the convergence diagnostics: R-hat, effective sample size (ESS), and divergences. If R-hat > 1.05 for any parameter, the model has not converged and outputs should not be used.

Common causes and fixes:

Cause
Symptom
Fix
Collinear spend channels
Wide posteriors for correlated channels
Aggregate channels or add stronger priors
Insufficient data history
Wide posteriors everywhere
Add more history; use stronger priors
Missing important covariate
Large residuals around specific periods
Add relevant holiday, promo, or external covariate
Weak priors on unbounded parameters
Divergences
Tighten priors; reparameterize
Too many channels relative to data
Identification failure
Aggregate minor channels into "other"


If divergences persist after reparameterization, increase target_accept to 0.99 and max_treedepth to 15. If R-hat remains > 1.05 after 4,000 warmup iterations, the model specification needs revision.
How do we handle channels we haven't run in 12+ months?
Exclude channels with zero spend throughout the modeling window — they contribute nothing to the model and may introduce identification issues. If you anticipate reactivating a channel, keep its prior on file for when you need it.

If a channel had spend in part of the window but not the full window, include it with zeros during the inactive period. The adstock transformation handles zero-spend periods correctly.
Can MMM measure brand effects?
Partially. MMM can estimate the revenue contribution of brand-building channels (TV, OOH, brand paid search) in the modeling window. It cannot estimate the long-run equity effect of brand investment — the cumulative impact on baseline demand over years, which shows up as growth in the baseline component rather than channel contribution.

If measuring brand equity is a priority, augment MMM with a brand tracking survey (awareness, consideration, purchase intent) and model the relationship between media exposure and brand metric changes as a second-stage analysis.


Incrementality testing questions
How do we choose between a geo-lift test and a holdout test?
Use geo-lift when:

The channel can be geo-targeted (most digital channels, TV, OOH, radio)
The business has sufficient revenue in individual markets to power a geographic test
User-level randomization is not feasible (most programmatic, social, and search channels)

Use holdout when:

The channel targets identified users (CRM, email, loyalty, some retargeting)
User-level randomization is possible and clean
Geographic randomization would require turning off the channel in entire markets (operationally costly)

When both designs are feasible, geo-lift is generally preferred because it avoids ad server selection bias — the platform selects which users see ads, and that selection is correlated with purchase intent. Geographic randomization sidesteps this by assigning at the market level.
How many markets do we need for a geo-lift test?
The minimum is 5–10 treatment markets and an equal or larger number of control markets, but power analysis is more reliable than rules of thumb.

The number of markets needed depends on:

Revenue variance between markets (higher variance → more markets needed)
Expected lift (smaller expected lift → more markets needed)
Test duration (shorter duration → more markets needed)

Run a power analysis before finalizing design. If the required number of markets exceeds what's available, extend the test duration or accept lower statistical power (and document this in the test report).
Can we run multiple geo-lift tests simultaneously?
Yes, with care. Tests can run simultaneously if:

Treatment and control markets do not overlap between tests
The channels being tested do not have strong cross-channel spillover effects
Data pipelines can cleanly attribute revenue to individual tests

Do not run simultaneous tests in the same markets — the treatments contaminate each other and results from both tests are unreliable.
What if our geo-lift test shows negative lift?
First, check the test quality: was the synthetic control fit good (R² > 0.80)? Were there contaminating events during the test? Was the test adequately powered?

If the test quality is sound and lift is genuinely negative or near zero, take it seriously. Possible interpretations:

Cannibalization: The channel is taking conversions that would have happened anyway through other channels (common for retargeting and branded search)
Saturation: The channel is operating beyond its optimal spend level
Audience quality: The campaign was targeting users unlikely to convert incrementally
Creative fatigue: The campaign creative was underperforming

A negative or zero iROAS result is valuable — it tells you where not to spend. Reduce spend on the channel and retest at a lower spend level if saturation is suspected.
How do we account for the opportunity cost of control markets?
The revenue foregone in control markets (the holdout cost) is a real cost of running the experiment. Calculate it as:

holdout_cost = expected_revenue_lift_per_market · n_control_markets · test_duration_weeks

For most tests, holdout cost is small relative to the value of the information gained. If holdout cost is material, weigh it against the expected reduction in wasted spend from the insight. A test that costs $50K in holdout revenue and reveals that a $2M/month channel has iROAS of 0.5 pays for itself immediately.


Attribution questions
Why does calibrated attribution give different numbers than what our ad platforms report?
Three reasons:

Different question: Platform attribution answers "how much revenue was attributed to this platform?" Calibrated attribution answers "how much revenue did this platform causally drive?" The first is always higher than the second.

Different methodology: Platforms use last-touch or proprietary data-driven attribution that assigns credit based on path presence, not incremental contribution. Calibrated attribution uses Shapley values corrected for selection bias using experimental evidence.

Cross-platform deduplication: Each platform counts conversions independently. A single conversion may be counted by Google, Meta, and TikTok simultaneously. Calibrated attribution deduplicates across platforms — total attributed conversions equal total actual conversions.

The sum of all platform-reported conversions will almost always exceed your actual conversion count. Calibrated attribution will not.
Our attribution shows branded search is low-incrementality. Should we turn it off?
Probably reduce it, but rarely eliminate it entirely. Branded search typically has low incrementality because most clicks come from users who would have found the brand anyway — they were already searching for it. Without the branded search ad, many would click the organic result.

However, branded search does provide some incremental value:

It defends against competitor conquesting (competitors bidding on your brand terms)
It captures users who would otherwise be distracted by competitor ads in the results page
In competitive categories, organic position is not guaranteed

The right action is usually to run a branded search holdout test (turn off brand bidding in a subset of markets for 4–6 weeks) and measure the actual conversion impact. Most brands find they can reduce branded search spend by 30–60% with minimal revenue impact; few find they can eliminate it entirely.
How do we handle attribution for long purchase cycles (B2B, auto, financial services)?
Long purchase cycles create two problems for attribution:

Path incompleteness: A user researching a car for three months will have touchpoints across many sessions, devices, and browsers. Path coverage degrades over longer windows.

Window mismatch: Standard attribution windows (7–30 days) miss the early-funnel touchpoints that initiated the consideration.

Solutions:

Extend attribution windows to match the category purchase cycle (90–180 days for auto and financial services, up to 12 months for B2B enterprise)
Use brand tracking surveys to measure upper-funnel impact of channels that cannot appear in conversion paths
Weight MMM outputs more heavily than attribution in long-cycle categories — MMM handles the full time horizon naturally
What path coverage rate is acceptable?
Coverage rates above 70% produce reliable attribution outputs. Between 50–70%, outputs are usable but should be interpreted with caution — note the coverage rate in all reporting. Below 50%, attribution is modeling an unrepresentative sample of conversions; increase MMM weight in the calibration mix and treat attribution as directional only.

Monitor coverage rate monthly. A declining trend (even if still above 50%) indicates a data infrastructure problem that should be addressed before it becomes critical.


Calibration questions
How long before the calibration loop produces reliable unified outputs?
Expect 6–9 months from first MMM output to a well-calibrated unified system. The bottleneck is the experiment program — each geo-lift test takes 4–8 weeks to run, and you need results from at least 3–4 channels before the calibration has meaningful experimental grounding.

During the first 3–6 months, MMM outputs are the most reliable layer. Attribution outputs calibrated only against MMM (no experimental data) are better than uncalibrated attribution but should be treated with more uncertainty than fully calibrated outputs.
What do we do when the MMM and a geo-lift test significantly disagree?
Divergence >40% between MMM iROAS and experimental iROAS triggers a red flag protocol (see specs/calibration-spec.md).

Do not immediately assume the test is right and the MMM is wrong — investigate both:

Check the test: Was the synthetic control fit acceptable (R² > 0.80)? Were there contaminating events? Was the test adequately powered? Did the treatment actually run as planned in the treatment markets?

Check the MMM: Is there collinearity between the tested channel and others? Is the tested channel's prior appropriate for the time period? Are there missing covariates around the test period?

If both check out, the experimental result takes precedence — it is closer to ground truth. Update the MMM prior and schedule a follow-up test within 6 months to confirm the calibrated estimate is stable.
Can we use UMM with only one or two channels?
Yes, though the value of unification is lower with fewer channels. A brand running only paid search and paid social can still benefit from MMM and incrementality testing — the calibration loop between those two is still valuable. Attribution calibration is less critical with only two channels.

UMM's full value emerges at 4+ channels, where the interaction effects, budget reallocation opportunities, and the risk of relying on any single measurement method are all larger.


Data and infrastructure questions
Can we use UMM with daily data instead of weekly?
Yes, and daily data is preferred for digital channels where spend and revenue move quickly. Daily data gives the MMM more data points, improves adstock estimation, and makes the model more responsive to spend changes.

The tradeoff: daily revenue data is noisier than weekly (day-of-week effects, one-off events, payment processing timing). The model needs additional covariates (day-of-week indicators) to handle this noise. If daily revenue is very noisy, weekly aggregation often produces more stable MMM estimates.

Use daily for digital-heavy brands with clean daily revenue data. Use weekly for brands with significant offline revenue or noisy daily reporting.
How do we handle currency and multi-market data?
For multi-market MMM:

Option 1 — Market-level models: Run separate MMM instances per market, each in local currency. Calibrate separately. Compare iROAS across markets using a common currency conversion. Best for markets with very different media landscapes or business models.

Option 2 — Pooled model: Run a single MMM with market indicators as covariates. Share adstock and saturation priors across markets (with market-specific offsets). Best for brands with consistent media mix and consumer behavior across markets.

For global brands with 5+ major markets, a hierarchical Bayesian model that pools information across markets while allowing market-specific parameters is the most powerful approach. This is the most complex to implement and requires significant data science resources.
What if we don't have granular spend data by channel — only total marketing spend?
A single-channel MMM tells you that marketing spend has an effect but cannot allocate it across channels. This is useful for demonstrating that marketing has incremental value but not for budget optimization.

If channel-level spend data exists in your finance system but is not yet aggregated cleanly, invest in that data pipeline before building the MMM. The channel-level splits are the primary output of the model — without them, the main use case is lost.

If channel-level spend genuinely does not exist (e.g., agency manages spend without providing channel breakdowns), negotiate for it in your next agency contract. It is standard data that agencies have and should share.
How do we handle major structural breaks in our data — acquisitions, rebrands, COVID?
Flag the break with an anomaly_flag covariate for the affected period. The model will fit a parameter for the flagged period, preventing the anomaly from distorting channel contribution estimates.

For very large breaks (COVID March–June 2020, for example), consider truncating the dataset to exclude the break period if the pre-break period is more than 2 years ago. The incremental value of 5-year-old data is limited, and forcing the model to span a major structural break can hurt fit quality.

When in doubt, run models with and without the break-spanning data and compare holdout accuracy. Use whichever specification produces better out-of-sample predictions.


Reporting and communication questions
How do we communicate iROAS to the CFO when it differs from platform-reported ROAS?
Frame the difference as the distinction between gross and net revenue measurement:

"Platform ROAS counts all revenue that occurred after someone saw an ad — including customers who would have bought anyway. iROAS measures only the revenue that happened because of the ad. The difference is what we were spending to re-acquire customers who didn't need acquiring."

Anchor with a specific number where possible: "Our platform reports 6x ROAS on retargeting. Our incrementality test shows 1.8x iROAS. The 4.2x gap represents $X in spend that was capturing existing intent, not driving new revenue."

CFOs respond to the framing of marketing spend as provable, evidence-based investment. The language of "causal" and "incremental" maps directly to how finance thinks about investment returns. Use it.
Should we share the full MMM uncertainty intervals with stakeholders?
Yes, with context. Suppressing uncertainty intervals implies false precision and erodes trust when estimates shift. Stakeholders who understand why estimates have uncertainty ranges make better decisions than those who expect point estimates to be exact.

Frame credible intervals as the range of outcomes consistent with the data: "Our best estimate is that TV drove $4.2M in incremental revenue last quarter, and we're 80% confident the true number is between $3.1M and $5.6M." This is more useful than "$4.2M ± some unspecified error."

For channels with very wide intervals, be direct: "We don't have a precise estimate for OOH yet — our model suggests $1–3M, and we've queued a geo-lift test to narrow this down."
How often should we report UMM outputs to the business?
Output
Cadence
Audience
Channel iROAS and budget curves
Monthly
CMO, finance, channel owners
Experiment results
As completed
CMO, relevant channel owner
Budget reallocation recommendations
Quarterly (or post-experiment)
CMO, CFO, finance
Attribution by campaign
Weekly
Channel managers
Full decomposition and portfolio iROAS
Quarterly
CMO, CFO, board


Avoid over-reporting — weekly MMM numbers create pressure to react to noise. Monthly is the right cadence for most strategic outputs. Weekly attribution is appropriate for campaign-level operational decisions.


