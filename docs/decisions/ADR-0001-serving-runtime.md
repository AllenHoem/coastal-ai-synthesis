# ADR-0001 — Serving runtime: llama.cpp `llama-server`, Vulkan, Windows-native

**Status:** Accepted · **Delegated by:** PRD §5 · **Date:** 2026-08-21

## Context

PRD P0-4 sets two hard requirements: per-token logprobs, and parallel request slots. §5 names llama.cpp and vLLM as candidates and instructs verification before committing.

The test rig is an AMD RX 7900 XT (RDNA3, Navi 31 — 20GB, 320-bit, ~800 GB/s, 315W) in a Windows 11
machine with WSL2. The CPU also carries integrated graphics, so Vulkan enumerates two devices.

## Decision

**llama.cpp `llama-server`, Vulkan backend, running natively on Windows.** WSL2 is the orchestration client, reaching it over loopback.

## Why not vLLM

- No Windows build.
- ROCm vLLM targets CDNA/Instinct. RDNA3 consumer support is experimental at best.
- Reaching it would require ROCm inside WSL2, which AMD does not support for consumer RDNA3.

## Why not ROCm-native or Vulkan-in-WSL

- ROCm on Windows via the HIP SDK works, but the install is heavy and llama.cpp's HIP builds for RDNA3 have lagged Vulkan on stability.
- Vulkan compute does not pass through to WSL2 on AMD.

## Verification required before Phase 1

Runbook Step 3 gates all three, and a failure on the first stops the project rather than routing around it:

1. `/v1/chat/completions` with `logprobs: true` returns non-null `content[].logprob`.
2. Eight concurrent requests complete in well under 8x a single request.
3. `temperature: 0` with a fixed seed produces byte-identical completions across three runs.
4. `--list-devices` confirms which index is the discrete GPU, and it is pinned with `--device`.
   Selecting the integrated GPU fails silently into either an allocation error or throughput an
   order of magnitude low, with nothing in the log naming the cause.

## Consequences

- **Positive:** prebuilt binary, driver already present, ~20 minutes to first token. GBNF and JSON-schema grammars available for the tool layer. `--metrics` exposes Prometheus counters for throughput.
- **Negative:** llama.cpp's continuous batching is less efficient than vLLM's paged-attention scheduler. High-concurrency throughput will understate a well-served T3 appliance. This is conservative for the thesis — see tradeoff T1.
- **Negative:** `--ctx-size` is divided across `--parallel` slots, so concurrency is KV-bound. See `architecture.md` §6.2.
- **Negative:** WSL has no GPU, so OCR and analysis are CPU-only. Arguably more representative of T0 anyway.

## Revisit if

The operator gains access to an NVIDIA card or a CDNA/Instinct part, at which point vLLM becomes viable and the concurrency ceiling should be re-measured before any T3 claim is finalized.
