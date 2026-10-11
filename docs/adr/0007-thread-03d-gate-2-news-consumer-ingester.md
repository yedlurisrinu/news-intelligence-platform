# 0007. Thread 03-D · Gate 2 — news-consumer-ingester

- **Date:** 10-Oct-2026
- **Deciders:** Srini Yedluri
- **Scope:** architectural decisions taken while making news-consumer-ingester consume from local Kafka and index into self-hosted Elasticsearch
- **Companions:** `gates/03D-G2-gate-record.md` · `learnings/03D-G2-learnings.md` · previous: `0006-thread-03d-gate-1-news-fetch-scheduler.md`

| ID | Decision | Status |
|---|---|---|
| 0007.1 | Local Elasticsearch security: security on, TLS off | Accepted |
| 0007.2 | Per-service, least-privilege API keys | Accepted |
| 0007.3 | Versioned index behind an alias, explicit lexical mapping | Accepted |
| 0007.4 | Consumer group and offset strategy | Accepted |
| 0007.5 | Confirms 0006.7: at-least-once delivery with an idempotent consumer | Accepted |
| 0007.6 | Elasticsearch 8.x client against a 9.x server | Accepted |
| 0007.7 | Single merged application config | Accepted (format migration deferred) |

---

## 0007.1 — Local Elasticsearch security: security on, TLS off

- **Status:** Accepted
- **Context source:** ledger T03-S964, S1104, S1108

### Context
The services authenticate to Elasticsearch with `ELASTIC_URL` + `ELASTIC_API_KEY`. The local node is `elasticsearch:9.2.3`, single-node. TLS on the HTTP layer would mean distributing a CA certificate to every client (ingester, news app, Kibana).

### Decision
- `xpack.security.enabled=true`: authentication on, so the API-key code path stays unchanged.
- HTTP and transport TLS off; `discovery.type=single-node`.
- Services and Kibana use `http://elasticsearch:9200`.

### Consequences
- No certificates to distribute. Verified: `http://localhost:9200` → 401 (auth required); `https://` → no TLS listener.
- The API key service does not depend on HTTP TLS: `xpack.security.authc.api_key.enabled` "Defaults to true". The **token** service does default off when HTTP TLS is off, and is not used.
- Credentials travel unencrypted on the Docker network. This is acceptable for local development only; cloud deployment needs TLS.

### Alternatives considered
- Security fully off: quicker, but it would need a dummy key or a separate code branch.
- HTTPS with a self-signed CA.

---

## 0007.2 — Per-service, least-privilege API keys

- **Status:** Accepted
- **Context source:** ledger T03-S1105–S1106, S1110, S1112

### Context
Each service needs its own Elasticsearch credential. A shared superuser credential would let any service write or delete anything.

### Decision
- Keys are created with `POST /_security/api_key` as `elastic`, scoped by `role_descriptors` on `news_articles*` (which covers the alias and the versioned indices):
  - `news-consumer-ingester-local`: `write`, `view_index_metadata`
  - `news-search-summary-app-local`: `read`, `view_index_metadata`
- The response's **`encoded`** value (base64 of `id:api_key`) is stored in Vault under each service's existing `ELASTIC_API_KEY` key, with `vault kv patch`.
- Keys don't expire (the default), for local use.

### Consequences
- A leaked app key cannot write, and each key can be revoked on its own.
- elasticsearch-py sends a string `api_key` unchanged as `Authorization: ApiKey <value>`; only a `(id, secret)` tuple gets encoded. So the stored value must already be the `encoded` form.
- The `elastic` password stays in Vault and is used only to administer keys.

### Alternatives considered
- One shared read+write key.
- Basic auth with the `elastic` user.

---

## 0007.3 — Versioned index behind an alias, explicit lexical mapping

- **Status:** Accepted
- **Context source:** ledger T03-S1137–S1144; contradiction C-6 (resolved)

### Context
The Cloud index used `content.semantic` (`semantic_text`, served by a trial-tier inference endpoint). Thread 03-D scope is **lexical search only**. Future retrieval threads will change the mapping, and an alias cannot share a name with an existing index.

### Decision
- The concrete index is `news_articles_v2`, created **manually with an explicit mapping**: `dynamic: strict`; `title` and `content` as `text` with the `english` analyzer; `article_id` and `source` as `keyword`; `link` as `keyword` with `index: false`; `published_at` and `fetched_at` as `date`; **no `content.semantic`**.
- Alias `news_articles` → `news_articles_v2`. Every service addresses the alias.
- Migration path used: create v2 with the mapping → `_reindex` from the first-time concrete index → delete the old index → add the alias.

### Consequences
- Future mapping changes become a zero-downtime alias swap: build v3, reindex, then move the alias in one `_aliases` call.
- `_reindex` copies documents only, not mappings or settings, so every new version must be created with its mapping first.
- Verified on v2: `dynamic: strict`, `english` ×2, `index: false`.
- Index creation stays manual; automating it is parked.

### Alternatives considered
- A concrete index with no alias (the first-time state; would need a reindex-and-alias migration later anyway).
- Index creation at ingester startup (couples schema changes to the write path).

---

## 0007.4 — Consumer group and offset strategy

- **Status:** Accepted
- **Context source:** ledger T03-S1102, S1123, S1126–S1129

### Context
Committed offsets live on the broker, keyed by `group.id`, and are shared by every consumer using that group, whether it runs in Docker or locally. `auto.offset.reset` only applies when a group has no committed offset.

### Decision
- Group `news_ai_summary_group` with `auto.offset.reset=earliest`, `enable.auto.commit=false`.
- The offset is committed **only after** a successful bulk index.
- To re-consume on purpose, stop every member, then either use `kafka-consumer-groups.sh --reset-offsets --to-earliest --execute` or start a never-used group name.

### Consequences
- A crash between indexing and committing leads to re-delivery, not loss. The log order confirms it (index `:53,026` → commit `:53,035`).
- Changing `auto.offset.reset` has no effect on a group that has already committed offsets.

### Alternatives considered
- Auto-commit (risks losing a batch that was committed but not indexed).
- `latest` reset (skips anything published before the first start).

---

## 0007.5 — Confirms 0006.7: at-least-once delivery with an idempotent consumer

- **Status:** Accepted, moving 0006.7 from Proposed to Accepted
- **Context source:** ledger T03-S1125, S1129–S1130

### Context
0006.7 proposed treating the pipeline as at-least-once, with the ingester made idempotent through `_id` = `article_id`.

### Decision
0006.7 is accepted. The ingester uses the bulk **`index`** action with an explicit `_id` equal to the producer's MD5 `article_id`.

### Consequences
- Verified: one document's `_id` equals its `article_id` (`5122952f7afb3b941bf4ae01108bb4ea`). A re-consume of all 104 messages under a fresh group left `_count` at 104, and no index operation failed.
- `index` upserts by `_id` (the version goes up, the result is `updated`). The `create` action would instead fail with 409 on duplicates.

### Alternatives considered
- See 0006.7.

---

## 0007.6 — Elasticsearch 8.x client against a 9.x server

- **Status:** Accepted
- **Context source:** ledger T03-S1000–S1002, S1132

### Context
The services pin an 8.x Python client; the local server is 9.2.3. Elastic documents the 8.x Python client as compatible with 9.x servers through a compatibility mode that is always on in the Python client.

### Decision
Keep the client and the server as they are. Upgrade the client only if a break is observed.

### Consequences
- Verified for this workload: bulk index and search run against 9.2.3 with no errors or deprecation warnings (A5).
- Compatibility does not imply feature parity; 9.x-only features would need a 9.x client.

### Alternatives considered
- Upgrade the client to 9.x now.
- Downgrade the server to 8.x.

---

## 0007.7 — Single merged application config

- **Status:** Accepted (format migration to YAML deferred)
- **Context source:** ledger T03-S1127–S1128, S1131–S1135, S1148

### Context
The ingester loaded Kafka and Elasticsearch settings through separate config modules. `INGESTER_BATCH_SIZE` was defined but never passed through, so the code used its own default.

### Decision
- All settings are merged into `src/config/application_config.py`, which reads the project config file (TOML for now) and exposes `get_config(key)`.
- The separate elastic and confluent config loaders are removed.
- `INGESTER_BATCH_SIZE` is now wired from config.

### Consequences
- One source of configuration. Tests must patch `get_config` **where it is looked up** (`ingester.elastic_ingester.get_config`), not where it is defined.
- Boolean values currently load as strings (`'True'`/`'False'`). librdkafka parses them, but Python treats `'False'` as truthy for the ES client kwargs.
- Deferred to the end of Thread 03-D:
  - migrate to YAML (PyYAML `safe_load`) with real booleans;
  - unit tests for `application_config.py` (43% coverage);
  - more coverage for `elastic_setup.py`.

### Alternatives considered
- Keep per-subsystem config modules.
- Migrate to YAML within Gate 2.