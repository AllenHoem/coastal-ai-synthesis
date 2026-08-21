# ADR-0009 — Measurement protocol: accuracy at c=1, throughput swept, invariance replay

**Status:** Accepted (operator decision) · **Date:** 2026-08-21

## Context

P0-9 requires every configuration measured at concurrency {1, 8, 32, 128}. P0-8 requires every arm across every model size. Naively that is a full cross-product.

Two complications:

- llama.cpp batching changes floating-point reduction order, so the same prompt can decode differently at c=1 and c=32. Accuracy measured across the sweep carries a non-determinism confound.
- Reaching high concurrency requires KV-cache quantization (`--cache-type-k/v q8_0`) and reduced per-slot context, which is itself an accuracy confound.

## Decision

Three passes rather than one cross-product.

**Pass A — accuracy.** Concurrency 1, f16 KV, temperature 0, fixed seed. All arms × all model sizes × all degradation levels. Produces the confusion matrix, the standalone silent-failure rate, and the minimum viable model size.

**Pass B — throughput and energy.** Concurrency {1, 8, 32, 128}, q8 KV where required. Fixed 100-document subset. Outputs are scored but never headline. Produces aggregate throughput, p50/p95 latency, TTFT, and tokens per joule.

**Pass C — invariance check.** A 50-document subset replayed at c=32 under Pass A's configuration. Either outcome rates match Pass A within noise — licensing the inference that batching is accuracy-neutral — or they diverge, which is itself a finding worth publishing.

## Why this over the full matrix

Roughly 4x less run time and cloud spend, and it *isolates* the non-determinism rather than leaving it as an unexplained confound spread across every cell.

## Consequences

- **Positive:** Pass C has no losing branch. Both outcomes are reportable.
- **Positive:** the accuracy numbers are clean — one sampling configuration, one KV dtype.
- **Negative:** accuracy at high concurrency is inferred rather than measured across the full corpus.
- **Negative:** a 50-document subset has limited power to detect a small shift. The report must state the **detectable effect size** rather than claiming equivalence.

## Required trajectory fields

`config.offered_concurrency`, `config.server_slots`, `config.kv_cache_dtype`, `attempts[].sampler`, `attempts[].ttft_ms`. Without these, records from the three passes are not separable after the fact.

## Revisit if

Pass C shows divergence. That converts the accuracy/throughput split from a shortcut into a research question, and Pass A would need re-running at the concurrency where the deployment actually operates.
