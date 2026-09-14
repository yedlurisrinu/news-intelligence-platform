# ADR-0001: One repository per independently deployable artifact

- **Status:** Accepted
- **Date:** 13-Sep-2026

## Context

The News Intelligence Platform began as a single repository, `ai`, holding
five Git submodules. The submodules were always independent top-level
repositories; `ai` stored only pointers. The arrangement made the services
appear nested on GitHub while giving none of the benefits of a monorepo —
no shared tooling, no atomic cross-service commits, no unified CI.

The services deploy independently. A fetch scheduler, a Kafka consumer and
a RAG agent have separate release cadences and separate runtime concerns.

Naming was also inconsistent: some repositories used underscores, others
hyphens.

## Options considered

### A — Monorepo
One repository containing all services. Atomic cross-service changes,
single CI configuration. Costs: every service shares a release cadence
unless tooling is built to avoid it, and CI must learn which paths trigger
which jobs.

### B — Repository per deployable artifact
Each service owns its repository, plus a separate index repository holding
architecture, compose, and decision records. Costs: cross-service changes
span multiple pull requests; shared CI must be centralised deliberately.

### C — Keep submodules
Preserves the appearance of a single project. Costs: clones without
`--recurse-submodules` yield empty directories, pointer commits carry no
content, and the parent repository must be updated whenever a child moves.

## Decision

**Option B.** One repository per independently deployable artifact:

| Repository | Role |
|---|---|
| `agent-platform-kit` | Reusable substrate — tracing, evals, secrets |
| `ci-workflows` | Reusable GitHub Actions workflows |
| `cloud-infra` | Terraform, private |
| `news-intelligence-platform` | Index: architecture, compose, ADRs |
| `news-fetch-scheduler` | Pipeline — fetch |
| `news-consumer-ingester` | Pipeline — ingest |
| `news-rag-agent` | LangGraph + FastAPI streaming agent |
| `reno-compass` | Second capstone |

Repository names use hyphens throughout. Python package names retain
underscores, which is a language constraint and not a naming inconsistency.

`news-rag-agent` is the existing `news_search_summary_app`, renamed rather
than rebuilt. The capstone strategy is to harden existing work, and a
repository with no history would contradict that.

`ai` was archived and made private rather than renamed to
`news-intelligence-platform`. Its history begins with a task-management
multi-agent application, not the news platform, so renaming would have
given the index repository a history describing a different project.

`ci-workflows` was added as a horizontal concern once a second repository
needed the same CI definition.

## Consequences

- Cross-service changes require coordinated pull requests. Accepted; the
  services are independently deployable and rarely change together.
- Shared CI must be centralised explicitly. `ci-workflows` exists for this
  and hosts reusable workflows invoked with `workflow_call`.
- A change to a shared workflow pinned at `@main` takes effect in every
  calling repository immediately, including a mistake. Accepted for a
  single-maintainer setup; the failure is loud and cheap to revert.
- The archived `ai` retains stale submodule pointers. Harmless on a
  read-only repository.
- Anyone cloning `agent-platform-kit` must be able to run it without the
  maintainer's local infrastructure. This constrains how its compose file
  and secret handling are written.

## Verification

All eight repositories exist with the names above. `gitleaks` runs as a
pre-commit hook and as a CI gate in each active repository, invoked from
`ci-workflows`, with scanned-commit counts reconciled against repository
history to confirm the scans cover full history rather than a shallow
checkout.
