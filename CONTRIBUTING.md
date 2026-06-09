# Contributing to Unified Marketing Measurement

Thank you for your interest in contributing. UMM is an open methodology — it gets better when measurement scientists, data practitioners, and marketing researchers bring their experience to it.

This document explains how to contribute, what kinds of contributions are most valuable, and the standards we hold contributions to.

---

## What we're looking for

### High-value contributions

- **Validation studies**: Empirical comparisons of calibrated vs. uncalibrated outputs using real or synthetic data. If you've run a geo-lift test and can show how it updated an MMM, that's exactly what this repo needs.
- **New channel specifications**: The current specs cover paid search, paid social, TV, OOH, and display. Extensions to retail media, CTV, podcast, influencer, and offline channels are in demand.
- **Worked examples and notebooks**: Clear, reproducible Jupyter notebooks walking through each methodology with sample data. The more concrete the better.
- **Calibration improvements**: Better methods for integrating experimental results into MMM priors, detecting calibration drift, or handling sparse experimental data.
- **Corrections**: If something in the methodology is wrong, imprecise, or misleading — open an issue. We'd rather be corrected than authoritative and wrong.

### Lower-priority contributions

- Cosmetic changes (formatting, punctuation) without substantive content improvement
- Additions that duplicate existing content without improving it
- Advocacy for specific tools or vendors

---

## How to contribute

### 1. Open an issue first

Before writing anything, open an issue describing what you want to add or change and why. This avoids duplicated effort and lets us give early feedback on direction.

Use the issue templates in `.github/ISSUE_TEMPLATE/` — there are templates for:
- Methodology corrections
- New content proposals
- Worked example proposals
- Bug reports (errors in specs or schemas)

### 2. Fork and branch

```bash
git clone https://github.com/lifesight/unified-marketing-measurement.git
cd unified-marketing-measurement
git checkout -b your-branch-name
```

Branch naming convention:
- `docs/` — documentation additions or updates
- `spec/` — changes to methodology specifications
- `example/` — new notebooks or sample data
- `fix/` — corrections to existing content

### 3. Make your changes

Follow the style guidelines below. Keep changes focused — one topic per pull request.

### 4. Open a pull request

Use the pull request template. Include:
- What you changed and why
- Any relevant references (papers, experiments, platform documentation)
- Whether you'd like a detailed review or a quick pass

---

## Style guidelines

### Writing

- **Be precise, not exhaustive.** This is a methodology specification, not a textbook. Assume the reader has a statistics background. Explain choices; don't over-explain fundamentals.
- **Define terms on first use.** Link to `docs/glossary.md` for any term in the glossary.
- **Use concrete examples.** Abstract methodology is harder to evaluate and harder to implement. Worked examples with numbers are strongly preferred.
- **Cite sources.** If a methodological choice is backed by published research, cite it. Format: `[Author et al., Year](URL)`.
- **Avoid vendor language.** This repository documents an open methodology. References to specific platforms, tools, or vendors should be generic (e.g., "Bayesian sampling frameworks such as Stan or PyMC" not "you must use Stan").

### Markdown

- Use `#` for document title (one per file), `##` for major sections, `###` for subsections
- Code blocks with language tags: ` ```python `, ` ```json `, ` ```sql `
- Tables for structured comparisons — keep them readable in raw markdown
- Relative links between files: `[methodology](../docs/methodology.md)` not absolute URLs

### Notebooks

- Use `examples/sample-data/` for any data files referenced in notebooks
- All notebooks must run from top to bottom without errors on the provided sample data
- Include a markdown cell at the top with: what the notebook demonstrates, prerequisites, expected outputs
- Clear all outputs before committing — outputs are regenerated, not stored

### Schemas

- JSON Schema draft-07
- All fields must have `description` properties
- Required fields must be explicit
- Enums must list all valid values with inline comments explaining each

---

## Review process

Pull requests are reviewed by Lifesight's measurement science team. We aim to respond within 5 business days.

Review criteria:
- **Correctness**: Is the methodology sound? Are claims supported by evidence?
- **Consistency**: Does it align with the rest of the framework?
- **Clarity**: Can a competent practitioner implement this from the specification?
- **Completeness**: Does it cover edge cases and limitations honestly?

We may request changes, ask for clarification, or suggest splitting large PRs into smaller ones. We will not merge contributions that introduce vendor bias, remove uncertainty acknowledgments, or overstate the precision of the methodology.

---

## Code of conduct

Be direct. Be honest about limitations. Engage with criticism on its merits. Measurement science is full of hard tradeoffs and genuine uncertainty — contributions that pretend otherwise will not be merged.

We do not tolerate harassment, dismissiveness toward newcomers, or bad-faith engagement.

---

## Questions

Open an issue with the `question` label, or reach out to the Lifesight measurement science team at [measurement@lifesight.io](mailto:measurement@lifesight.io).
