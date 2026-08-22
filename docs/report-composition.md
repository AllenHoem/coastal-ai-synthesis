# Report Composition Spec

How `trajectory.jsonl` becomes two documents. Decided before building, per [ADR-0012](decisions/ADR-0012-report-composition.md).

---

## The shape

```
trajectory.jsonl  →  DuckDB · ~12 named views  →  ┬→  report-operator.html
                                                  └→  report-executive.html
```

Neither rendering computes anything. Both read the same SQL, so they cannot disagree —
which is the failure mode when an executive summary is maintained by hand alongside a
technical report.

---

## Rendering A — `report-operator.html`

Audience: Allen running the experiment, and a skeptical technical reviewer afterwards.

| Block | Content |
|---|---|
| Run header | `run_id`, seed, corpus SHA, lockfile hash, llama.cpp build, node configs, wall clock |
| Silent failure | Standalone, by model size, with the 2% threshold marked |
| Confusion matrix | Full 2×2 per arm × size |
| Capability curve | Task success and silent failure vs model size, small multiples |
| Cost detail | Per arm × size × concurrency, broken out by rung |
| Throughput | Aggregate t/s, p50/p95, TTFT, docs/hour, per node, per concurrency |
| Projection | Fitted utilization, both anchors, residuals, ±15% validation verdict |
| Gate diagnostics | Which checks fired, how often, false-positive rate per check |
| Pass C | Invariance replay: outcome rates at c=1 vs c=32, detectable effect size |
| Appendix | Tokens per joule (modeled), envelope check, degradation detail |

---

## Rendering B — `report-executive.html`

Audience: the decision maker at a company Coastal is pursuing an SOW with.
Constraint: **fits on two screens.** Everything below the fold is optional reading.

### Seven composition rules

1. **Verdict first, in a sentence someone can repeat from memory.**
   *"This workload qualifies at tier T1 — the laptops already on your refresh cycle."*
   Not a chart. A chart is an argument; the verdict is the conclusion.

2. **Translate every metric into a business unit.** A rate with three decimals gets
   discarded; a count of documents does not.

   | Internal | Executive |
   |---|---|
   | silent-failure rate 1.4% | **14 invoices per 1,000 posted with wrong values and nothing flagged them** |
   | task success 98.4% | 984 of 1,000 straight through |
   | $0.00137/record | **$1.37 per thousand invoices, versus $38.81 today** |
   | 312 docs/hour | **your 40,000/month needs one machine, ~6 hours a night** |
   | 58% deterministic absorption | **58 of every 100 invoices never touch an AI model at all** |

3. **Never put model size on the x-axis.** It is the operator's variable, not theirs.
   They get its conclusion — a tier — as a labelled verdict.

4. **Show cost composition, not just the total.** The deterministic tier absorbing 58%
   at zero marginal cost is the most surprising finding in the study and the one that
   most changes a buyer's mental model of what they are buying.

5. **Lead the risk exhibit with the bad number.** Silent failure gets its own block,
   stated as a count, with the escalation that catches the rest shown beside it. Burying
   it reads as concealment to exactly the reader whose trust is the deliverable.

6. **Every figure carries a provenance badge** — measured, modeled, projected. An
   executive exhibit that quietly mixes measured and projected numbers is the failure
   this whole methodology exists to prevent.

7. **The methodology strip is mandatory and goes last.** Sample size, seed, corpus
   composition, what was measured versus projected, and the kill criteria that were *not*
   triggered. This is what survives the client's own technical reviewer. Removing it to
   look cleaner would trade the only thing that makes the document worth more than a
   vendor deck.

### Block order

```
1  VERDICT ......... tier, one sentence, qualified / not qualified
2  FOUR NUMBERS .... cost per 1,000 · silent failures per 1,000 · % never touching a model · machine-hours
3  EXHIBIT 1 ....... where the money goes  (stacked, cascade vs cloud-only)
4  EXHIBIT 2 ....... risk, per 1,000 documents  (segmented, status-coloured, labelled)
5  EXHIBIT 3 ....... fleet sizing  (docs/hour by tier, projected tiers marked)
6  WHAT YOU BUY .... hardware, one table, with the do-nothing option priced
7  DEGRADATION ..... escalation vs scan quality — the "our scans are worse" answer
8  METHODOLOGY ..... n, seed, measured/modeled/projected, kill criteria not triggered
```

Blocks 1–4 are the two-screen budget. 5–8 are for the reader who keeps going, and for
the technical reviewer they forward it to.

---

## Translation layer

The mappings in rule 2 live in `analysis/translate.py`, are unit-tested against the
views, and are written **once**. Rounding is deliberate: executive figures round to the
precision the decision needs (`$1.37`, `14 per 1,000`), while the operator rendering
keeps full precision. Both cite the same underlying view column, so a discrepancy is a
bug rather than an editorial choice.

## Anti-goals

- No PPTX generation. Figures get screenshotted into whatever template the engagement
  uses; a layout engine is a dependency in service of a format that gets restyled anyway.
- No interactive filtering in the executive rendering. A filter is an invitation to
  find a different answer, and the verdict is the product.
- No client logo, no co-branding. This is evidence, not a proposal.
