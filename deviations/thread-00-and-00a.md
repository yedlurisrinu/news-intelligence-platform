# Plan deviations — Thread 00 and 00A

Changes to `bridge-program-plan`, `bridge-decision-register` and
`bridge-repo-and-conventions` arising from execution. Intended to be fed back
into the planning thread so the source documents can be updated.

**Covers:** 12-Sep-2026 to 14-Sep-2026
**Status:** Thread 00 and 00A complete, with two items carried forward

---

## 1. Corrections to `bridge-repo-and-conventions` §1

| # | Recorded | Actual |
|---|---|---|
| 1.1 | GitHub handle `syedluri` | **`yedlurisrinu`** — affects every repo URL, the OIDC trust policy, and the profile README repo name |
| 1.2 | `news-rag-agent` listed as a **new** repo | Renamed from the existing `news_search_summary_app`. A repo with no history contradicts the harden-existing-projects strategy |
| 1.3 | `news_search_summary_app` absent entirely | It is the LLM-facing agent application and the third News App service. Omitted because it was never surfaced during planning |
| 1.4 | Kafka listed in the `agent-platform-kit` compose stack | Moved to `docker_services`. Kafka is shared infrastructure consumed by separate service repos; the kit's compose holds only what the kit itself needs |

## 2. Repository additions not in the plan

| # | Repo | Visibility | Purpose |
|---|---|---|---|
| 2.1 | `ci-workflows` | Public | Reusable GitHub Actions workflows — gitleaks, unit tests, evals. Horizontal concern, separated from the agent substrate |
| 2.2 | `cloud-infra` | **Private** | Terraform, with `aws/` and `gcp/` top-level directories |

## 3. Repository restructuring

| # | Action | Detail |
|---|---|---|
| 3.1 | `ai` archived and made private | Not renamed to `news-intelligence-platform` as first considered. Its history begins with a task-management multi-agent app, not the news platform |
| 3.2 | `news-intelligence-platform` created fresh | Holds architecture, compose, ADRs |
| 3.3 | `news_fetch_scheduler` → `news-fetch-scheduler` | Naming normalised to hyphens across all repos |
| 3.4 | `news_search_summary_app` → `news-rag-agent` | Per 1.2 |
| 3.5 | `postgres` and `pypi-server` repos archived | Were submodules of `docker_services`; content converted to tracked directories there |
| 3.6 | `todo_crew_app`, `todos-api` | Left as-is. Early agentic work, out of program scope but retained as evidence of prior progress |

**Final repository set (8 active):** `agent-platform-kit`, `ci-workflows`,
`cloud-infra`, `news-intelligence-platform`, `news-fetch-scheduler`,
`news-consumer-ingester`, `news-rag-agent`, `reno-compass`, plus `docker_services`
as running infrastructure.

## 4. Schedule

| # | Item | Detail |
|---|---|---|
| 4.1 | Thread 00 budgeted Sep 11–12 (2 days) | Ran Sep 12–14. Sep 11 was consumed by planning; execution began Sep 12 |
| 4.2 | Thread 00A budgeted 0.5 day, Sep 13 | Ran Sep 14, after Thread 00 completion |
| 4.3 | Net position | Approximately 1.5 days behind at the end of week 1 of 10 |

**Causes of overrun, all unbudgeted:** the repository restructuring (§3), a
full-history secret audit expanded from 3 repos to 11 (§6.2), and Terraform
adoption pulled forward from the cloud deployment thread (§7.1).

**Partial offset:** Terraform adoption means some cloud-thread work is already
done. Threads 01 and 02 are reading-heavy and historically compress.

## 5. Observability — supersedes decision A9 scope

| # | Change |
|---|---|
| 5.1 | Langfuse deferred from Thread 00 to the observability thread. Local backend is OTel Collector + Jaeger |
| 5.2 | AWS target named as **X-Ray**, not previously specified. Managed, no containers, and the collector exporter is the only thing that changes |
| 5.3 | **Exit criterion restated:** "a `gen_ai.*` trace lands in local Langfuse" → "lands in the local OTel trace backend" |
| 5.4 | LangSmith retained for comparison only, unchanged from A9 |

**Reason:** Langfuse self-hosted requires Postgres, ClickHouse, Redis and
S3-compatible storage plus web and worker processes. AWS has no managed ClickHouse
equivalent, so an AWS deployment would exceed the entire program budget alone.
Running Langfuse in both environments would also demonstrate nothing about
portability — it would be the same backend twice.

Recorded as ADR-0002.

## 6. Security posture

| # | Change |
|---|---|
| 6.1 | **`bridge-program-plan` §9 is wrong.** The "private until Oct 8" window does not exist — the News App repos and `reno-compass` were already public. For anything already committed, rotation is the only remedy |
| 6.2 | Full-history secret audit moved from October to Thread 00, and scope expanded from the News App repos to **all 11 public repositories**. Result: zero findings, every scanned-commit gap accounted for |
| 6.3 | gitleaks run as a **raw binary** in CI rather than `gitleaks-action` — avoids a third-party action in the supply chain and a licensing dependency if repos ever move into an organisation |
| 6.4 | **`aws login`** used for human CLI access instead of IAM access keys. Not in the plan; issues temporary credentials via an existing browser session, so no key material exists on disk |
| 6.5 | Secrets architecture confirmed as a provider abstraction — `EnvSecretProvider` (default), `VaultSecretProvider`, `AwsSecretsManagerProvider` — honouring decision D4 |
| 6.6 | GitHub OIDC confirmed and verified end to end. No long-lived AWS credentials in GitHub |

## 7. Infrastructure as code

| # | Change |
|---|---|
| 7.1 | **Terraform adopted in Thread 00**, not the cloud deployment thread. ECR repositories, lifecycle policies, the OIDC provider and the CI role are all managed |
| 7.2 | The pre-existing ECR repository was **imported** rather than destroyed and recreated |
| 7.3 | Layout deliberately flat — no `environments/` directories, no modules — until a second environment or consumer exists |
| 7.4 | Local state, gitignored. Remote backend deferred |

Recorded as ADR-0003.

## 8. Evals

| # | Change |
|---|---|
| 8.1 | `tests/unit` and `tests/evals` are **separate trees with separate workflows**, not one suite with markers |
| 8.2 | The eval workflow is **`workflow_dispatch` only from the outset**, while the harness is still a free placeholder — so no later change is required to prevent billing on every push |
| 8.3 | Model and sample size are pytest CLI options via `conftest.py`, surfaced as workflow inputs |
| 8.4 | Tracing assertions are **unit tests**, not evals. The span contract is deterministic |

Recorded as ADR-0004.

## 9. Kafka — supersedes the Thread 00A framing

| # | Change |
|---|---|
| 9.1 | **Confluent Cloud dropped entirely.** Local broker for development, MSK for AWS |
| 9.2 | `apache/kafka` 4.0 in KRaft mode, single node. ZooKeeper was removed in Kafka 4.0, so this is one container |
| 9.3 | Kafbat UI, not Provectus — Provectus paused development and the maintainers moved to the fork |
| 9.4 | `confluent-kafka-python` retained as the client. It wraps librdkafka and speaks the open protocol against any conforming broker |
| 9.5 | Kafka lives in `docker_services`, per 1.4 |

Recorded as ADR-0005.

## 10. Scope additions

| # | Addition |
|---|---|
| 10.1 | **Vector database concepts** added to the technology bridge list: indexing internals (HNSW parameters, recall/latency), quantization and memory sizing, hybrid search and reranking, chunking strategy, filtering and payload indexing, multi-tenancy, access control |
| 10.2 | Placement: indexing, quantization, chunking, hybrid search and filtering fold into the embedding bake-off thread, where a corpus and measurement harness will already exist. Multi-tenancy and access control are **study to interview depth**; implementation deferred to the interview/revision phase |
| 10.3 | Qdrant authentication enablement added to the multi-tenancy work. The local instance currently runs without authentication — confirmed not reachable from the LAN |

## 11. ADR numbering — supersedes the plan

| ADR | Subject | Note |
|---|---|---|
| 0001 | One repository per independently deployable artifact | As planned |
| 0002 | Local trace backend | New |
| 0003 | Infrastructure as code | New |
| 0004 | Eval harness | New |
| 0005 | Kafka broker | **Was 0003 in the plan** |

**Location:** `news-intelligence-platform/docs/adr/`, not `agent-platform-kit`.
Repository layout, observability backends and message brokers are platform-level
concerns rather than library concerns.

**Template:** includes a non-standard **Verification** section recording how the
decision was confirmed to work, or that it was not exercised.

## 12. Carried forward

| # | Item | To | Blocker |
|---|---|---|---|
| 12.1 | Docker build and push to ECR with git-SHA tags | Thread 03 | `news-fetch-scheduler`'s Dockerfile installs `py_commons_per` from a local package index unreachable from a CI runner |
| 12.2 | `py_commons_per` installed from its public GitHub URL rather than the local index | Thread 03 | Removes a dependency on infrastructure only the development machine has |
| 12.3 | Kafka client configuration made conditional — PLAINTEXT locally, TLS with IAM or SASL on MSK | Service rewrite threads | Deliberately excluded from Thread 00A to contain scope |
| 12.4 | AWS Budget alert **confirmed by email** | First real spend | Creating a budget does not send an alert. Marked provisionally complete |

Also noted: `news-fetch-scheduler`'s Dockerfile has a missing runtime `WORKDIR`, a
bare `uvicorn` with no `--host 0.0.0.0`, and an orphaned `PYTHONPATH` comment. All
Thread 03 work.

## 13. Week 10 cleanup list — new

Deliberate deferrals, recorded so they read as choices rather than oversights. Full
rationale in `docs/learning-log.md`.

- `aws-actions/configure-aws-credentials@v5` — resolves the Node 20 deprecation
- Pin `ubuntu-24.04` instead of `ubuntu-latest`
- SHA-pin `actions/checkout` instead of `@v6`
- Move the OIDC trust policy to `job_workflow_ref` scoping
- Add a `data.aws_caller_identity` precondition to Terraform
- Remote Terraform state backend
- `tflint` and `tfsec`/`trivy` in `cloud-infra`

## 14. Budget

| # | Item |
|---|---|
| 14.1 | AWS denied new-account credits — the account was matched against a prior certification account on payment instrument and identity, not email alone. The advertised $200 does not apply |
| 14.2 | Budget set at **$30/month** rather than $300 total. A $300 monthly budget would permit triple the program ceiling across three calendar months without alerting |
| 14.3 | Thresholds: 50% actual, 80% actual, 100% **forecasted**. No Budget Actions — an automated response to a situation not yet experienced risks worse outcomes than a $30 overage |
| 14.4 | Expect to raise the limit at the cloud deployment thread, when Fargate, MSK and Bedrock at eval volume begin |