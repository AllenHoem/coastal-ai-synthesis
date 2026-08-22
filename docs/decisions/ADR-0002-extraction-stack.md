# ADR-0002 — Extraction stack: pypdfium2 text layer, RapidOCR fallback

**Status:** Accepted · **Delegated by:** PRD §5 · **Date:** 2026-08-21

## Context

P0-3 requires a deterministic extraction tier that runs fully local with zero network egress, attempts text-layer extraction before OCR, and emits character- and field-level confidence. P0-1 requires handling both digital-native and rasterized PDFs.

D3 makes this tier's absorption rate one of the headline findings: the largest expected cost result is work that never needed an LLM at all.

## Decision

**`pypdfium2` for text-layer detection and extraction; `RapidOCR` (ONNXRuntime, PP-OCR weights) on fallback.**

Field confidence is aggregated from character/line confidences over the span matched to each field, with the aggregation function recorded in the trajectory so it can be audited rather than trusted.

## Why not Tesseract

One `apt-get` and thirty years of tuning make it tempting, and it emits word-level confidence. But it degrades sharply on the skew, compression artifacts, and occlusion of L3/L4 — exactly the range where the degradation curve (P1-1) has to be credible. A weak OCR tier would show up as inflated escalation rates and be misread as a model finding.

## Why not a cloud OCR service

P0-3 requires zero network egress, asserted in test. Non-negotiable.

## Consequences

- **Positive:** pip-installable with no system packages. Runs CPU-only, which is what WSL has (ADR-0001) and what T0 would have.
- **Positive:** text-layer-first means digital-native invoices cost essentially nothing, which is the point of P0-1's insistence on including them.
- **Negative:** ONNX model weights add ~100MB to the install.
- **Negative:** field confidence is a derived aggregate, not a native output. The aggregation is a modelling choice and must be documented as one.

## Revisit if

L3/L4 extraction accuracy is poor enough to dominate the degradation curve. In that case run Tesseract alongside as a comparison baseline before concluding anything about model capability — an OCR floor and a model ceiling look identical in the aggregate numbers.
