# Learning log

Mechanisms, gotchas, and deferrals accumulated during the bridge program,
sectioned by thread.

Decisions live in [`adr/`](./adr/). Changes against the original plan live in
[`deviations/`](./deviations/). Outstanding deferrals and unresolved questions
live in [`open-items.md`](./open-items.md). This document holds what none of
those contain: how things work and what cost time.

**This file is append-only.** Each thread adds a `# Thread NN` section at the
bottom and rows to the index. Nothing above is rewritten.

---

## Index by topic

Mechanisms do not respect thread boundaries. Use this to find something when you
remember the subject but not when you met it.

| Topic | Section |
|---|---|
| Advertised listeners, client-side routing | [00A](#kafka-advertised-listeners) |
| AWS credential chain | [00](#the-aws-credential-chain) |
| CUDA compute capability, cubin vs PTX | [00](#cuda-compute-capability) |
| Distroless images and healthchecks | [00](#distroless-images-cannot-have-docker-healthchecks) |
| Exit status, `$?`, pipelines | [00](#assorted-git-and-shell) |
| Git fetch vs pull, `--all`, `--prune` | [00](#assorted-git-and-shell) |
| GitHub Actions triggers and reusable workflows | [00](#github-actions) |
| gitleaks modes and limits | [00](#gitleaks-scanning) |
| Healthcheck cost and `start_period` | [00](#start_period-and-start_interval) |
| OIDC token flow, trust policy claims | [00](#github-oidc-to-aws) |
| OpenTelemetry span model | [00](#the-opentelemetry-span-model) |
| Public repos and remediation | [00](#public-repos-have-no-remediation-window) |
| Python packaging, src layout, facades | [00](#python-packaging) |
| Replication factor, ISR | [00A](#kafka-replication-factor-on-a-single-node) |
| Submodules and gitlinks | [00](#submodules-and-gitlinks) |
| Terraform `for_each`, import, state | [00](#terraform-mechanics) |
| Vacuous passes and surface signals | [00](#a-control-that-reports-clean-is-indistinguishable-from-one-that-is-not-running) |

---

# Thread 00 — Platform kit, repos, and guardrails

**12–14 Sep 2026.** Deviations: [`deviations/thread-00.md`](./deviations/thread-00.md)

## Mechanisms

### CUDA compute capability

A compute capability is a GPU's **SM version**, written `major.minor`, identifying
the hardware feature set. It is not the CUDA toolkit version, despite both looking
like version numbers — an RTX 5080 is compute capability 12.0 and needs toolkit
12.8 or newer to target it.

Architecture names do not determine compatibility. Ampere spans 8.0, 8.6 and 8.7
across different dies. Blackwell spans 10.0 (datacenter) and 12.0 (consumer), with
no 11.x at all.

A compiled binary carries **cubin** (machine code for one target) and optionally
**PTX** (forward-compatible virtual assembly the driver JIT-compiles). Binary
compatibility runs forward across minor revisions only: a cubin for X.y runs on
X.z where z ≥ y, never backwards and never across a major revision. PTX runs on
anything at or above its target, including higher majors.

`nvidia-smi --query-gpu=name,compute_cap --format=csv` is the authoritative answer.

Two consequences that recur:

- Container images are usually built for one compute capability. A default tag
  generally means datacenter Ampere 8.0.
- In WSL2 the Windows driver is projected into containers through
  `/usr/lib/wsl/lib`. Only `nvidia-container-toolkit` is installed inside the
  distro; installing a Linux driver there breaks passthrough.

The `nvidia-smi` version reported inside a container differs from the host driver
version. The binary comes from the image; the driver it queries is the host's.
Normal under WSL, not a fault.

**Applies later at:** the embedding bake-off thread, where TEI image selection
depends on it.

### The OpenTelemetry span model

A **span** is one named, timed operation. A **trace** is a DAG of spans describing
one end-to-end request.

The tree is assembled from two fields — trace ID and parent span ID. There is no
coordinator; each span records its parent locally and the backend matches IDs.
Across process boundaries the same two values travel in the `traceparent` header
per W3C Trace Context, which is what allows several services to produce one trace.

Three objects, with distinct lifetimes:

| Object | Role | Cardinality |
|---|---|---|
| `TracerProvider` | Holds resource attributes, processors, exporters, sampler | One per process |
| `Tracer` | Manufactures spans; named after the instrumented component | One per module |
| `Span` | The unit of work | Many |

The provider **manufactures** tracers rather than holding them. Tracers are cheap;
the provider is the stateful thing, because it owns the export pipeline.

**Resource attributes and span attributes answer different questions.** Resource
describes *who is emitting* — `service.name`, `service.version` — set once at
provider construction and copied onto every exported span. Span attributes describe
*what this operation did* — `gen_ai.system`, `gen_ai.request.model`,
`gen_ai.usage.input_tokens` — and vary per call.

The pipeline separates two concerns. A **SpanProcessor** decides when and whether
spans are handed onward: `SimpleSpanProcessor` exports synchronously on span end,
correct for tests and wrong for production; `BatchSpanProcessor` queues and exports
on a background thread. An **Exporter** decides where and in what format.

`OTLPSpanExporter` is a client that stays in the process. What travels is protobuf
bytes over gRPC — an `ExportTraceServiceRequest`. Application to collector and
collector to backend are two independent hops, each with its own exporter.

Three behaviours worth knowing:

- `set_tracer_provider` **silently ignores** a second call. Installing twice does
  not error; it fails invisibly. Adding a processor twice to the same provider
  duplicates every span.
- `get_tracer` is safe before configuration — the SDK returns a no-op tracer, so a
  library can instrument itself unconditionally.
- `start_as_current_span` sets ambient context so nested spans become children.
  `start_span` does not, and silently produces orphans.

See [ADR-0002](./adr/0002-local-trace-backend.md) and
[deviations §5](./deviations/thread-00.md).

### GitHub OIDC to AWS

A workflow requests a JWT from GitHub's OIDC provider, signed with GitHub's private
key and carrying claims about the run. The workflow presents it to
`sts:AssumeRoleWithWebIdentity`. AWS fetches GitHub's public keys, verifies the
signature, checks the claims against the role's trust policy, and returns temporary
credentials.

Nothing is stored. There is no credential to steal.

`permissions: id-token: write` is what allows a job to request the token at all.
It is off by default, and its absence fails with a message about the token endpoint
rather than about permissions.

Two undocumented findings under Gotchas below.

See [ADR-0003](./adr/0003-infrastructure-as-code.md) and
[deviations §6](./deviations/thread-00.md).

### The AWS credential chain

The Terraform AWS provider, the CLI, and every AWS SDK resolve credentials through
the same ordered chain, first match winning:

1. Static credentials in the provider block
2. Environment variables
3. Shared files — `~/.aws/credentials`, `~/.aws/config`
4. Container credentials (ECS task role)
5. Instance metadata (EC2 instance profile)

This is why Terraform finds an account without being configured with one — an
`aws login` session writes a profile that step 3 picks up.

**The provider is silent about which account it resolved.** A stray environment
variable plans against the wrong account with no confirmation prompt. The guard is
a `data.aws_caller_identity` precondition comparing against an expected account ID.

### Terraform mechanics

**`for_each` over `count`.** `for_each` keys resources by name, so removing an item
from the middle of a list leaves the others untouched. `count` renumbers, causing
Terraform to destroy and recreate resources that did not change.

**Derived resource references create dependencies and prevent drift.** An IAM policy
whose resource list is `[for r in aws_ecr_repository.this : r.arn]` follows the ECR
resource map automatically. Hardcoded ARNs would require remembering, and forgetting
produces an access-denied error against a repository that plainly exists.

**Computed attributes do not force a diff when absent from config.** An imported ECR
repository showed `encryption_configuration` with no change despite never being
declared.

**`import` blocks** (Terraform 1.5+) adopt existing resources and are reviewable in
a plan before they run. The `id` format varies by resource type. Delete the block
once consumed, or it is re-evaluated on every plan.

**Brackets are a list constructor, not a list marker.** `[var.x]` builds a
one-element list; correct when `var.x` is a string, wrong when it is already a list.
The reliable question is what type the variable holds versus what the argument wants.

**What is committed:** `*.tf` and `.terraform.lock.hcl`. **Never:** `*.tfstate`
(may contain secrets in plaintext), `.terraform/`, `*.tfvars`, `*.tfplan`.

### gitleaks scanning

Two modes answering different questions: `gitleaks git` walks commit history;
`gitleaks protect --staged` checks an uncommitted diff. A pre-commit hook on a repo
with years of history says nothing about that history.

`--log-opts` is a **passthrough to the underlying `git log` call**, not a logging
flag. Without `--log-opts="--all"` the scan covers only history reachable from HEAD,
so a secret on a non-default branch is invisible.

In CI, `fetch-depth: 0` is required. The default checkout is shallow and a history
scan against it passes having examined one commit.

**Validity check:** scanned commits should equal total commits minus patchless ones.
A commit is patchless — and invisible to gitleaks — when it produces no textual
diff: an empty merge, a gitlink-only commit (mode `160000`, submodule pointer), or
an all-binary commit. A gap needs an explanation; a gap is not evidence of a
coverage gap.

**Three limits of a clean result:**

1. **Text only.** Binary files are never scanned. Archive traversal is off unless
   `--max-archive-depth` is set.
2. **Credentials only.** GCP project IDs, account emails and filesystem paths match
   no rule.
3. **Correlated patterns are conditional.** A bare AWS key ID does not fire without
   a paired secret access key.

`gitleaks dir` scans the filesystem and would read a real `.env`. `gitleaks git`
reads commit history and ignores the working tree.

See [deviations §6](./deviations/thread-00.md).

### Submodules and gitlinks

A submodule is a pointer: the parent stores a URL and a commit SHA as a **gitlink**,
file mode `160000`. Such commits carry no blob content and produce no patch.

`M` against a submodule path in `git status` means the *pointer* is dirty — the
submodule's checked-out commit differs from the SHA the parent records — not that
the parent's files changed.

`git clone` does not populate submodules without `--recurse-submodules`.

Converting a submodule to tracked content: `git rm --cached <path>`,
`rm -rf <path>/.git`, `rm -rf .git/modules/<path>`, remove the `.gitmodules`
section, then `git add`. Files on disk are untouched. A correct result shows
`D <path>` paired with `A <path>/<file>` entries.

### Assorted Git and shell

**`git fetch` is always safe.** It writes only to `refs/remotes/`, so it cannot
conflict or lose work. `git pull` is `fetch` + `merge` and can do both. Prefer
`fetch`, inspect with `git log HEAD..origin/main`, then decide.

`pull.ff only` is worth setting globally: silent when there are no local commits,
loud when histories diverge.

**`--all` means different things in different commands.** `git fetch --all` means
all *remotes*. `git log --all` means all *refs*.

**`--prune` performs the pruning**, deleting remote-tracking refs whose branches no
longer exist upstream.

**`$?` holds the exit status of the most recent foreground pipeline.** It is a shell
parameter, not an environment variable, and it is destroyed by the next command —
capture with `rc=$?` if needed later. In `a | b | c` it reports `c` only;
`PIPESTATUS` exposes the chain and `set -o pipefail` reports the first failure.
Non-zero does not mean broken: `grep` returns 1 for no match, gitleaks returns 1 for
findings.

**Editing a merged PR's title on GitHub does not change the merge commit message.**
They decouple once merged.

**A plain `git push` does not push tags.**

**`feat/` and similar branch prefixes** come from Conventional Commits and mean
nothing to Git. The slash creates a real namespace under `refs/heads/`, which is
what lets GitHub and IDEs group branches — and why a branch named `feat` cannot
coexist with `feat/anything`.

### GitHub Actions

**`workflow_call`** makes a workflow reusable; it never runs on its own.
**`workflow_dispatch`** is the manual trigger. Neither has anything to do with
running a test locally.

A dispatchable workflow must exist on the **default branch** before the Run button
appears, with no error explaining its absence.

Inputs are declared twice in a caller/reusable pair — the two blocks do not share
definitions. `type: choice` is valid only in `workflow_dispatch`; `type: number`
only in `workflow_call`. Mismatched types fail at parse time, so strings on both
sides is the safe shape.

**Runners are ephemeral per job.** A clean VM, destroyed on completion, sharing
nothing with other jobs by default. This is a security property, and it is why
secrets must be injected per run rather than installed once.

**Centralise when the shape recurs, not when the content does.** Seven repositories
running an identical secret scan justified a shared workflow. Eval suites differ in
content but share a runner shape, which justified it equally.

### Python packaging

**src layout** makes it impossible to import the package from the working directory
instead of the installed one, which is the usual cause of works-locally-fails-in-CI.

Re-exporting from `__init__.py` is a **facade**: it separates the public contract
from the file layout, so internals can be reorganised without touching consumers.
Worth it for a library, unnecessary for an application. `__all__` also signals to
linters that an unused import is deliberate re-export.

`uv tool install` is for standalone CLI tools — isolated environment, binary linked
into `~/.local/bin`. Distinct from project dependencies.

`pre-commit init-templatedir` plus `init.templateDir` installs hooks automatically
on every future `git init` or `git clone`.

**A global that resists testing is a design signal, not a testing inconvenience.**
`configure_tracing` both built a provider and installed it globally; splitting
`build_provider` out made it testable and was the better design regardless.

## Gotchas

### A control that reports clean is indistinguishable from one that is not running

This appeared three times in different costumes, and once inverted.

**The allowlisted canary.** A gitleaks canary using `AKIAIOSFODNN7EXAMPLE` returned
clean. Gitleaks allowlists AWS's documented example values so it does not fire on
every README. The tool matched, then suppressed — output identical to finding
nothing. A canary must use a value that is *not* a published example.

**The shallow-checkout scan.** A CI history scan against a default checkout examines
one commit and reports success.

**The missing command.** `jq` absent from a pipeline yields empty output rather than
a loud failure.

**Inverted: the delivered-but-invisible trace.** A span reached Jaeger correctly but
the UI showed nothing at any date range. Its hardcoded timestamps placed it a year
in the past. The API confirmed delivery while the interface showed nothing.

**The discipline:** make the tool fail on purpose before trusting it to pass, and
reconcile a number you can predict against a number the tool reports. When something
disagrees, check the layer that cannot lie — the API, the count, the exit code —
rather than the rendering.

### Published guidance ages, and the most-copied parts age worst

Four instances in one day:

- **The OIDC thumbprint.** Since July 2023 AWS validates GitHub's OIDC certificate
  against its own trusted CA library. `thumbprint_list` is optional and unused for
  GitHub, yet most guides still include one — and two different obsolete values
  circulate, plus a workaround of forty `f` characters.
- **`apt` and the AWS CLI.** The widely-repeated claim that Ubuntu ships CLI v1 is
  no longer true; 24.04 packages v2.
- **Static access keys.** `aws login`, shipped in CLI 2.32.0 (November 2025),
  authenticates through an existing browser session and issues temporary credentials.
  AWS lists it as the top recommended CLI authentication method, above Identity
  Center and above IAM user keys.
- **The `sub` claim format.** Below.

### GitHub's OIDC `sub` claim contains numeric IDs

Every published example assumes `repo:<owner>/<repo>:ref:refs/heads/main`. The
actual claim is:

```
repo:<owner>@<owner_id>/<repo>@<repo_id>:ref:refs/heads/main
```

An exact-match condition built from the documented format fails. The `repository`
claim is clean; `sub` is not.

Found by decoding the JWT payload inside a workflow run. That is the only reliable
way to see it.

### IAM requires a trust policy to constrain `sub` or `job_workflow_ref`

A trust policy conditioned only on `repository` and `ref` — both clean and readable
— is rejected with `MalformedPolicyDocument`. AWS enforces that one of those two
claims is constrained, specifically to prevent trusting every workflow in every
repository.

The working shape is `StringLike` on `sub` with wildcards covering only the numeric
IDs, alongside exact-match conditions on `repository`, `ref` and `aud`. The
wildcards sit where the IDs go; every human-readable component stays pinned, and is
pinned redundantly by the other conditions.

### Distroless images cannot have Docker healthchecks

The OTel Collector image has no shell, no `curl`, no `wget` — `docker run
--entrypoint sh` fails with "executable file not found". A `CMD` healthcheck is
impossible.

This is deliberate: no shell means no shell to exploit. The cost is exactly this,
and it means nothing in Compose can wait on the container's readiness.

Kubernetes sidesteps it with `httpGet` probes performed by the kubelet from outside
the container. Plain Compose has no equivalent.

### `start_period` and `start_interval`

`start_period` is the window during which failing checks do not count toward
`retries`. Too short and a slow-starting container flaps to unhealthy before it has
a chance — TEI needs 90s on a cold model cache, and larger models may need more.

`start_interval` gives a faster probe cadence during the start period, then reverts
to `interval`. Verified supported in the current Compose schema.

**A healthcheck's cost is a standing tax, not a one-off.** See
[00A](#kafka-healthchecks-are-expensive) for the case where this matters.

### Public repos have no remediation window

A leaked secret in a public repository cannot be excised by history rewrite —
anyone may already hold it. Rotation is the only remedy. This is why a full-history
audit belongs at the start rather than before a planned visibility flip.

Making a repository private afterwards does not un-expose what was public.
Archiving keeps it public and browsable, so **scan before archiving** — an archived
repository is read-only and cannot accept a rewrite without unarchiving.

Public → private permanently deletes stars and watchers.

---

# Thread 00A — Local Kafka broker

**14 Sep 2026.** Deviations: [`deviations/thread-00.md`](./deviations/thread-00.md) §9

## Mechanisms

### Kafka advertised listeners

Kafka connections have two phases, and they use different addresses.

**Bootstrap** connects to `bootstrap.servers` and asks one question: describe the
cluster. The broker replies with metadata naming every broker and the address to
reach it at. The client then **reconnects to that address** for all real work,
because clients talk directly to partition leaders rather than through a proxy.

`listeners` is where the broker binds. `advertised.listeners` is the string in
that metadata reply. They differ whenever the broker's own view of its address is
not the client's view — behind Docker NAT, a load balancer, or a VPC boundary.

**Kafka never validates the advertised value.** A wrong one produces a successful
bootstrap followed by failures against an address the client never configured.

A listener advertises exactly one address, so two audiences require two listeners:

| Client location | Bootstraps at | Told to use |
|---|---|---|
| Container on the shared network | `kafka:9092` | `kafka:9092` |
| Host process | `localhost:29092` | `localhost:29092` |

There is no producer address and consumer address — producers and consumers use
identical connection configuration. The split is *where the client runs*.

Listener names are arbitrary labels pairing a bind entry with an advertised entry
and a security protocol.

This is not Docker-specific, and not Kafka-specific. **Any protocol that routes
clients to specific nodes must know what those nodes are called from the client's
perspective, and cannot infer it.** Redis Cluster has `cluster-announce-ip`,
Cassandra has `broadcast_address`, MongoDB replica sets carry hostnames in the
replica set config. Same problem, different names.

Managed services hide it, which is why it is easy to work with Kafka for years
without meeting it.

**Resurfaces at:** MSK, where VPC DNS creates the same problem — a client outside
the VPC receives broker addresses it cannot resolve.

See [ADR-0005](./adr/0005-kafka-broker.md).

### Kafka replication factor on a single node

Replicas must sit on distinct brokers — three copies on one broker protect against
nothing, so Kafka refuses rather than pretending.

Three internal-topic settings default to 3 and must be overridden to 1 on a single
node: `offsets.topic.replication.factor`, `transaction.state.log.replication.factor`,
`transaction.state.log.min.isr`.

**The failure is delayed and misleading.** The broker starts and reports healthy;
it fails when a consumer group first commits an offset, because `__consumer_offsets`
was never created.

`min.insync.replicas` sets how many replicas must be caught up for an `acks=all`
write to be accepted. RF=3 with `min.insync.replicas=2` is the standard durable
configuration — it tolerates one broker failing while still accepting writes.
Setting it equal to RF stops writes on any single failure.

A single node cannot exercise any of this. Leader election, ISR shrink, and
`acks=all` semantics under failure are unobservable locally, so producer behaviour
tuned here may differ on MSK.

### KRaft

Kafka 4.0 removed ZooKeeper entirely; KRaft is the only metadata mode. A local
broker is one container rather than two.

`process.roles = broker,controller` is combined mode — one process doing both jobs.
Correct for development, not for production, where controllers are separate.

The controller listener is internal to the quorum and never connected to by a
client.

## Gotchas

### Kafka healthchecks are expensive

`kafka-broker-api-versions.sh` is a genuine readiness probe — it performs a real
client handshake, so it distinguishes "port open" from "serving metadata". But it
launches a **fresh JVM on every invocation**. At a 15-second interval that is
roughly 5,760 JVM starts a day.

`interval: 5m` with `start_interval: 10s` and `start_period: 90s` keeps the
meaningful signal at a fraction of the cost. A 5-minute interval means a dead
broker stays marked healthy for up to five minutes, which is acceptable locally and
is what real monitoring exists for elsewhere.

Also note the timeout must accommodate JVM startup — a 5-second timeout marks a
healthy broker unhealthy.

### Kafka UI maintainership

Provectus paused active development of `kafka-ui` and the project moved to
`kafbat/kafka-ui`, maintained by contributors from the original project. Docs and
tutorials still reference the Provectus image.