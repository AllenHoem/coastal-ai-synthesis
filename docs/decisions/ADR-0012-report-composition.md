# ADR-0012 — Two renderings from one set of views: operator and executive

**Status:** Accepted · **Date:** 2026-08-21

## Context

The report has two audiences with genuinely different questions, and ADR-0004
specified only one rendering.

**The operator** (running the experiment) asks: did the run complete, is the
projection model holding, where is the capability curve crossing, which checks are
firing, what is the config hash.

**An executive at a company Coastal is pursuing an SOW with** asks: can this run on
hardware we already own, what does it cost per document versus today, how often will
it be wrong in a way nobody catches, what do we have to buy, and why should I believe
your numbers.

A technical report with an executive summary bolted on top serves neither. The
executive scrolls past a capability-versus-model-size curve because model size is not
a variable they control; the operator has no use for a fleet-sizing sentence.

## Decision

`make report RUN=<id>` emits **two HTML files from the same DuckDB views**:

| File | Audience | Shape |
|---|---|---|
| `report-operator.html` | Allen, and a skeptical technical reviewer | Full detail, every arm × size × concurrency, diagnostics, config hash, raw check firings |
| `report-executive.html` | Client-side decision makers | One verdict, four numbers, three exhibits, one methodology strip. Fits on two screens. |

The views are the interface. Neither rendering computes anything — both read the same
SQL, so the two documents cannot disagree.

## Composition rules for the executive rendering

1. **Verdict first, in a sentence a person can repeat from memory.** "This workload
   qualifies at tier T1 — the laptops on your existing refresh cycle." Not a chart.
2. **Translate every metric into a business unit.** Silent-failure rate becomes
   *"14 invoices per 1,000 posted with wrong values and nothing flagged them."* A
   controller feels that number; a percentage with three decimal places is discarded.
3. **Throughput becomes fleet sizing.** Not "312 docs/hour" but *"your 40,000
   invoices a month need one machine running about six hours a night."*
4. **Show the cost composition, not just the total.** The deterministic tier
   absorbing 58% of the work at zero marginal cost is the most surprising finding in
   the study and the one that most changes a buyer's mental model.
5. **Never show model size on the x-axis.** It is the operator's variable. The
   executive gets its *conclusion* — a tier — as a labelled verdict.
6. **The methodology strip is mandatory and is the last thing on the page.** Sample
   size, seed, corpus composition, what was measured versus modeled versus projected,
   and the kill criteria that were not triggered. This is what survives the client's
   own technical reviewer, and omitting it to look cleaner would be a mistake.
7. **Every figure carries its provenance badge.** Same three-state system as the
   operator rendering. An executive deck that quietly mixes measured and projected
   numbers is the failure mode this entire methodology exists to avoid.

## Why not a slide deck generator

Tempting, and wrong for now. An HTML page is one artifact that renders identically
everywhere, and its figures can be screenshotted into whatever template the
engagement calls for. Generating PPTX adds a dependency and a layout engine to
maintain in service of a format the consultant will restyle anyway. Revisit after the
first real client conversation, when the actual reuse pattern is known rather than
guessed.

## Consequences

- **Positive:** the executive rendering is a client-facing artifact from Phase 2
  onward, not something assembled by hand afterwards.
- **Positive:** shared views mean the two documents can never disagree — a real risk
  when an exec summary is maintained separately.
- **Positive:** forces the translation layer (percentage → per-1,000-documents,
  throughput → machine-hours) to be written once, in code, and reviewed.
- **Negative:** roughly a day of additional work in Phase 2 for the second template
  and the translation helpers.
- **Negative:** two templates to keep in sync when a view changes. Mitigated by both
  reading the same named views and failing loudly on a missing column.

## Revisit if

The first real client conversation shows the executive rendering is being used
differently than assumed — for example pasted wholesale into a proposal, which would
argue for a print stylesheet, or read on a phone, which would argue for a different
layout entirely.
