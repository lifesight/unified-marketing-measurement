# Data Requirements

This document specifies what data is needed to run UMM, how to source each data type, minimum quality standards, and how to handle common data problems.

For the formal data schema, see [schemas/input-schema.json](../schemas/input-schema.json).

---

## Summary

| Data type | Required | Minimum history | Primary source |
|---|---|---|---|
| Revenue / outcome | Yes | 78 weeks (18 months) | Finance / data warehouse |
| Spend by channel | Yes | Matching revenue history | Finance / media buying platform |
| Holiday indicators | Yes | Matching revenue history | Calendar (derived) |
| Promotion flags | Yes | Matching revenue history | Marketing / trade calendar |
| Price index | If pricing changed | Matching revenue history | Finance / commercial |
| Distribution index | If distribution changed | Matching revenue history | Commercial / supply chain |
| Competitor activity | Recommended | Matching revenue history | Third-party data / press |
| Macro / category index | Recommended | Matching revenue history | Public data / third-party |
| Anomaly flags | As needed | Matching revenue history | Business records |

---

## Revenue / outcome data

### What we need

Weekly total revenue (or primary conversion KPI), reconciled to finance records, for the full modeling window. Weekly is the minimum granularity; daily is preferred and enables more precise adstock estimation.

### Source

Use your authoritative revenue source — typically your ERP or data warehouse. Do not use:
- Platform-reported revenue (attributed, not actual)
- Revenue figures from multiple disconnected sources without reconciliation
- Gross revenue if net revenue is what the business manages to (pick one and be consistent)

### Adjustments

If the revenue series contains revenue that is clearly not marketing-driven — large B2B contracts closed by direct sales, one-time licensing deals, extraordinary items — remove those amounts and flag with `adjusted: true` and a note in metadata. Including non-marketing revenue inflates baseline estimates and compresses channel contribution estimates.

### Quality checks

Before submitting:
- [ ] Revenue reconciles to finance records within 1%
- [ ] No unexplained gaps (missing weeks) — impute or flag
- [ ] No unexplained step changes of >30% without corresponding covariate
- [ ] Series is in consistent currency throughout (no mid-series currency changes)
- [ ] Seasonality is visible and plausible (year-over-year patterns make sense)

---

## Spend data

### What we need

Weekly spend by channel, net of agency fees, in the same currency as revenue. Each channel that represents >2% of total budget should be reported separately. Channels below 2% can be aggregated into `other`.

### Source hierarchy

Use this priority order:

1. **Finance-reconciled actual spend** (invoices, media buys as billed): most reliable
2. **Media buying platform exports** (Google Ads, Meta Ads Manager, DV360, etc.): reliable for digital; may differ from invoiced amounts by a few percent
3. **Agency-provided consolidated reports**: acceptable if reconciled against invoices
4. **Estimates or modeled spend**: acceptable only as a last resort; flag clearly

### Channel definitions

| Channel | What to include | What to exclude |
|---|---|---|
| `paid_search_brand` | Spend on keywords containing the brand name | Competitor keyword targeting |
| `paid_search_nonbrand` | Category, competitor, and generic keyword spend | Brand keyword targeting |
| `paid_social` | Spend on Meta, TikTok, Pinterest, LinkedIn, Snapchat, X | Organic social (zero spend) |
| `display` | GDN, programmatic display, retargeting on display | Video placements (use online_video) |
| `online_video` | YouTube, pre-roll, mid-roll, outstream video | CTV/OTT placements (use tv_ctv) |
| `tv_linear` | National + local broadcast and cable TV | Streaming/CTV |
| `tv_ctv` | Connected TV, OTT (Hulu, Peacock, etc.) | Linear TV |
| `ooh` | Billboards, transit, street furniture, DOOH | Print |
| `email` | ESP platform costs + list rental (not staff time) | Organic/newsletter sends with no list cost |
| `affiliate` | Commission payments + platform fees | In-house refer-a-friend programs (usually immaterial) |
| `retail_media` | Amazon Ads, Walmart Connect, Criteo, other RMNs | Trade promotion (use other or separate) |

### Gross vs. net spend

Use **gross spend** (media cost before agency commissions) or **net spend** (after commissions) consistently throughout. The model produces iROAS relative to whatever spend metric is provided — gross and net will give different iROAS numbers. Most brands use gross. Document the choice in metadata.notes.

### Quality checks

- [ ] Spend sums across channels match total marketing budget in finance records (within 5%)
- [ ] No implausible spikes (>5x the weekly average) without a known explanation
- [ ] Zero spend weeks are reported as 0, not left blank
- [ ] Split between brand and non-brand paid search is available (required; do not aggregate)
- [ ] If a channel was not active in a period, it is reported as 0

---

## Holiday and event data

### Holiday indicators

Required for all markets. Flag the week containing the holiday (not just the holiday date). For holidays with multi-week effects (Christmas, major shopping events), flag 1–2 lead-up weeks separately.

Standard holidays to flag (adjust for market):

| Holiday | Lead weeks to flag | Post weeks to flag |
|---|---|---|
| Christmas / New Year | 2–3 | 1 |
| Black Friday / Cyber Monday | 1 | 0 |
| Prime Day (if relevant) | 0 | 0 |
| Valentine's Day | 1 | 0 |
| Mother's Day | 1 | 0 |
| Easter | 1 | 0 |
| Back to school | 2 | 0 |

### Promotion flags

Flag any week where a material price promotion, bundle offer, or trade-down event occurred. "Material" means a promotion that would have driven measurable incremental volume independent of media — typically discounts of >10% or high-visibility promotional events.

Source: marketing calendar, trade planning records, CRM.

### Product launches and distribution events

Flag the week of any major product launch (new SKU with expected demand impact), new retail partner launch, or significant distribution gain or loss. These events shift baseline demand and will be misattributed to media if not controlled for.

---

## External factors

### When to include

External factors are required when:
- The category is visibly affected by macroeconomic conditions (consumer durables, luxury, financial services)
- Competitor activity caused measurable demand shifts during the data window
- A macroeconomic shock (recession, pandemic, supply disruption) occurred during the data window

They are optional (but recommended) in all other cases.

### Macroeconomic indicators

| Indicator | Source | Notes |
|---|---|---|
| Consumer confidence index | Conference Board (US), GfK (EU/UK), national equivalents | Monthly; interpolate to weekly |
| Category spend index | Credit/debit card panel data (Earnest Analytics, Bloomberg Second Measure) | Most relevant for categories with good card coverage |
| Unemployment rate | BLS (US), Eurostat (EU), ONS (UK) | Monthly; interpolate to weekly |
| Fuel prices | EIA, OPEC | Relevant for categories with strong fuel price sensitivity |

### Competitor indicators

Ideal: competitor media spend (from Nielsen Ad Intel, Kantar, or equivalent). Where not available:
- Competitor product launch dates (press, news monitoring)
- Price change dates (pricing intelligence tools)
- Competitor out-of-stock or recall events (relevant if they shifted category demand)

Binary flags are acceptable when continuous data is unavailable.

---

## Anomaly handling

### What counts as an anomaly

- COVID-19 lockdown periods and demand shocks (March 2020 – March 2022, by market)
- Supply chain disruptions (out-of-stock periods)
- Data pipeline failures (missing or corrupted revenue or spend data)
- One-time extraordinary events (factory fire, product recall, major PR crisis)

### How to flag

Set `anomaly_flag: 1` for the affected weeks. The model will fit a separate parameter for flagged periods, preventing their effects from contaminating channel contribution estimates.

Document each anomaly in `metadata.notes` with:
- Date range affected
- Description of the anomaly
- Estimated revenue impact if known

### Imputation

Do not impute missing weeks with zeros. Use linear interpolation for gaps of 1–2 weeks. For gaps longer than 2 weeks, flag with `anomaly_flag: 1` and use the observed value if partial data is available, or a business-justified estimate if not. Document all imputations in metadata.notes.

---

## Minimum history requirements

| Scenario | Minimum history | Recommended |
|---|---|---|
| Standard brand | 78 weeks (18 months) | 104–208 weeks (2–4 years) |
| Highly seasonal business | 104 weeks (2 years) | 156 weeks (3 years) |
| Frequent model updates | 78 weeks (rolling) | 104+ weeks |
| New brand with limited history | 52 weeks | Test with holdout validation |

Fewer than 52 weeks of data is insufficient for MMM regardless of circumstances. The model cannot separate seasonality from channel contribution with less than one full annual cycle.

---

## Data delivery format

Submit data as:
- **JSON**: conforming to [schemas/input-schema.json](../schemas/input-schema.json) (preferred)
- **CSV**: one row per week, columns matching the schema field names (acceptable)

Do not submit:
- Excel files with formulas or multiple sheets (convert to CSV first)
- Data with mixed currencies in a single file
- Unreconciled data from multiple sources without consolidation

---

## Common data problems and fixes

| Problem | Symptom | Fix |
|---|---|---|
| Revenue from platform attribution (not finance) | Revenue is 2–4x higher than actual; very high channel ROI estimates | Replace with finance-reconciled revenue |
| Missing brand/non-brand split in paid search | Model conflates branded and non-branded efficiency | Request split from paid search platform export |
| Spend reported as net (agency commission removed) | iROAS is higher than benchmarks suggest | Clarify with finance; use gross throughout or document net |
| Holiday effects not flagged | Large residuals around holiday periods; channel contributions absorb holiday demand | Add holiday indicators; re-run |
| Anomalous periods not flagged | Model parameters distorted by COVID or other shocks | Flag anomalous weeks; re-run |
| Channel collinearity (channels always move together) | Very wide credible intervals for correlated channels | Use stronger priors; consider aggregating correlated channels |
| Revenue series has structural break (acquisition, major rebranding) | Model fits pre-break data poorly | Add a step-change covariate at the break date |
