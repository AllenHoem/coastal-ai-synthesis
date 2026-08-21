# ADR-0004 — Analysis: DuckDB over JSONL, static self-contained HTML report

**Status:** Accepted · **Delegated by:** PRD §5 · **Date:** 2026-08-21

## Context

PRD §5 delegates dataframe library, notebook-vs-script, and chart toolkit. P0-12 fixes the required outputs. The operator's stated requirement is a *simple* means to review and extract results, on a home DHCP network with no external tunnel.

## Decision

- **DuckDB** queries `trajectory.jsonl` in place via `read_json_auto`. No import step, no ETL.
- **~12 named SQL views** in `analysis/views.sql` are the analysis interface. Every consumer reads views, not raw records.
- **matplotlib → inline SVG**, embedded in a **Jinja2** template, producing **one self-contained `report.html`**.
- Exports alongside it: `summary.csv`, `trajectories.parquet`, `results.duckdb`.

## Why not Streamlit

Adds a process to keep running and a port to expose. The audience is a skeptical technical reviewer and eventually a CFO — both want a document they can keep, not a URL that may not resolve tomorrow. On a DHCP LAN with no tunnel, a file is strictly more portable than a service.

## Why not a notebook

Weakest option for reproducibility and for handing a clean artifact to a third party. PRD §9's secondary metric — "degradation curve reproduced by a third party from spec alone" — argues for a build output, not a session someone ran once.

## Why not pandas/polars as the primary

DuckDB reads nested JSONL natively, which matters because the trajectory record is deeply nested. SQL views are also more legible to a reviewer than dataframe chains.

## Non-negotiable report properties

1. **Self-contained** — inline SVG, no CDN, no external asset. Opens from an email attachment.
2. **Provenance badges** — every number visibly marked measured, modeled, or projected. Required by the modeled-energy decision (ADR-0008).
3. **Silent-failure rate first**, standalone, never averaged into anything (PRD D9).
4. **Deterministic** — same `run_id` yields a byte-identical file.

## Consequences

- **Positive:** `make report RUN=…` is the whole interface. Nothing to administer.
- **Positive:** ad-hoc exploration is `duckdb results.duckdb` plus SQL against named views.
- **Negative:** no interactive filtering. A hypothesis the report didn't anticipate means writing a query.

## Revisit if

Exploration becomes the bottleneck. A Streamlit app over the same views is an afternoon's work — the views are the interface, the report is just one consumer of them.
