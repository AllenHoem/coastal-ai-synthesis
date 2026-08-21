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
| [`docs/decisions/`](docs/decisions/) | 10 ADRs covering the PRD §5 delegated decisions |
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

One command produces one self-contained HTML file — inline SVG, no CDN, no server, no port to expose. It opens from an email attachment or a USB stick.

Alongside it: `summary.csv`, `trajectories.parquet`, and `results.duckdb` for ad-hoc SQL against ~12 named views.

Three properties are non-negotiable:

1. **Silent-failure rate is the first thing on the page**, standalone, never averaged into aggregate success.
2. **Every number carries a provenance badge** — measured, modeled, or projected. The test rig models power while the Mac measures it; that asymmetry is labelled, never smoothed.
3. **Deterministic.** Same `run_id`, byte-identical report.

---

## Two things that are easy to get wrong

**Logprobs under grammar-constrained decoding are post-mask.** A token can look near-certain because the JSON schema forbade every alternative. Any calibration analysis mixing constrained and unconstrained decodes produces a beautiful reliability diagram that is wrong. Segment on `attempts[].grammar_constrained`.

**Rate cards expire.** Claude Sonnet 5's introductory pricing lapses 2026-08-31. Cost figures pin `cost.rate_card_id`, and cross-run comparisons assert matching ids before plotting.

---

## Status

Architecture complete; implementation not started. Phase 2 (`docs/architecture.md` §9) is the minimum publishable result — if effort has to be cut, it comes out of Phase 3.
