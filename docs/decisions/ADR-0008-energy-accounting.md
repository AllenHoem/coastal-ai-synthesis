# ADR-0008 — Energy accounting: modeled from spec sheets, with a measured seam

**Status:** Accepted (operator decision) · **Date:** 2026-08-21

## Context

P0-10 requires local energy accounting; P0-12 requires tokens-per-joule compared against the PRD §2 baseline. That baseline table is quoted in **system watts** (Strix Halo ~130W, DGX Spark ~150–170W).

Options were real wall metering via a LAN smart plug, GPU-rail telemetry, or modeling from spec sheets. **The operator elected to model from spec sheets.**

## Decision

Power is derived from a configurable model in `config/hardware.toml` — idle draw, TDP, and a duty-cycle coefficient per node — rather than measured on the test rig.

Three mitigations are mandatory, not optional:

1. **`cost.energy_source` is a required trajectory field:** `modeled` | `measured_gpu` | `measured_wall`. Without it, modeled and measured figures become indistinguishable the moment someone opens the CSV.
2. **The MacBook is metered for free.** `powermetrics` reports real package power on Apple Silicon, so the T1 anchor's tokens-per-joule is *measured*. This gives one honest calibration point against the modeled figures.
3. **`PowerSource` is an interface** with `ModeledPower` (default), `LibreHardwareMonitorPower` (GPU rail via its JSON endpoint, ~20 min to wire), and `SmartPlugPower` (wall watts, ~$20 + 30 min). Swapping is a config line; no rework.

## Scope of the consequence

Stated precisely, because it is narrower than it first appears:

| Affected | Not affected |
|---|---|
| Tokens-per-joule row of P0-12 | **All of PRD §7** — bandwidth projection is independent of power |
| The §2 baseline comparison | Confusion matrix; silent-failure rate |
| Local component of cost-per-record | Cloud cost (billed, exact) |
| | Minimum viable model size |
| | Every PRD §9 primary metric except cost-per-record's local half |

One row of P0-12 and one component of one §9 metric. The projection model — the thing a CFO will actually interrogate — is untouched.

## Consequences

- **Positive:** zero setup, zero purchase.
- **Negative:** tokens-per-joule is an estimate. Comparing a TDP-derived figure to the §2 table's *system* watts is not defensible unless said out loud — so the report says it on the chart, not in a footnote.
- **Negative:** the modeled/measured asymmetry between the two nodes must be labelled everywhere it appears, never smoothed.

## Revisit if

Tokens-per-joule stops being a supporting figure and becomes an argument in a client conversation. A Kasa KP115 or Shelly Plus PlugS measures the exact quantity the §2 table uses. Buy it *before* that conversation, not after. See tradeoff T2.
