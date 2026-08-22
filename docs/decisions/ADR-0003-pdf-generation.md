# ADR-0003 — Corpus synthesis: Jinja2 → WeasyPrint → pypdfium2 → NumPy

**Status:** Accepted · **Delegated by:** PRD §5 · **Date:** 2026-08-21

## Context

P0-1 requires ≥300 documents, deterministically regenerable from a seed, across ≥8 visual templates, with 1–40 line items, multi-currency, multi-tax-rate, discounts and credit memos, machine-readable ground truth, and both digital-native and rasterized variants.

P0-2 requires a degradation ladder L0–L4 that is deterministic for a given seed and tagged with severity.

## Decision

Four stages, each independently seeded:

1. **Ground truth first.** A seeded generator produces the record; the document is rendered *from* it. Truth is never parsed back out of a rendered artifact.
2. **Jinja2 → WeasyPrint** for the PDF. Eight templates are eight CSS themes over one semantic HTML structure.
3. **pypdfium2** rasterizes at a configured DPI for L1+.
4. **Pillow + NumPy** apply the degradation ladder — every operation seeded from `(doc_seed, level)`.

L0 keeps the WeasyPrint output untouched, preserving a real text layer.

## Why not Playwright/Chromium

400MB of browser, and rendering shifts between Chromium versions — which would silently break "deterministically regenerable from a seed" across a driver update. WeasyPrint is pure Python and pinned by the lockfile.

## Why not ReportLab

Lower-level. Eight visually distinct templates are a CSS exercise in WeasyPrint and a layout-code exercise in ReportLab.

## Consequences

- **Positive:** the full corpus rebuilds from one integer. Corpus SHA folds into `run_id`.
- **Positive:** ground-truth-first eliminates a whole class of label error.
- **Negative:** WeasyPrint needs four system libraries (pango, harfbuzz, ffi) — the only `apt` step in the project.
- **Negative:** CSS-rendered invoices are visually cleaner than scanned real ones. This is PRD D2, accepted, and answered by the L0–L4 ladder plus the P1-5 real holdout. See tradeoff T7.

## Revisit if

Reviewers find the eight templates insufficiently diverse. Adding templates is cheap and does not invalidate prior runs, since template ID is recorded per document.
