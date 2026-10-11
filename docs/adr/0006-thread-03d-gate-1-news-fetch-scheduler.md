# 0006. Thread 03-D · Gate 1 — news-fetch-scheduler

- **Date:** 10-Oct-2026
- **Deciders:** Srini Yedluri
- **Scope:** architectural decisions taken while making news-fetch-scheduler run against local Kafka in Docker
- **Companions:** `gates/03D-G1-gate-record.md` · `learnings/03D-G1-learnings.md`

| ID | Decision | Status |
|---|---|---|
| 0006.1 | Local Kafka broker for development | Accepted |
| 0006.2 | Shared Docker network topology | Accepted |
| 0006.3 | uv as the Python build manager | Accepted |
| 0006.4 | Retire the private package index | Accepted |
| 0006.5 | Container log persistence via bind mount | Accepted |
| 0006.6 | Phased mypy adoption | Accepted |
| 0006.7 | At-least-once delivery with an idempotent consumer | Accepted (confirmed in 0007.5) |

---

## 0006.1 — Local Kafka broker for development

- **Status:** Accepted
- **Context source:** Thread 03-D, Gate 1 (ledger T03-S1007, S1017–S1019, S1027, S1049, S1054)

### Context
The pipeline published to Confluent Cloud over `SASL_SSL`, with bootstrap servers and SASL credentials loaded from Vault. The end-to-end integration runs locally against `apache/kafka:4.0.0` (single-node KRaft), which has no credentials. The broker advertises two listeners: `PLAINTEXT://kafka:9092` (containers) and `PLAINTEXT_HOST://localhost:29092` (host). It also has `auto.create.topics.enable=true` with `num.partitions=1`.

### Decision
1. Services connect to `PLAINTEXT://kafka:9092` and run as containers on the shared network (see 0006.2).
2. Client config: `bootstrap.servers=kafka:9092` and `security.protocol=PLAINTEXT` from the properties file, with no `sasl.*` keys. Vault `CONFLUENT.*` values are **not** merged locally.
3. Topic `news_ai_pipeline_dev_t_0` keeps its existing name and is created **manually, before the first publish**, with 3 partitions and replication factor 1.
4. Messages stay keyed by the article hash: `produce(topic, value, key)`.

### Consequences
- The client protocol matches the broker's container listener. `SASL_PLAINTEXT` would fail: without `sasl.mechanisms` librdkafka defaults to `GSSAPI`, which is not built into 2.16.0.
- Not merging Vault values removes the risk of Vault's Cloud bootstrap silently overriding the local one. Stray SASL keys under `PLAINTEXT` would only produce a `CONFWARN`.
- Manual creation stops auto-create from producing a 1-partition topic.
- Keyed messages put all copies of one article on the same partition.
- The `settings.py` comment ("partitions: 6") still describes Confluent (contradiction C-4, open).
- The Cloud path is kept for the Terraform/AWS thread.

### Alternatives considered
- Run the service on the host via `localhost:29092`.
- Keep merging Vault values and tolerate the `CONFWARN`.
- Rely on broker auto-create.
- Connect over `SASL_PLAINTEXT`.

---

## 0006.2 — Shared Docker network topology

- **Status:** Accepted
- **Context source:** Thread 03-D, Gate 1 (ledger T03-S1021, S1023–S1024; contradiction C-3)

### Context
Infrastructure containers were spread across `infra-network`, `services_network` and `vault-network`. The integration brief assumed the broker was on `infra-network`, but it was on `services_network` (contradiction C-3). Containers resolve each other by service name only when they share a network.

### Decision
- All runtime infrastructure sits on **`services_network`**: kafka, postgres, elasticsearch, redis, mongodb, qdrant, tei and the OTel collector / Jaeger.
- Vault stays on **`vault-network`**.
- Each application service (scheduler, ingester, news-search-summary-app) joins `services_network` + `vault-network`, both declared `external: true`.
- In-container configuration uses service names, for example `kafka:9092` and `postgres`, never `localhost`.

### Consequences
- `infra-network` is no longer needed at runtime. `pypi-server` was its last user, and it is retired (see 0006.4).
- news-search-summary-app moves from `infra-network` to `services_network`.
- A misplaced `localhost` inside a container points at that container itself.

### Alternatives considered
- Per-service networks with multi-homing.
- Keep `infra-network` for application services.

---

## 0006.3 — uv as the Python build manager

- **Status:** Accepted
- **Context source:** Thread 03-D, Gate 1 (ledger T03-S1029–S1047, S1052, S1084–S1086)

### Context
news-rag-agent already used uv; the scheduler and ingester used pip with `requirements.txt` files. Test packages shipped inside the runtime image. One build manager across repos reduces drift.

### Decision
1. Each Python service uses uv: `pyproject.toml` + `uv.lock` + `.python-version` (3.12, matching `python:3.12-slim`). `requirements*.txt` files are removed.
2. Test and tooling packages go in `[dependency-groups] dev`, and images install with `uv sync --locked --no-dev`.
3. Docker uses a multi-stage build:
   - the uv binary is copied from a pinned `ghcr.io/astral-sh/uv` image;
   - `UV_COMPILE_BYTECODE=1`, `UV_LINK_MODE=copy` and `UV_PYTHON_DOWNLOADS=0` are set;
   - the builder's `/app/.venv` is copied into the runtime stage and put on `PATH`;
   - `.dockerignore` excludes `.venv`.
4. Projects stay unpackaged (no `[build-system]`). **`PYTHONPATH=src` is retained** as the import root.
5. Version validation for a newly migrated repo is satisfied by the same-day uv resolution plus the lockfile. The resolved versions are recorded as the stamp.

### Consequences
- The lockfile pins every resolved version. `>=` floors resolved to current releases (scheduler stamp: confluent-kafka 2.16.0, pydantic 2.14.0, uvicorn 0.54.0, …).
- uv installs the project itself only when `[build-system]` exists. Without one, `src/` is not importable unless `PYTHONPATH=src` is set; removing it causes `ModuleNotFoundError`.
- mypy must use the same root: `mypy_path = "src"` + `explicit_package_bases`.

### Alternatives considered
- Stay on pip with `mypy.ini`.
- Package each service (`[build-system]`, editable install) to drop `PYTHONPATH`. Parked; trigger: Terraform/AWS thread.

---

## 0006.4 — Retire the private package index

- **Status:** Accepted
- **Context source:** Thread 03-D, Gate 1 (ledger T03-S1025, S1033–S1036)

### Context
Logging setup and Vault config loading lived in a private wheel, `py-commons-per`, served by a local `pypi-server` on `infra-network`. Builds depended on that index being reachable. The Dockerfile installed it through a separate `requirements_locales.txt` step.

### Decision
Inline the shared code into each service (for example `src/config/logging_setup.py` and `src/config/vault_secrets.py` in the scheduler) and retire `pypi-server` and `py-commons-per`.

### Consequences
- Builds no longer need a private index or a specific build network.
- The code is duplicated across services; changes to logging or Vault loading must be repeated per repo.
- Dockerfile steps that copied `requirements_locales.txt` had to be removed; leaving them breaks the build, because `COPY` of a missing file aborts it.

### Alternatives considered
- Keep the private wheel, served from the host-published port.
- Publish the shared code as a uv workspace member.

---

## 0006.5 — Container log persistence via bind mount

- **Status:** Accepted
- **Context source:** Thread 03-D, Gate 1 (ledger T03-S1067–S1068)

### Context
Services write file logs to `/app/logs` inside the container's writable layer. That layer is lost when the container is removed or recreated. Containers run as a non-root `appuser`.

### Decision
Bind-mount `./logs:/app/logs` in each service's compose file. Create the host directory before the first run, and keep `logs/` in `.gitignore` and `.dockerignore`.

### Consequences
- Logs survive container recreation and can be read directly on the host.
- The bind mount masks the image's `chown` of `/app/logs`. A host directory created by Docker is root-owned and unwritable by `appuser`, so the host directory's owner must match `appuser`'s UID.
- Applying the change needs `docker compose up -d` (a recreate), not `docker restart`.

### Alternatives considered
- A named volume: Docker-managed ownership, but less convenient to browse.
- stdout-only logging via `docker logs`.

---

## 0006.6 — Phased mypy adoption

- **Status:** Accepted
- **Context source:** Thread 03-D, Gate 1 (ledger T03-S1051, S1072, S1079–S1089)

### Context
The type checker standard for the programme is **mypy 2.4.0**: current on PyPI, and already used in the learning lab. The scheduler has no type hints, and `--strict` reports 41 errors, mostly missing annotations plus one real defect (`postgres_setup.py:18/22` reuses a file-handle variable as a dict). news-rag-agent is about to be migrated to a LangGraph graph and should not need typing rework afterwards.

### Decision
1. Make each service Docker-deployable first, then type-check it.
2. **news-rag-agent** adopts mypy inside Thread 03-D (Gate 3), before the graph migration.
3. **Scheduler and ingester** adoption is parked until the end-to-end model works (trigger: after the Final Gate). The scheduler's groundwork stays in place: mypy 2.4.0 and `types-hvac` in the dev group, `mypy_path = "src"` + `explicit_package_bases`.

### Consequences
- The upstream services run untyped through the integration; type defects there surface only at runtime until adoption.
- When adoption resumes, fix the `postgres_setup.py` defect first.
- Without `__init__.py` files, a bare `mypy src/` fails with "Source file found twice"; the package-base settings above are required.

### Alternatives considered
- Type every service before integration.
- Keep `ty` (it produced false positives in the programme's libraries; see the type-checker migration item).

---

## 0006.7 — At-least-once delivery with an idempotent consumer

- **Status:** Accepted — confirmed by 0007.5 (re-consume test, 10-Oct-2026)
- **Context source:** Thread 03-D, Gate 1 (ledger T03-S1062–S1063)

### Context
The scheduler produces to Kafka **before** it writes the dedup record. During Gate 1, runs whose dedup INSERT failed had already published. At the first clean run the topic held at least 172 messages against 98 dedup IDs, so duplicates exist on the topic. Producer-side dedup prevents most republishing (second run: 116 cached, 102 fetched, 1 published), but cannot guarantee it.

### Decision
Treat the pipeline as **at-least-once**. The ingester must be **idempotent**: the Elasticsearch document `_id` is the *article_id*, so a duplicate message overwrites the same document instead of creating a new one.

### Consequences
- Duplicates on the topic are harmless downstream.
- Gate 2 asserts that one document's `_id` equals its *article_id*, and that re-consuming produces no new documents.
- Reordering the scheduler to write dedup before producing would trade duplicates for possible message loss on a failed produce.

### Alternatives considered
- Reorder to dedup-then-produce (at-most-once risk).
- Kafka transactions / exactly-once producer semantics. That is heavier, and it doesn't cover the external Postgres write.