# ADR-0006 — Repo layout, packaging, and config format

**Status:** Accepted · **Delegated by:** PRD §5 · **Date:** 2026-08-21

## Context

PRD §5 delegates repo layout, packaging, and config format. P0-10 requires runs reproducible from seed + config hash.

## Decision

- **`uv`** for dependency management and virtualenv. **`src/` layout**, single package `edgefirst`.
- **TOML** config parsed with stdlib `tomllib`, validated by **Pydantic** models.
- Three config files with distinct lifecycles: `experiment.toml` (changes per sweep), `rates.toml` (changes when a vendor changes pricing), `hardware.toml` (changes when a node changes).
- Secrets in `.env`, never in TOML, never committed.
- **`run_id` = blake2b** over seed, config, `uv.lock`, model file SHA256s, llama.cpp build hash, and corpus SHA.
- **`Makefile`** is the entire operator interface.

## Why TOML over YAML

No parser dependency, no significant-whitespace failures, no YAML type-coercion surprises (the `NO`-parses-as-`false` class of bug). Canonical serialization for hashing is straightforward.

## Why split the config files

They change on different clocks. Folding a rate-card update into the experiment config would change `run_id` for every run and make cost comparisons across runs look like configuration drift.

## Why the lockfile is in the hash

An OCR or PDF library upgrade can change extraction output. If the lockfile isn't in `run_id`, two runs with identical configs can silently differ and nothing records why.

## Consequences

- **Positive:** `uv sync` is a cold install in seconds.
- **Positive:** `run_id` collision implies genuinely identical inputs, so resume and cache are safe.
- **Negative:** any dependency bump invalidates prior `run_id`s. Correct, but it means dependency updates should be deliberate and batched, not incidental.

## Revisit if

Never expected to. This is the least consequential decision in the set.
