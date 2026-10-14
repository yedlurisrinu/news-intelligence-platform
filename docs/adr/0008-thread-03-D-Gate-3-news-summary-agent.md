# 0008. Thread 03-D · Gate 3 — news-summary-agent

- **Date:** 11-Oct-2026 → 13-Oct-2026
- **Deciders:** Srini Yedluri
- **Scope:** architectural decisions taken while renaming the summary app to news-summary-agent, migrating it from LCEL to a hand-built LangGraph `StateGraph` workflow, and wiring it to self-hosted Elasticsearch (lexical), PostgresSaver and OpenAI
- **Companions:** `gates/03D-G3-gate-record.md` · `gates/03D-FG-gate-record.md` · `learnings/03D-G3-learnings.md` · previous: `0007-thread-03d-gate-2-news-consumer-ingester.md`

Legend: **D** = our discussion / a verified run · **W** = web, cited · **C** = model knowledge · **U** = unverified.

| ID | Decision | Status |
|---|---|---|
| 0008.1 | Rename the summary app to news-summary-agent | Accepted |
| 0008.2 | App Elasticsearch key: cluster `monitor` + index `read`, `view_index_metadata` (amends 0007.2) | Accepted |
| 0008.3 | Vault path rename `secret/DB_SERVICES` → `secret/CREDENTIALS` by expand–contract | Accepted (contract step pending) |
| 0008.4 | Hand-built `StateGraph` **workflow**, not a `create_agent` agent | Accepted |
| 0008.5 | One chat-model factory in `adapters/`, injected; prompt style | Accepted (prompt style: recommendation, adoption **U**) |
| 0008.6 | One class per node; dependencies via `__init__`, bound at build time | Accepted |
| 0008.7 | Connection scoping: FastAPI lifespan in its own module; graph compiled once | Accepted (interim — see Consequences) |
| 0008.8 | Checkpointer on a shared pool; `thread_id = str(uuid4())` per run | Accepted |
| 0008.9 | Lexical retrieval: query templates in YAML, filled in code | Accepted |
| 0008.10 | Result type at the API boundary; discriminated-union response models | Accepted |
| 0008.11 | Freeze library versions until the end of the programme; `mypy --strict` | Accepted |
| 0008.12 | 3a LCEL baseline skipped; 3b lexical results are the retrieval baseline | Accepted |

---

## 0008.1 — Rename the summary app to news-summary-agent

- **Status:** Accepted
- **Context source:** ledger pre-S1201 (compacted)

### Context
The repo carried "rag" and "search-summary" names from its LCEL era. Thread 03-D turns it into a LangGraph workflow, and the API key, Vault canary and records all needed a stable service name.

### Decision
Repo and service name: **news-summary-agent**. The ES key is `news-summary-agent-local-key`.

### Consequences
- The name says "agent" while the implementation is a workflow (0008.4). That is accepted as the product name; code-level names use "workflow" (`run_graph_workflow.py`).

---

## 0008.2 — App Elasticsearch key: cluster `monitor` + index `read`

- **Status:** Accepted · **Amends:** 0007.2 (app key `news-search-summary-app-local`)
- **Context source:** ledger pre-S1201 (compacted); S1258–S1259; contradiction C-7

### Decision
- New key `news-summary-agent-local-key`, created as `elastic` with `role_descriptors`:
  - cluster: `monitor` (allows `ping`/cluster info);
  - indices `news_articles*`: `read`, `view_index_metadata`.
- `encoded` value stored in Vault as `ELASTIC_API_KEY`.

### Consequences
- There is **no** bare cluster `read` privilege; `monitor` is the read-only cluster privilege (C-7).
- An API key's effective rights are the intersection of its role descriptors and its owner's; owner `elastic` → the descriptors are the only limit. **C**
- Verified in use: a stale index name (`news_index`) produced `403 security_exception … granted by the index privileges [read,all]`; the key was correct, the target index was not (S1259). **D**

---

## 0008.3 — Vault path rename by expand–contract

- **Status:** Accepted (expand done; contract pending)
- **Context source:** ledger pre-S1201 (compacted); S1281–S1283

### Context
`secret/DB_SERVICES` had become a misleading name for a path holding ES, Postgres and OpenAI credentials. Three services read it.

### Decision
- **Parallel change (expand–contract):** copy to `secret/CREDENTIALS` (same mount), move news-summary-agent first as the **canary**, then the scheduler and ingester, then delete `DB_SERVICES`.
- Copy (no native copy command): `vault kv get -format=json secret/DB_SERVICES | jq '.data.data' | vault kv put secret/CREDENTIALS -`.
- ACL policy paths must include the new path before any service switches.

### Consequences
- Both paths coexist for a while (Srini, 13-Oct); a rotation must update both until the contract step.
- Final delete is `vault kv metadata delete` (permanent, all versions), not `vault kv delete` (soft, latest version only).

---

## 0008.4 — Hand-built `StateGraph` workflow, not a `create_agent` agent

- **Status:** Accepted
- **Context source:** ledger pre-S1201 (compacted); S1237

### Context
The flow is fixed: classify intent → (off-topic exit) → search → summarise. Anthropic and LangGraph both distinguish **workflows** (predefined code paths) from **agents** (the model directs its own tool use).

### Decision
A hand-built `StateGraph` with explicit nodes and conditional routing — prompt chaining plus routing. `create_agent` is reserved for any future node that genuinely needs an open-ended tool loop.

### Consequences
- Control flow, retries and checkpoints are visible and testable per node.
- Vocabulary in code: "workflow", not "agent".

---

## 0008.5 — One chat-model factory, injected

- **Status:** Accepted (prompt style: recommendation; adoption **U**)
- **Context source:** S1201–S1206

### Decision
- `src/adapters/llm.py` builds the model once (`get_chat_model()`, `functools.lru_cache`), typed as `BaseChatModel`, and it is injected into the nodes that need it (0008.6).
- Recommended inside nodes: plain message lists (`SystemMessage`/`HumanMessage`) with system prompts as plain string constants; `with_structured_output(Model)` instead of `JsonOutputParser` for structured outputs. Whether every node adopted these is **U**.

### Consequences
- langchain-openai already shares one cached `httpx` pool across `ChatOpenAI` instances with the same base URL/timeout (verified, 1.6.2); the factory's value is one config point and testability, not connection reuse.
- `BaseChatModel` typing lets unit tests pass `FakeListChatModel`.
- `ChatPromptTemplate` gotcha avoided: literal `{`/`}` must be doubled, an unescaped JSON example raises `KeyError` (verified).

### Alternatives considered
`init_chat_model("openai:…")` (provider-agnostic factory); keeping `ChatPromptTemplate | llm | parser` chains inside nodes.

---

## 0008.6 — One class per node; dependencies bound at build time

- **Status:** Accepted
- **Context source:** S1207–S1214

### Decision
- Each node is its own class: dependencies through `__init__` (e.g. `Search(llm, es)`), the node body in `__call__(state)`.
- Registered with an explicit name: `g.add_node("search", Search(llm, es))`.
- Dependencies are fixed for the process, so they are bound when the graph is built; no `context_schema` / `Runtime` injection.

### Consequences
- Without an explicit name, LangGraph names a callable instance after its class (`"ClassifyIntentNode"`) and a `functools.partial` `"partial"` (verified).
- Instances are shared by every request: no per-request data on `self`; per-request data lives in state.

### Alternatives considered
`functools.partial(node_fn, llm=llm)`; closures; one shared nodes class; LangGraph `context_schema` + `Runtime` (per-invocation dependencies).

---

## 0008.7 — Connection scoping: FastAPI lifespan in its own module

- **Status:** Accepted (interim)
- **Context source:** S1217–S1247

### Decision
- `src/adapters/lifespan.py`: an `@asynccontextmanager` lifespan opens the Postgres pool and the ES client once per process, builds and compiles the graph once, and stores it on `app.state.graph`.
- Routes get the graph through a typed accessor (`get_graph(request) -> CompiledStateGraph`); `Depends(get_graph)` recommended so tests can use `dependency_overrides`.

### Consequences
- Verified behaviour (FastAPI 0.115.0 / Starlette 0.38.6): lifespan runs once per process (1 enter / 1 exit over 3 requests); each uvicorn worker runs its own (2 PIDs) → size the pool so workers × max_size < Postgres `max_connections`.
- Lifespan must be an **async** context manager even with sync code inside; a sync `@contextmanager` raises `TypeError`.
- It does not run under `TestClient(app)` without `with`, or for a mounted sub-app (verified) — the cause of the `KeyError: 'graph'` seen during wiring.
- **Accepted coupling:** the API layer still builds and holds the graph. Target design (follow-up): the workflow package owns composition (`open_news_workflow(settings)` yielding a narrow interface), with lifespan only scoping it; `build_graph(...)` stays public with `checkpointer=None` so a future parent can embed it as a subgraph (`input_schema`/`output_schema` as its contract).

### Alternatives considered
Opening the saver and ES per request (the previous code); a service wrapper built in the API (rejected by Srini: still couples the API to the graph); out-of-process graph service / `RemoteGraph`.

---

## 0008.8 — Checkpointer on a shared pool; per-run `thread_id`

- **Status:** Accepted
- **Context source:** S1236–S1242; S1283

### Decision
- `PostgresSaver(pool)` on a psycopg `ConnectionPool` (accepted type: `Connection | ConnectionPool`).
- `thread_id = str(uuid4())` per run.

### Consequences
- The previous `hash(query)` thread id was a defect: `str` hashing is randomised per process (two runs gave two values), and a reused thread accumulates reducer state across requests (`['hit','hit']`). Both verified.
- For the planned email-approval node (interrupt/resume): the thread id identifies one run and is returned to the client; a browser session (signed cookie) is the run's **owner**, checked on `/approve`, not the thread id. Deferred until the E2E flow is stable (Srini).
- A stable per-query key, if ever needed, is `hashlib.sha256(query.strip().casefold().encode())` — a cache key, not a thread id.

---

## 0008.9 — Lexical retrieval: query templates in YAML

- **Status:** Accepted
- **Context source:** S1250–S1265; Final Gate logs

### Decision
- Static query shape in YAML config (`safe_load`), dynamic parts filled in code on a `copy.deepcopy` of the template.
- Search query (as run): `bool.must` = `multi_match` on `title^2`, `content` with `minimum_should_match: "2<75%"`; `filter` = `range published_at gte now−7d` (computed in code); `sort` = `published_at desc`, then `_score`; `size: 5`; `_source: [article_id]`, then `_mget` for the full documents.
- Category terms from ClassifyIntent go into a boosted `should` on `title` (1.5) when present.

### Consequences
- With `published_at` sorted first, `_score` only breaks ties, so the category boost does not change order (observed: scores out of order). **D**
- `2<75%` stops a generic word ("news") from matching the whole index.
- The ClassifyIntent category check aggregates on `source.keyword`, but `source` is mapped as `keyword` (0007.3), so that sub-field does not exist and the aggregation returns no buckets; with empty categories the check matches all 1398 docs. Tracked as a finding.

### Alternatives considered
JSON file per query; ES Mustache search templates (stored form needs a cluster privilege the read-only key lacks).

---

## 0008.10 — Result type at the API boundary

- **Status:** Accepted
- **Context source:** ledger pre-S1201 (compacted)

### Decision
- The workflow run returns a result object with a `status` (ok/failed) and an error description instead of raising; one boundary catch at the seam.
- FastAPI response: `QuerySuccessResponse | QueryFailedResponse`, a discriminated union on `status` (`Annotated[Union[...], pydantic.Field(discriminator="status")]`), failure mapped to an HTTP status code.

### Consequences
- Off-topic and empty-articles outcomes are `status: ok` with an explanatory summary, not failures (observed in the UI).
- Client-facing error messages are sanitised; full detail goes to `logger.exception`.

---

## 0008.11 — Freeze versions until the end of the programme; `mypy --strict`

- **Status:** Accepted
- **Context source:** S1283

### Decision
- Pin `langchain==1.4.2` (was `>=1.4.2`), alongside the existing exact pins; no upgrades until the programme ends (version-drift item already parked).
- `uv run mypy src/ --strict` is the gate check: 28 files, no issues.

---

## 0008.12 — 3a LCEL baseline skipped; 3b is the baseline

- **Status:** Accepted
- **Context source:** S1250

### Decision
3a (lexical search on the old LCEL app) is not run; the LCEL code has since been removed. The lexical results of the 3b workflow are the baseline the Retrieval & Qdrant and Embedding threads will measure against. Qdrant stays out of Thread 03-D (09-Oct scope cut).