# ADR-0005 — Orchestration: asyncio + httpx, semaphore concurrency, JSONL-ledger resume

**Status:** Accepted · **Delegated by:** PRD §5 · **Date:** 2026-08-21

## Context

P0-10 requires a single-command three-arm runner whose runs are resumable and individually reproducible from seed + config hash. P0-9 requires measurement at concurrency {1, 8, 32, 128}. Two serving nodes are now in play (ADR-0010).

## Decision

- **`asyncio` + `httpx.AsyncClient`**, one `Semaphore` per node sized to that node's offered concurrency.
- **Hand-rolled escalation state machine** for R0–R3.
- **Resume by ledger:** read `trajectory.jsonl`, skip any `doc_id` already present, append the rest. No checkpoint file.
- **Spend guard:** a running total of `cloud_usd` aborts the run at a configured ceiling.

## Why not Ray / Celery / Prefect

Two nodes and one queue. A broker, scheduler, or cluster runtime is pure setup cost against constraint C1 (minimize time to configure).

## Why not LangGraph / CrewAI for the agent loop

The escalation ladder is the object of study. A framework's built-in retry logic would silently contaminate rung accounting — the single most important measurement in the project. Instrumenting around a framework costs more than writing the state machine. See tradeoff T9.

## Concurrency semantics

Two distinct numbers, both recorded, never conflated (see `architecture.md` §6.2):

- **Offered concurrency** — in-flight requests from the client; the P0-9 axis.
- **Server slots** — `--parallel N` actually resident; the real batching width.

Above `slots`, load queues rather than batches. Reporting a queued-128 result as a batched-128 result would be the easiest serious error to make here.

## Consequences

- **Positive:** append-only JSONL means killing the process mid-run is always safe. This matters on a personal machine that reboots for Windows updates.
- **Positive:** the spend guard makes a runaway retry loop a bounded cost rather than a surprise.
- **Negative:** no distributed scheduler. Adding a third node means another semaphore and a round-robin entry — acceptable at this scale.
- **Negative:** the state machine is ~300 lines that a framework would have supplied. Worth it.

## Revisit if

The node count grows past three or four, at which point a real work queue starts to earn its setup cost.
