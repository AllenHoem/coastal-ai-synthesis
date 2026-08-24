# ADR-0010 — MacBook Pro M4 as a second measured anchor for the projection model

**Status:** Accepted · **Date:** 2026-08-21

## Context

PRD §7 is normative: the projection model must be shown to predict test-rig performance within ±15% *before* being used to project to hardware that was not measured. Step 3 of §7 is what makes the projection defensible in front of a CFO.

With one anchor, that validation is a line fitted through a single point. The operator has a MacBook Pro M4 with 24GB unified memory available on the same network.

## Decision

Run a second `llama-server` (Metal backend) on the MacBook and treat it as a **measured** anchor, not a projection target.

| | Test rig | MacBook |
|---|---|---|
| Bandwidth | ~800 GB/s | ~120 GB/s (M4 base) |
| Usable model memory | 20GB physical, ~18.5GB usable | ~20GB of 24GB unified, via `iogpu.wired_limit_mb` |
| Tier | T2 | **lands on T1** (ladder says ~110–135 GB/s) |
| Power metering | modeled (ADR-0008) | **measured** via `powermetrics` |

The M4 runs the 1.7B, 4B, 8B, and 14B rungs of the D7 ladder comfortably; 30B-A3B at Q4 is ~18GB and viable only with the wired limit raised.

## Why this matters more than it looks

Two anchors spanning a ~7x bandwidth range is a qualitatively different claim from one. A model fitted to one point and extrapolated is a line through a single dot. Fitted to two points an octave apart, it has been **falsifiable at least once** — and survived. This is the largest available credibility upgrade in the project for roughly half a day of work.

It also converts T1 from projected to measured, which is the tier the PRD's central question most often lands on (§9 target: minimum viable model size ≤ 8B = T1).

And it supplies the project's only free power metering, partially offsetting ADR-0008.

## Verify before relying on the tier label

`make probe-node NODE=mac` measures bandwidth rather than assuming it. An M4 **base** is ~120 GB/s and lands on T1. An M4 **Pro** is ~273 GB/s — T3 bandwidth with T1 capacity, which is a different and arguably more interesting anchor. The projection model fits from measurement either way; only the narrative changes.

## Consequences

- **Positive:** two-point validation of a normative model. T1 measured. One measured power anchor.
- **Positive:** zero new code — the node is another base URL and another semaphore (ADR-0005).
- **Negative:** ~half a day for setup, LAN reachability, and characterization.
- **Negative:** a second `--parallel`/context configuration to keep straight. Handled by `config.host` on every record.
- **Negative:** macOS `iogpu.wired_limit_mb` does not persist across reboot. Runbook notes it; a launchd plist fixes it if it becomes annoying.

## Revisit if

The probe shows the machine is thermally throttling under sustained load, which would make it a poor throughput anchor even while remaining a fine accuracy and bandwidth anchor. Record sustained-vs-burst separately if so.
