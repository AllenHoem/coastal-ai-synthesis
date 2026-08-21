# Tradeoffs Register

Every consequential choice, what it buys, what it costs, and what would reverse it.
Severity is **impact on the credibility of the Phase 2 result**, not on engineering effort.

Legend: 🔴 threatens a headline claim · 🟠 needs an explicit caveat in the report · 🟡 bounded, noted for completeness

---

## T1 · llama.cpp/Vulkan instead of vLLM 🟠

**Bought:** A working logprob-capable, slot-capable server on RDNA3 in ~20 minutes, on hardware where the alternative may never work at all.

**Cost:** llama.cpp's continuous batching is less efficient than vLLM's paged-attention scheduler at high concurrency. The c=32 and c=128 throughput numbers will **understate** what a well-served T3 appliance achieves.

**Direction of the error:** Conservative for the thesis. The concurrency argument (PRD §2, the ~13x row) gets *weaker*, not stronger, from this substitution. A finding that survives it survives a better server.

**Caveat text required in the report:** "Throughput measured under llama.cpp continuous batching. vLLM-class serving would raise these figures; treat them as a floor."

**Reverses if:** the operator gets access to a CDNA/Instinct card or an NVIDIA card, at which point vLLM becomes available and the concurrency ceiling should be re-measured.

---

## T2 · Energy modeled from spec sheets, not metered 🟡

*Downgraded from 🟠. See [ADR-0011](decisions/ADR-0011-energy-demoted.md).*

**Bought:** Zero setup, zero hardware purchase.

**Cost:** Tokens-per-joule is an estimate — and that no longer matters, because
tokens-per-joule is no longer a headline. Energy is **2.1% of the local arm's cost** and
**0.077% of the gap between arms**. A 5× error in the power model moves cost-per-record
from 28.3× to 26.1× cheaper than cloud; the PRD's success target is ≥5×. Nothing the
scorecard decides is sensitive to it.

**What changed:** the metric was demoted, not the measurement. Energy now appears as one
line in the cost breakdown — shown *with* its sensitivity band, because proving electricity
is a rounding error closes the "but you pay for the power" objection permanently — and as a
binary thermal/battery envelope check against the tier ladder.

**The smart-plug recommendation is withdrawn.** The `PowerSource` interface stays because
it is already specified and costs nothing; the MacBook's free `powermetrics` anchor stays
for the same reason. Neither is on a critical path.

**Reverses if:** a buyer scores energy directly — sustainability reporting, a data-center
power cap, or IT objecting to battery drain on employee laptops. The third is the most
likely, and it is an envelope check, not a curve.

---

## T3 · Accuracy at c=1 only, throughput swept separately 🟡

**Bought:** ~4x reduction in run time and cloud spend. Removes batching non-determinism as a confound in the accuracy numbers rather than having to explain it.

**Cost:** Strictly, accuracy at c=32 is inferred rather than measured across the full corpus.

**Mitigation:** Pass C replays a 50-document subset at c=32 with the accuracy configuration. Either it matches — and the inference is licensed — or it doesn't, and the divergence is itself a result worth publishing. There is no losing branch.

**Residual risk:** a 50-document subset has limited power to detect a small shift. State the detectable effect size in the report rather than claiming equivalence.

---

## T4 · Cloud concurrency capped by account tier 🟠

**Bought:** Nothing — this is imposed by using a personal pay-as-you-go account.

**Cost:** The cloud arm cannot be measured at c=32 or c=128. A naive harness would hit 429s, retry with backoff, and report the resulting inflation as cloud latency.

**Handling:** cloud concurrency above 8 is reported as *not measured — account tier limited*, never silently dropped. Bulk cloud accuracy runs through the Batch API (50% cost, no RPM pressure) with `via_batch_api: true`, and those records are excluded from every latency chart.

**Does it hurt the thesis?** No. The concurrency argument is about **local batch economics**. Cloud is the comparison baseline, and its per-token price does not change with concurrency.

**Reverses:** ~$40 of cumulative spend typically moves an account up a rate-limit tier.

---

## T5 · Windows-native serving, WSL orchestration 🟡

**Bought:** The only reliable RDNA3 path. Prebuilt binary, existing driver.

**Cost:** WSL has no GPU. OCR, rasterization, and analysis are CPU-only. One extra loopback hop per inference call.

**Why it's minor:** the hop is sub-millisecond and applies identically to every arm, so it cannot bias a comparison. CPU-side OCR is arguably *more* representative of a T0 deployment than GPU-accelerated OCR would be.

**One real gotcha:** keep the repo on the WSL ext4 filesystem (`~/…`), never on `/mnt/c`. Cross-filesystem I/O in WSL2 is roughly an order of magnitude slower and will dominate corpus generation time.

---

## T6 · 24GB card reported as 20GB (PRD D11) 🟡

**Bought:** Consistency with the PRD's conservative reporting assumption; forces T0/T1-relevant model sizes to the foreground, which is the actual research question.

**Cost:** 4GB of real headroom goes unused. Configurations that would fit in 24GB but not 20GB are untested.

**Handling:** enforced as an assertion, not a hope — the harness records peak resident VRAM and fails the run if it exceeds the configured cap. An assumption that isn't checked is a footnote; one that fails the build is a constraint.

---

## T7 · Synthetic corpus 🟠

**Bought:** Perfect ground truth at zero labeling cost, controlled difficulty, no data-access negotiation. This is PRD D2 and is settled.

**Cost:** "Synthetic data is too clean" is the first objection any skeptical reviewer raises, and it is not an unreasonable one.

**Handling:** the degradation ladder (L0–L4) exists precisely to answer it, and P1-5's 20–30 real-invoice holdout is the direct rebuttal. If Phase 3 gets cut, **the holdout is the one piece worth rescuing from it** — it is cheap and it neutralizes the loudest objection.

---

## T8 · Static HTML report instead of a live dashboard 🟡

**Bought:** Nothing to run, nothing to expose, nothing to maintain. Works across a DHCP LAN with no tunnel. Emailable, archivable, diffable, deterministic.

**Cost:** No interactive filtering. Exploring a hypothesis not anticipated by the report means writing SQL.

**Why it's the right call here:** the audience for this output is a skeptical technical reviewer and eventually a CFO. Both want a document they can keep, not a URL that may not resolve tomorrow. DuckDB covers the exploration case without a running process.

**Reverses cheaply:** a Streamlit app over the same DuckDB views is a later afternoon's work if exploration becomes the bottleneck. The views are the interface; the report is one consumer of them.

---

## T9 · Hand-rolled agent loop instead of a framework 🟡

**Bought:** The escalation ladder is the object of study. Every rung transition, retry, and gate decision is explicit and logged. No hidden retry inflates a rung count.

**Cost:** More code to write than importing LangGraph. No free tracing UI.

**Why it's not close:** a framework's built-in retry logic would silently contaminate rung accounting — the single most important measurement in the project. Instrumenting around a framework costs more than writing 300 lines of state machine.

---

## T10 · Logprobs under grammar-constrained decoding 🔴 (design-level)

**Not a choice — a property to avoid being caught by.**

Constrained decoding (GBNF / JSON schema) masks invalid tokens before the softmax. The resulting logprobs are **post-mask** and systematically over-confident: a token can show near-certainty simply because the grammar forbade every alternative.

**Consequence:** P1-2's calibration analysis and any pre-emptive router built on mean field logprob are **invalid** if they mix constrained and unconstrained decodes without distinguishing them.

**Handling:** `attempts[].grammar_constrained` is a required field. Calibration analysis segments on it. Where the reliability diagram is the point, run an unconstrained decode alongside and compare.

**Why it's flagged red:** this is easy to miss, produces a beautiful-looking reliability diagram, and is wrong. It would survive casual review and fail a sharp one.

---

## T11 · Anthropic as the primary cloud tier 🟡

**Bought:** One vendor, two clean tiers (Sonnet 5 mid, Opus 5 frontier), exact billed token counts from `usage`.

**Cost:** No cross-vendor sensitivity check on the escalation results.

**Note:** the Messages API does **not** expose per-token logprobs. This is fine — escalation is decided locally — but it means every calibration signal in P1-2 must come from the local tier or from OCR. A router design that assumes cloud confidence scores would have to be reworked.

**Handling:** the cloud adapter is an interface; OpenAI and Gemini are config swaps if a sensitivity check is wanted. Rates live in a dated `rates.toml`, never in code.

---

## T12 · Rate cards drift mid-experiment 🟠

**Specific and live:** Claude Sonnet 5's introductory pricing ($2/$10 per MTok) expires **2026-08-31**, reverting to $3/$15. That is inside this project's build window.

**Consequence if unhandled:** two runs a week apart produce different cost-per-record for identical behavior, and the difference is invisible.

**Handling:** `rates.toml` entries carry `effective_from` / `effective_until`; every trajectory record pins `cost.rate_card_id`; the report shows which card each figure used. Cost comparisons across runs assert matching card IDs and refuse to plot otherwise.

---

## Summary

| # | Tradeoff | Severity | Reversible? |
|---|---|---|---|
| T1 | llama.cpp over vLLM | 🟠 | With different hardware |
| T2 | Modeled energy | 🟡 | Metric demoted — ADR-0011 |
| T3 | Accuracy at c=1 | 🟡 | Pass C mitigates |
| T4 | Cloud concurrency capped | 🟠 | ~$40 spend |
| T5 | Windows/WSL split | 🟡 | No — and no need |
| T6 | 20GB cap on a 24GB card | 🟡 | Config line |
| T7 | Synthetic corpus | 🟠 | P1-5 holdout |
| T8 | Static report | 🟡 | Afternoon's work |
| T9 | No agent framework | 🟡 | Wouldn't want to |
| T10 | Constrained-decode logprobs | 🔴 | Handled by design |
| T11 | Anthropic-only cloud | 🟡 | Config swap |
| T12 | Rate card drift | 🟠 | Handled by design |

**The one that would actually damage a client conversation if mishandled is T10.** It is
handled structurally, by a required field the calibration analysis segments on.

T2 was the other candidate until the arithmetic retired it: energy is 2.1% of the local
arm's cost, so a metric nobody weighs was carrying an unpriced hardware dependency. Demoting
it removed the dependency and produced a better exhibit — a sensitivity table that closes
the power objection instead of inviting it.

The remaining 🟠 entries are all disclosure problems rather than measurement problems. Each
is handled by saying the thing out loud in the report: what the runtime understates, what
the account tier prevented, which rate card applied, how clean the corpus is.
