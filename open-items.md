# Open items

Everything outstanding across the bridge program: things deliberately postponed,
and questions not yet resolved.

This is the one file here that is **rewritten rather than appended** — items are
added, closed and removed as they resolve. Keep it short enough to read at a
glance; if it stops being scannable, it has stopped doing its job.

Mechanisms and gotchas live in [`learning-log.md`](./learning-log.md). Decisions
live in [`adr/`](./adr/).

**Last reviewed:** 14-Sep-2026

---

## Deliberate deferrals

Choices, not oversights. Each has a reason for waiting and a trigger for revisiting.

| # | Deferred | From | Until | Why not now |
|---|---|---|---|---|
| D1 | Pin `ubuntu-24.04` instead of `ubuntu-latest` | 00 | Supply-chain thread | Job depends only on `curl`, `tar`, `sha256sum`, `sudo` — nothing for an LTS bump to break |
| D2 | SHA-pin `actions/checkout` instead of `@v6` | 00 | Supply-chain thread | First-party GitHub action; same reasoning as D1 |
| D3 | `aws-actions/configure-aws-credentials@v5` | 00 | Week 10 | Runner already forces Node 24 and it works; the real deadline is unannounced |
| D4 | OIDC trust scoped to `job_workflow_ref` | 00 | When the set of workflows needing AWS access stabilises | Would block any new workflow on `main` from assuming the role |
| D5 | `data.aws_caller_identity` precondition in Terraform | 00 | Second AWS account exists | One account, nothing to confuse it with |
| D6 | Remote Terraform state backend | 00 | Second machine or person applies changes | Local state has no locking and no history |
| D7 | `tflint`, `tfsec`/`trivy` in `cloud-infra` | 00 | Cloud deployment thread | gitleaks does not catch an over-broad IAM policy or an unencrypted bucket |
| D8 | Langfuse as a trace backend | 00 | Observability thread | Comparing backends by swapping the collector exporter is the exercise there |
| D9 | Terraform environment directories and modules | 00 | Second environment or second consumer | A module with one caller is indirection without reuse |
| D10 | Qdrant API key authentication | 00A | Multi-tenancy work | Breaks existing clients; enabling it is part of that topic. Local instance confirmed not reachable from the LAN |
| D11 | Kafka client config made conditional — PLAINTEXT local, TLS + IAM/SASL on MSK | 00A | Service rewrite threads | Touching the services would have ballooned broker scope |
| D12 | Vector DB multi-tenancy implementation | 00A | Interview/revision phase | Study to interview depth only; building the negative-authorization test doubles as rehearsal material |

## Carried-forward work

Items that were in a thread's scope and did not complete.

| # | Item | From | To | Blocker |
|---|---|---|---|---|
| C1 | Docker build and push to ECR with git-SHA tags | 00 | 03 | `news-fetch-scheduler`'s Dockerfile installs `py_commons_per` from a local package index unreachable from a CI runner |
| C2 | `py_commons_per` installed from its public GitHub URL rather than the local index | 00 | 03 | Removes a dependency on infrastructure only the development machine has |
| C3 | AWS Budget alert confirmed by email | 00 | First real spend | Creating a budget does not send an alert. Marked provisionally complete |
| C4 | `news-fetch-scheduler` Dockerfile fixes — missing runtime `WORKDIR`, bare `uvicorn` with no `--host 0.0.0.0`, orphaned `PYTHONPATH` comment | 00 | 03 | Service is rewritten there anyway |

## Open questions

Unresolved, and each one has a cost if it turns out badly.

| # | Question | From | Why it matters |
|---|---|---|---|
| Q1 | `gen_ai.*` semantic convention stability — pinned at a beta release, development status | 00 | Attribute names may be renamed, breaking dashboards and assertions |
| Q2 | X-Ray free tier limit — 100,000 traces/month recorded is the figure on hand, unverified against current pricing | 00 | Determines whether AWS tracing is free at eval volume |
| Q3 | TEI `start_period` for larger models — 90s covers BGE-small on a warm cache | 00 | Qwen3-Embedding-0.6B and BGE-M3 on a cold volume may exceed it and flap to unhealthy |
| Q4 | Qdrant JWT payload filters — removed in 1.16, but a later discussion thread suggests otherwise | 00 | **A token lacking the expected restriction is a security failure even when application-level filtering appears to work.** Verify against the running version before writing any token |
| Q5 | Trace context propagation through Kafka headers | 00A | Without it the pipeline produces three disconnected traces instead of one |
| Q6 | `kafka-python` maintenance state — assumed less actively maintained than `confluent-kafka-python`, unverified | 00A | Currently a non-issue; `confluent-kafka-python` retained |

---

## Closed

Moved here rather than deleted, so a decision that was reversed is visible.

*(none yet)*