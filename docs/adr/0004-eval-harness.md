# ADR-0004: Evals as pytest, separated from unit tests by trigger

- **Status:** Accepted
- **Date:** 13-Sep-2026

## Context

Agent behaviour needs to be measured against golden cases, and the result
needs to gate changes. Roughly 150–200 cases per capstone are anticipated.

Evals differ from unit tests in ways that matter operationally. They call
live models, so they cost money per run. They are non-deterministic, so
they assert against thresholds rather than exact values. They are slow.
A unit test that fails means the code is wrong; an eval that fails may
mean the model drifted, the prompt changed, or the threshold was
optimistic.

Running both under one command means either the unit tests inherit the
cost and flakiness, or the evals are never run.

## Options considered

### A — A bespoke eval framework
Full control over reporting, thresholds and case management. Costs:
everything pytest provides — fixtures, parametrisation, selection,
reporting, CI integration — has to be rebuilt.

### B — A third-party eval library
Purpose-built abstractions for LLM evaluation. Rejected for the same
reason the observability backend was chosen at the protocol level: the
goal is to understand the mechanics, not to adopt an abstraction over
them. A library can be introduced later once the mechanics are known.

### C — pytest, with evals and unit tests in one suite
Simplest structure. Rejected: no way to give the two categories different
triggers, so either paid model calls run on every push or they never run
in CI at all.

### D — pytest, with evals and unit tests separated
Two directories, an `eval` marker, and distinct CI workflows.

## Decision

**Option D.**

`tests/unit` holds deterministic
