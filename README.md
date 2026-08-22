# Coastal AI Synthesis — Edge-First Agentic Inference Prototype

An instrument for answering one question with evidence:

> **What is the smallest model that clears the silent-failure threshold for a given workload — and therefore what hardware tier does that workload require?**

This is not a document-ingestion product. It is a measurement harness that produces a defensible answer to hardware sizing and cost projection for agentic workloads running on infrastructure a client already owns.

Implements `Edge-First Agentic Inference PRD v0.2`.

---

## Start here

| Document | What it covers |
|---|---|
| [`docs/architecture.md`](docs/architecture.md) | System design, diagrams, the four measurement problems, build sequence |
| [`docs/tradeoffs.md`](docs/tradeoffs.md) | 12 tradeoffs, severity-rated, each with its reversal condition |
| [`docs/runbook-bringup.md`](docs/runbook-bringup.md) | Zero to first token in ~4 hours |
| [`docs/report-composition.md`](docs/report-composition.md) | How one JSONL becomes two reports — operator and executive |
| [`docs/decisions/`](docs/decisions/) | 12 ADRs covering the PRD §5 delegated decisions and two revisions |
| [`schemas/trajectory.schema.json`](schemas/trajectory.schema.json) | The system of record. Frozen before anything writes to it. |

---

## Deployment shape

```
Windows 11 host          llama-server (llama.cpp, Vulkan, RX 7900 XTX)  :8080
     │                   ← ROCm-in-WSL does not support consumer RDNA3
     │  loopback
WSL2 Ubuntu              corpus · extraction · orchestrator · analysis
     │  LAN
MacBook Pro M4 24GB      llama-server (Metal) — T1 anchor, measured power
     │  HTTPS
Anthropic API            R2/R3 escalation only
```

Everything is LAN-local. Nothing is exposed. The deliverable is a file.

---

## Operator interface

```bash
make bringup            # verify both nodes; refuses to pass until every check is green
make probe-node NODE=   # bandwidth + power characterization
make corpus             # regenerate 300 documents from one seed
make smoke              # one document end to end — the Phase 1 exit gate
make run CFG=           # execute a sweep; resumable, spend-guarded
make report RUN=        # trajectory.jsonl → report.html + csv + parquet
make serve-report RUN=  # LAN-serve the report; prints the current DHCP IP
```

---

## Reviewing results

One command produces **two** self-contained HTML files from the same DuckDB views — inline SVG, no CDN, no server, no port to expose. Both open from an email attachment or a USB stick.

- `report-operator.html` — full detail, for running the experiment and defending it to a technical reviewer.
- `report-executive.html` — one verdict, four numbers, three exhibits, a methodology strip. Fits on two screens. Built for the decision maker on the other side of an SOW.

Because both read the same SQL, they cannot disagree.

Alongside it: `summary.csv`, `trajectories.parquet`, and `results.duckdb` for ad-hoc SQL against ~12 named views.

Three properties are non-negotiable:

1. **Silent-failure rate is the first thing on the page**, standalone, never averaged into aggregate success.
2. **Every number carries a provenance badge** — measured, modeled, or projected. The test rig models power while the Mac measures it; that asymmetry is labelled, never smoothed.
3. **Deterministic.** Same `run_id`, byte-identical report.

---

## Two things that are easy to get wrong

**Logprobs under grammar-constrained decoding are post-mask.** A token can look near-certain because the JSON schema forbade every alternative. Any calibration analysis mixing constrained and unconstrained decodes produces a beautiful reliability diagram that is wrong. Segment on `attempts[].grammar_constrained`.

**Rate cards expire.** Claude Sonnet 5's introductory pricing lapses 2026-08-31. Cost figures pin `cost.rate_card_id`, and cross-run comparisons assert matching ids before plotting.

**Energy is a rounding error, and proving it is worth more than measuring it.** Electricity is ~2% of the local arm's cost; a 5× error in the power model still leaves the result 26× cheaper than cloud. Tokens per joule is an appendix figure, not a headline — the headline is documents per hour per machine, because that is what sizes a fleet. See [ADR-0011](docs/decisions/ADR-0011-energy-demoted.md).

---

## Status

Architecture complete; implementation not started. Phase 2 (`docs/architecture.md` §9) is the minimum publishable result — if effort has to be cut, it comes out of Phase 3.
