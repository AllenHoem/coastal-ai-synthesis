# ADR-0011 — Energy demoted from headline metric to a sensitivity footnote

**Status:** Accepted · **Supersedes in part:** ADR-0008 · **Date:** 2026-08-21

## Context

PRD P0-12 lists "tokens per joule, compared against the §2 baseline" as a required
output, and §2 presents a performance-per-watt table whose closing row — 862 tok/s
at concurrency 256, ~5.1 tokens per joule — is called "the most consequential number
in this document."

ADR-0008 accepted modeled power and set up mitigations to keep that comparison
defensible. On review, the operator's position is that tokens-per-joule is not a
forcing function for a buyer choosing between edge and cloud compute.

That position is correct, and the harness's own arithmetic settles it.

## The arithmetic

On a representative Arm A run (300 invoices, Qwen3-8B, cascade arm):

| Line | Amount | Share |
|---|---|---|
| Cloud escalation (20 × R2 field, 7 × R3 document) | $0.3965 | 97.9% |
| Local energy, modeled at 310W over 620 GPU-seconds | $0.0085 | **2.1%** |
| **Cascade total** | **$0.4050** | |
| Cloud-only comparison | $11.5500 | |

Electricity is **2.1% of the local arm's cost** and **0.077% of the difference
between the arms**. Sensitivity, holding everything else fixed:

| Power model error | Cost per successful record | Advantage vs cloud-only |
|---|---|---|
| Exact | $0.00137 | 28.3× |
| 2× wrong | $0.00140 | 27.7× |
| 3× wrong | $0.00143 | 27.1× |
| **5× wrong** | **$0.00149** | **26.1×** |

A five-fold error in the power model moves the headline from 28× to 26×. The PRD's
success target is ≥5× and its stretch is ≥15×. **No plausible energy error changes
any decision the scorecard makes.**

## Decision

1. **Tokens per joule is removed from the P0-12 headline set** and reported once, in
   a methodology appendix, labelled modeled.
2. **The §2 baseline comparison is dropped.** It compares a TDP-derived figure
   against published system watts — different quantities — and nothing depends on it.
3. **Energy appears in exactly two places going forward:**
   - one line in the cost breakdown, shown with the sensitivity band above, because
     demonstrating that electricity is a rounding error retires the "but you pay for
     the power" objection permanently and in one row;
   - a **binary envelope check**, not a curve: does sustained draw fit the tier's
     thermal and battery budget? This is the PRD §11 "operational tolerability"
     dimension, where power is a constraint rather than an economic metric.
4. **The metric that takes the vacated headline slot is throughput per device** —
   documents per hour per machine, at each concurrency. That is what sizes a fleet
   and therefore what drives capex, and it is the CFO-legible form of the same
   batching finding §2 was reaching for.
5. **The smart-plug recommendation is withdrawn.** ADR-0008's mitigation 3 (the
   `PowerSource` interface) stays because it is thirty lines and already written;
   mitigation 2 (the Mac's free `powermetrics` anchor) stays because it costs
   nothing. Neither is now on the critical path.

## What does not change

`cost.energy_source` remains a required trajectory field. It is one enum on a record
that is already being written, and provenance for a figure that still appears — even
a de-emphasized one — is cheaper to keep than to reconstruct.

Pass B still sweeps concurrency and still measures throughput and latency. **Only
the units of the headline changed**, from tokens per joule to documents per hour.
No measurement is removed from the harness.

## Consequences

- **Positive:** removes the only 🟠 tradeoff that had an unpriced hardware dependency.
  Tradeoff T2 drops to 🟡.
- **Positive:** replaces a metric no buyer weighs with one that maps directly to
  "how many machines do we need," which is a procurement question.
- **Positive:** the sensitivity table is more persuasive than a tokens-per-joule
  chart would have been. It closes the objection instead of inviting it.
- **Negative:** a reviewer holding the PRD may ask why §2's headline row is not
  reproduced. The appendix answers that with the sensitivity table.
- **Negative:** if a client ever raises sustainability or ESG reporting as a
  procurement criterion, this becomes a gap. Judged unlikely in mid-market, and the
  `PowerSource` seam plus a $20 plug closes it in an afternoon.

## Revisit if

A buyer raises energy as a scoring criterion in a real qualification conversation —
sustainability reporting, a data-center power cap, or a battery-life objection from
IT about running inference on employee laptops. The third is the most likely, and it
is an envelope check, not a tokens-per-joule curve.
