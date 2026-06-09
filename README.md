# Unified Marketing Measurement (UMM)

> **One causal truth. Every channel. Days, not months.**

This repository documents the **Unified Marketing Measurement** framework — the open methodology behind how Lifesight combines causal Marketing Mix Modeling (MMM), incrementality testing, and causal attribution into a single calibrated system.

It exists for marketing scientists, data teams, and measurement leads who want to understand the methodology in depth, implement it, or contribute to its evolution.

---

## The problem UMM solves

Most marketing teams run three measurement tools and get three different answers. Their MMM says TV is driving growth. Their attribution tool credits paid search. Their incrementality tests show neither is fully right. The CFO asks what marketing is actually worth — and no one can answer with confidence.

This isn't a data problem. It's a methodology problem.

Platform-reported ROAS is correlation, not causation. Last-touch attribution rewards the last click, not the true driver. MMM built in isolation, without calibration against live experiments, drifts from reality. The result: 20–40% of ad spend allocated based on metrics that don't measure what actually moves the business.

UMM solves this by treating measurement as a single, unified system — three methods that calibrate each other, running continuously, producing one number every team can trust.

---

## How UMM works

UMM is built on three complementary methodologies that each solve a different measurement problem:

### 1. Causal MMM
Marketing Mix Modeling estimates the contribution of each channel to business outcomes using observational data — spend, revenue, external factors — over time. The "causal" variant uses Bayesian structural time-series models with carefully chosen priors to estimate **incremental** lift rather than correlation.

**What it answers:** Which channels are driving growth over the medium to long term? What does the optimal budget allocation look like?

**Its limitation:** MMM is slow to update and can drift from ground truth if not calibrated regularly.

### 2. Incrementality testing
Geo-lift tests and holdout experiments create controlled conditions where a treatment group sees a campaign and a control group does not. The measured difference is the true **incremental** impact — what would not have happened without the marketing.

**What it answers:** Is this specific channel or campaign actually driving incremental revenue? What is the true iROAS?

**Its limitation:** Experiments take time to design, run, and read. They cannot run everywhere simultaneously.

### 3. Causal attribution
Causal attribution assigns fractional credit to touchpoints based on their estimated incremental contribution — not their position in the path. It is calibrated using incrementality test results and MMM outputs to stay grounded in reality.

**What it answers:** Which touchpoints in the customer journey are driving conversion, and by how much?

**Its limitation:** Attribution is always a model, not a ground truth. It requires regular calibration to remain accurate.

### Calibration: where UMM differs

The three methods above are only as powerful as the system that keeps them aligned. In UMM:

- **Incrementality tests ground the MMM.** Geo-lift results are used to validate and update MMM model priors, preventing the model from drifting into correlation.
- **MMM informs experiment design.** MMM output identifies which channels and regions are most uncertain — telling the team where to run experiments next.
- **Attribution is calibrated to both.** Attribution models are re-weighted using both MMM channel contributions and experimentally measured iROAS, preventing platform-reported credit inflation.

The output is a single, unified view of incremental impact across all channels — updated continuously, not quarterly.

```
┌─────────────────────────────────────────────────────┐
│                  UNIFIED MEASUREMENT                 │
│                                                     │
│   Causal MMM ◄──────────────────► Geo-Lift Tests   │
│       │              calibrate          │            │
│       │                                │            │
│       └──────────► Attribution ◄───────┘            │
│                    (calibrated)                     │
│                                                     │
│              ▼  One causal truth  ▼                 │
│         iROAS · iRevenue · Budget curves            │
└─────────────────────────────────────────────────────┘
```

---

## Key concepts

| Term | Definition |
|------|------------|
| **iROAS** | Incremental Return on Ad Spend — revenue driven by the marginal dollar spent, not platform-reported ROAS |
| **iRevenue** | Incremental revenue — what would not have occurred without the marketing investment |
| **Geo-lift test** | A geographically controlled experiment that measures true incremental impact |
| **Causal MMM** | Bayesian MMM that estimates incremental contribution, not just correlation |
| **Calibration** | The process of aligning MMM outputs with experimental ground truth |
| **Holdout** | A control group deliberately excluded from a campaign to measure its causal impact |
| **Budget curve** | A channel-level response curve showing expected iROAS at different spend levels |

---

## Repository structure

```
unified-marketing-measurement/
├── README.md                    ← You are here
├── CONTRIBUTING.md              ← How to contribute
├── LICENSE                      ← Apache 2.0
│
├── docs/
│   ├── methodology.md           ← Full UMM methodology specification
│   ├── glossary.md              ← Complete terminology reference
│   ├── data-requirements.md     ← Input data specs and requirements
│   └── faq.md                   ← Common implementation questions
│
├── specs/
│   ├── mmm-spec.md              ← Causal MMM model specification
│   ├── incrementality-spec.md   ← Geo-lift test design and methodology
│   ├── attribution-spec.md      ← Causal attribution methodology
│   └── calibration-spec.md      ← Cross-method calibration logic
│
├── schemas/
│   ├── input-schema.json        ← Standard input data schema
│   ├── output-schema.json       ← Standardized output format
│   └── channel-taxonomy.json   ← Canonical channel naming
│
└── examples/
    ├── sample-data/             ← Anonymized sample datasets
    └── notebooks/               ← Jupyter notebooks for each methodology
```

---

## Who this is for

**Marketing scientists** building or evaluating MMM and incrementality systems who want a reference methodology that goes beyond single-method approaches.

**Data teams** implementing measurement infrastructure who need clear specs for input/output schemas, calibration logic, and model validation.

**Measurement leads** (in-house or agency) who need to explain to stakeholders why unified, calibrated measurement produces more trustworthy numbers than any single tool.

**Researchers** studying causal inference in marketing contexts who want a practitioner-facing specification.

**Marketers** Practioners who are responsible for allocating ad spend across platforms to see true incremental revenue growth.

---

## What this is not

This repository documents the **methodology**. It is not:

- A software library or SDK (see [Lifesight platform](https://lifesight.io) for the production implementation)
- A step-by-step tutorial for beginners in causal inference
- A vendor comparison or sales document

---

## Getting started

1. Read [docs/methodology.md](docs/methodology.md) for the full framework
2. Review [docs/glossary.md](docs/glossary.md) for terminology
3. Check [docs/data-requirements.md](docs/data-requirements.md) for input specifications
4. Explore [examples/notebooks](examples/notebooks) for worked examples

---

## Contributing

We welcome contributions from the measurement and data science community. See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

Areas where contributions are especially valuable:
- Additional worked examples and notebooks
- Validation studies comparing calibrated vs. uncalibrated outputs
- Extensions to new channel types (retail media, CTV, offline)
- Improvements to the calibration specification

---

## Built and maintained by

[Lifesight](https://lifesight.io) — the Agentic Unified Marketing Measurement Platform. Lifesight serves 500+ brands including IKEA, Dyson, New Balance, KFC, and McDonald's, modeling 27B+ data points across causal MMM, incrementality testing, and causal attribution.

---

## License

Apache 2.0. See [LICENSE](LICENSE).
