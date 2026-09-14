# ADR-0002: OTLP to a local collector, with Jaeger as the first backend

- **Status:** Accepted
- **Date:** 13-Sep-2026

## Context

The platform needs distributed tracing across a fetch scheduler, a Kafka
consumer and a RAG agent, with LLM-specific attributes — model, token
counts, provider — attached to spans. OpenTelemetry defines these under
the `gen_ai.*` semantic conventions, which are still development-status
and subject to rename.

The original plan specified Langfuse self-hosted as the local backend.
Two constraints made that worth re-examining: a $300 program budget, and
the goal of working at the protocol level rather than through a vendor
abstraction.

## Options considered

### A — LangSmith
Near-zero instrumentation for a LangGraph stack. Rejected: the free tier
allows 5,000 traces per month with 14-day retention and one seat, and
self-hosting requires an Enterprise plan. Evaluation runs of 150–200
golden cases gated in CI would exhaust a month's allowance in under three
days, and trace evidence from early threads would expire before it could
be used for review.

### B — Langfuse self-hosted
Open source, no licence cost, OTLP ingestion. Rejected as the *first*
backend: it requires Postgres, ClickHouse, Redis and S3-compatible blob
storage alongside web and worker processes. AWS has no managed ClickHouse
equivalent, so an AWS deployment would mean operating ClickHouse directly
at a cost exceeding the program budget on its own.

### C — Arize Phoenix
Single container, LLM-aware UI. Rejected: its native vocabulary is
OpenInference rather than OTel `gen_ai.*`. Spans arrive and are visible,
but its specialised views key off the other convention, which obscures
the convention being learned.

### D — OTel Collector with Jaeger
Two containers, roughly 400 MB. Jaeger renders arbitrary span attributes
without opinion. On AWS the collector's exporter targets X-Ray, which is
managed and requires no containers.

## Decision

**Option D.** The application emits OTLP to a local OpenTelemetry
Collector. The collector exports to Jaeger locally and will export to
AWS X-Ray in the cloud environment. Application code is identical in both
cases; only the collector's exporter configuration changes.

Langfuse is deferred to the observability thread, where swapping the
collector's exporter and comparing backends is the exercise. Running
Langfuse in both environments would demonstrate nothing about portability —
it would be the same backend twice.

The tracing bootstrap reads its endpoint from `OTEL_EXPORTER_OTLP_ENDPOINT`,
so backend choice is a deployment concern rather than a code concern.

`build_provider` constructs a `TracerProvider` without installing it as the
process-wide default; `configure_tracing` builds and installs, guarded for
idempotency. Installing twice would stack span processors and duplicate
every span.

## Consequences

- The originally stated exit criterion — a `gen_ai.*` trace reaching local
  Langfuse — is restated as reaching the local OTel trace backend.
- Jaeger has no LLM-aware rendering. `gen_ai.*` attributes appear as
  ordinary span tags. For learning the convention this is a feature; for
  reading an agent trace quickly it is not.
- Migrating to X-Ray or Langfuse requires no application change, which is
  the property this decision exists to obtain.
- The `gen_ai.*` conventions are development-status. Attribute names may
  change, and the semantic-conventions package is pinned at a beta version.
- The collector image is distroless, so it carries no shell or HTTP client
  and cannot run a Docker healthcheck. Health is observable on its own port
  from outside the container. Nothing in Compose can wait on collector
  readiness.

## Verification

A hand-constructed OTLP span carrying `gen_ai.system` and
`gen_ai.request.model` was posted to the collector's HTTP receiver and
confirmed present in Jaeger via its API and UI.

The Python bootstrap was then exercised end to end, producing the same
attributes from application code through the collector into Jaeger.

Four unit tests assert the contract without a running collector, using an
in-memory exporter: span export, `gen_ai.*` attribute survival, resource
identity, and parent-child span linkage.

One incident during verification is worth recording: the first span was
delivered successfully but invisible in the UI, because its hardcoded
timestamps placed it a year in the past. The API confirmed delivery while
the interface showed nothing.
