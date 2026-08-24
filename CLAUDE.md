# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A measurement harness (not a product) answering one question with evidence: **what is the smallest model that clears the silent-failure threshold for a given workload, and therefore what hardware tier does that workload require?** It implements `Edge-First Agentic Inference PRD v0.2` for Coastal AI Synthesis.

**Current status: architecture complete, implementation not started.** The repo contains only documentation, config templates, and the frozen trajectory schema. There is no `Makefile`, `pyproject.toml`, or `src/` yet — the Make targets in the README are the *planned* operator interface. When implementation begins, per PRD §10 the trajectory schema (`schemas/trajectory.schema.json`) is built and frozen **before** any component that writes to it, and Phase order in `docs/architecture.md` §9 is binding: Phase 2 is the minimum publishable result and is protected; cuts come out of Phase 3.

## Commands (planned operator interface)

All operator interaction goes through Make. From README/architecture §10:

```bash
make bringup            # verify both nodes; refuses to pass until every check is green
make probe-node NODE=   # bandwidth + power characterization of one node
make corpus             # regenerate 300 documents from one seed
make smoke              # one document end to end — the Phase 1 exit gate
make run CFG=           # execute a sweep; resumable, spend-guarded
make report RUN=        # trajectory.jsonl → both HTML reports + csv + parquet
make serve-report RUN=  # LAN-serve the report; prints the current DHCP IP
```

Packaging is **uv with a src layout** (`src/edgefirst/`), config is **TOML + Pydantic**, and there is deliberately **no CI/CD, no containerization** (architecture §12). Bring-up steps (drivers, llama-server flags, WSL mirrored networking, MacBook node) are in `docs/runbook-bringup.md`; keep the repo on ext4 inside WSL, not `/mnt/c`.

## Architecture — the big picture

Deployment spans three machines plus a cloud tier; everything is LAN-local and the deliverable is a file, not a hosted service:

- **Windows 11 host** runs `llama-server` (llama.cpp, Vulkan) on the RX 7900 XTX — ROCm does not support consumer RDNA3 under WSL, so serving is Windows-native and reached over loopback HTTP (ADR-0001).
- **WSL2 Ubuntu** runs everything else: corpus generation (Jinja2→WeasyPrint→pypdfium2, seeded NumPy degradations), extraction (text-layer first, RapidOCR fallback), the hand-rolled asyncio orchestrator, verifier gate, and analysis. No agent framework — the R0→R3 escalation ladder *is* the experiment, and a framework's retry logic would contaminate rung accounting.
- **MacBook Pro M4** is a second measured anchor (llama-server/Metal) for the projection model, with real power via `powermetrics` (ADR-0010).
- **Anthropic API** (Sonnet 5 mid-tier, Opus 5 frontier) is used only for R2/R3 escalation, with a hard spend guard and the Batch API for bulk accuracy passes (ADR-0007).

Data flow: every document attempt lands as one record in an **append-only `runs/<run_id>/trajectory.jsonl`** — the system of record. DuckDB reads it in place through ~12 named views, and one command renders **two HTML reports from the same views** (`report-operator.html` full detail, `report-executive.html` two-screen verdict). Neither rendering computes anything, so they cannot disagree (ADR-0012, `docs/report-composition.md`). `run_id` is a blake2b hash of seed + config + uv.lock + model SHA256 + llama.cpp build hash; resume means "read the JSONL, skip what's already there."

Measurement is split into three passes (ADR-0009): Pass A measures accuracy at concurrency 1 (f16 KV, temp 0), Pass B sweeps throughput at c∈{1,8,32,128} (q8 KV where needed), and Pass C replays a subset to check that batching is accuracy-neutral — either outcome of Pass C is publishable.

## Invariants that must not be violated

These recur across the docs and are the credibility of the result; breaking one silently is the project's main failure mode.

- **Gate decision and correctness check are computed independently and both written.** `outcome` is their cross-product; `SILENT_FAILURE` = gate passed × answer wrong. Nothing may collapse those two axes. Silent-failure rate is the first thing on every report, standalone, never averaged into aggregate success.
- **Schema changes are additive only.** No field renamed, retyped, or removed once frozen.
- **Every number carries a provenance badge** — measured, modeled, or projected. The Mac measures power while the test rig models it; that asymmetry is labelled, never smoothed.
- **Logprobs under grammar-constrained decoding are post-mask** and systematically over-confident. Calibration analysis must segment on `attempts[].grammar_constrained` or the reliability diagram is wrong.
- **Rates are never hardcoded in logic.** `config/rates.toml` holds dated rate cards; every record pins `cost.rate_card_id`, and cross-run cost comparisons assert matching ids. (Sonnet 5 intro pricing expires 2026-08-31, inside the build window.)
- **Offered concurrency ≠ server slots.** Both are logged and both reported, so a queued-128 result is never read as a batched-128 result (§6.2).
- **Batch API latency is flagged** (`attempts[].via_batch_api`) and excluded from every latency chart; cloud concurrency above 8 is reported as "not measured, account-tier limited," never silently omitted.
- **Energy is demoted** (ADR-0011): documents-per-hour-per-machine is the headline; tokens-per-joule is an appendix figure with a sensitivity band. Don't re-promote it.
- **Reports are deterministic**: same `run_id`, byte-identical output.
- **Scope discipline** (architecture §12): no live CRM, no fine-tuning, no quantization sweep, no auth, no multi-tenancy, no NPU path, no agent framework. Adding any of these before the Phase 2 exit gate is drift.

## Documentation map

- `docs/architecture.md` — system design, the four measurement problems, build sequence. Start here.
- `docs/tradeoffs.md` — 12 severity-rated tradeoffs, each with its reversal condition and any caveat text the report must carry.
- `docs/decisions/ADR-00NN-*.md` — 12 ADRs; treat as binding. ADR-0011 supersedes ADR-0008's framing (energy), ADR-0012 extends ADR-0004 (two renderings).
- `docs/report-composition.md` — composition rules for both reports, including the metric→business-unit translation table for the executive rendering.
- `docs/specimens/report-specimens.html` — design reference for the reports. **Every number in it is invented**; it carries a warning banner and must never reach a client. Exhibits are named for the question they answer, never "Cut A"/"Cut B".
- `config/experiment.example.toml`, `config/hardware.toml`, `config/rates.toml` — sweep definition, tier ladder + power model, dated rate cards. Everything in the experiment config folds into `run_id`.
