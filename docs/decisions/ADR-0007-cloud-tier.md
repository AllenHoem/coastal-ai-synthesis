# ADR-0007 — Cloud tier: Anthropic Sonnet 5 (mid) and Opus 5 (frontier), dated rate cards, Batch API for bulk

**Status:** Accepted · **Delegated by:** PRD §5 · **Date:** 2026-08-21

## Context

PRD §5 requires picking one mid-tier and one frontier-tier cloud model, recording exact model strings and published rates in config and never hardcoding them in logic. P0-10 requires cloud spend computed at published rates. R2/R3 are the escalation rungs that use it.

The operator has pay-as-you-go API keys on a personal account.

## Decision

| Rung role | Model string | Published rate (in / out per MTok) |
|---|---|---|
| Mid-tier | `claude-sonnet-5` | $2 / $10 **introductory, through 2026-08-31**, then $3 / $15 |
| Frontier | `claude-opus-5` | $5 / $25 |

- Cost is computed from the response's actual `usage` counts, never from an estimate.
- Rates live in `config/rates.toml` with `effective_from` / `effective_until`. Every trajectory record pins `cost.rate_card_id`.
- Bulk cloud accuracy passes run through the **Message Batches API** at 50% cost. Those records carry `via_batch_api: true` and are excluded from every latency chart.
- The provider interface is an adapter; OpenAI and Gemini are config swaps for a sensitivity check.

## Two properties that shape the design

**No logprobs.** The Anthropic Messages API does not expose per-token logprobs. This is fine — escalation is decided locally — but it means every calibration signal available to P1-2 must come from the local tier or from OCR. A router design assuming cloud confidence scores would need rework.

**Rate-limit ceiling.** A new pay-as-you-go account sits in the lowest rate-limit tier. Offering 32 or 128 concurrent requests produces 429s; a retry-backoff loop would inflate wall-clock that then gets reported as *cloud latency*. It is not — it is an artifact of the account. Cloud concurrency above 8 is therefore reported as **not measured, account-tier limited**, never silently omitted. See tradeoff T4.

## Why Anthropic as primary

One vendor gives two clean tiers with a shared API surface and exact billed token counts, minimizing adapter work. Cross-vendor comparison is not a PRD goal (D6/§13.4 rule out leaderboards).

## Consequences

- **Positive:** exact billed costs, not estimates. Batch API halves the largest single line of spend.
- **Positive:** dated rate cards mean a mid-experiment price change is visible rather than silent — and one is scheduled inside the build window.
- **Negative:** no cross-vendor sensitivity check unless explicitly added.
- **Negative:** expected total spend $60–150 across all passes including debugging. The spend guard (ADR-0005) bounds the failure mode.

## Revisit if

- The Sonnet 5 introductory rate lapses mid-experiment — the rate card handles it, but any cross-run cost comparison must assert matching `rate_card_id` and refuse to plot otherwise.
- A reviewer challenges vendor-specific results, in which case run one arm against OpenAI or Gemini as a sensitivity check.
