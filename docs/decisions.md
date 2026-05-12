# Architecture Decision Records

> **This is an architecture showcase repository.** Source code is private. These decisions are extracted from the project's ADR directory (ADR-QC-001 through ADR-QC-004).

---

## ADR-QC-001: Why LangGraph for Orchestration

**Status:** Accepted (2026-03-21)

**Decision:** Use LangGraph `StateGraph` as the orchestration framework rather than a plain `asyncio` loop or task queue.

**Rationale:**
- LangGraph enforces a typed state contract (`TypedDict`) that requires all data to be declared before any node can write it. This eliminates a class of bugs where nodes read stale or undeclared state.
- The conditional edge system (`route_after_select_well`, etc.) makes loop control an explicit, testable part of the graph definition rather than buried control flow inside node functions.
- Graph topology (nodes, edges, conditional edges) is defined once in `graph.py` and tested independently of node logic.

**Tradeoff:** LangGraph state must be JSON-serializable. This forced the design decision to hold live resources (API client, file handles, rule engine) on the graph class instance rather than in state, using closure injection to make them available inside nodes.

**Alternative rejected:** Plain `asyncio` loop. Works at small scale but does not enforce a state contract, making it harder to reason about data flow across multiple node iterations.

---

## ADR-QC-002: Why API Adapter Pattern (Option A) Over Direct API Consumption (Option B)

**Status:** Accepted (2026-04-07)

**Decision:** Migrate from browser extraction to API calls by adding a pure translation layer (`api_adapter.py`) that reshapes API JSON into the existing `extracted_data` dict format, rather than rewriting the 29 evaluation functions to consume API JSON directly.

**Rationale:**
- The 29 evaluation functions and 600+ tests represent validated, production-verified business logic. Rewriting them would require full re-validation against live data -- high risk with no correctness benefit.
- The adapter functions have a clear failure boundary: they either produce the correct dict shape or they do not. Failures surface as `INCONCLUSIVE` results with audit log entries, not silent wrong answers.
- Option B (direct consumption) is the correct long-term design and is tracked as a deferred task. It is deferred until Option A is validated in production.

**Context:** The migration was triggered by the April 3 incident where browser session degradation after 45+ minutes of continuous operation caused 111 incorrect scores to be published. Session degradation was traced to browser memory state accumulation -- a fundamental failure mode, not a timing or selector bug. The API path removes this failure mode entirely.

**Alternative rejected:** Browser session reset between operators. Adds latency and does not address within-operator degradation on large operators.

---

## ADR-QC-003: Why Two-Wave Concurrent Execution

**Status:** Accepted (2026-04-10)

**Decision:** Execute checks in two waves using `asyncio.gather`. Wave 1 runs all independent checks concurrently; Wave 2 runs dependency-dependent checks after Wave 1 completes.

**Rationale:**
- Sequential execution at ~1m 47s per well (API baseline) was acceptable but left no headroom as the portfolio grew. Three API endpoints are known to hang for up to 55 seconds -- in sequential execution, any one of these blocks all subsequent checks.
- The two-wave model achieves maximum concurrency for the 25+ independent checks while guaranteeing that dependency conditions are evaluated against completed Wave 1 results.
- A semaphore caps simultaneous in-flight requests more accurately than a token bucket rate limiter. The goal is not to space requests apart but to cap concurrent requests.

**Key design elements:**
- Request coalescing via per-endpoint `asyncio.Lock`: shared endpoints are fetched once, regardless of how many concurrent checks need them.
- Per-well circuit breaker: consecutive and total timeout counters prevent a pathologically slow well from blocking the full run.
- Run-level circuit breaker: consecutive aborted wells trigger queue drain to halt the run entirely.

**Alternative rejected:** Fully concurrent execution without wave separation. Creates a race condition where Wave 2 checks could run before their Wave 1 dependencies write results, producing spurious `INCONCLUSIVE` values.

---

## ADR-QC-004: Why Supabase as Score-of-Record

**Status:** Accepted (2026-04-16)

**Decision:** Write per-well check results to Supabase (persistent database) after every well, unconditionally. The dashboard receives only an operator-level summary. Supabase is the authoritative record.

**Rationale:**
- The dashboard is not a database. It has no query API suitable for trend analysis, no row-level history, and no mechanism to detect score drift between runs without re-fetching and comparing every column manually.
- Per-well history requires a queryable store. Questions like "what was the survey score on wells drilled in Q1?" are unanswerable against flat JSON reports on disk at any scale.
- `--no-publish` governs the dashboard only. Suppressing the Supabase write would defeat the score-of-record purpose -- a dry run that produces no database record is not a useful dry run from a data integrity perspective.

**Alternative rejected:** Expand dashboard columns to hold per-check data. The dashboard GraphQL rate limits, column ID management overhead, and lack of historical row preservation made this approach increasingly difficult to maintain as the check set grew.

---

## ADR-QC-001 (Foundation): Startup Security Gate

**Status:** Accepted (2026-03-21)

**Decision:** A security gate runs as the first node in the graph, before any network activity. Six checks verify the security environment. Any failure raises `SecurityPolicyViolation` and exits immediately.

**Rationale:**
- The orchestration framework's tracing integration phones home to an external service by default. The network policy allows exactly two domains. The external tracing service is not one of them. A passive environment check is insufficient -- the gate must block execution, not warn.
- Credential leakage via logs is a known failure mode in automation tools. Defense in depth requires both preventing leakage at the source (security gate) and scrubbing it if it reaches the logger (log sanitizer).

**Checks in order:**
1. Tracing disabled (checked by env var value, not just presence)
2. No external tracing API key
3. Required credentials present and non-empty
4. `.env` exists on disk and is listed in `.gitignore`
5. Output directory listed in `.gitignore`
6. No prohibited telemetry env vars (Sentry, Datadog, etc.)

**Alternative rejected:** Passive warnings only. Warnings are ignored under time pressure.

---

## ADR-QC-001 (Foundation): CSV Input Manifest

**Status:** Accepted (2026-03-21, superseded by API discovery in v0.9.0)

**Decision:** (Historical) The agent's input was a CSV file provided before each run, listing wells to evaluate.

**Rationale (at time of decision):**
- An earlier scope version scraped the active well list from the platform. This coupled the agent to platform UI changes and made each run's scope implicit.
- A CSV manifest made each run explicit and auditable: the input is a durable artifact independent of platform state.

**Superseded by:** API-driven well discovery (v0.9.0). The platform's search API can return the same data with operator and status filters, making a manually maintained CSV a liability. The CSV parser module is retained as a utility but is no longer in the orchestrator's active code path.

---

## ADR-QC-001 (Foundation): Token Bucket Rate Limiter

**Status:** Accepted (2026-03-21, platform bucket superseded in v0.8.0)

**Decision:** A shared singleton rate limiter controls all outbound requests. Hard floors and ceilings are enforced in code and cannot be overridden by configuration.

**Rationale:**
- The O&G cloud platform is a production system shared with real users and real clients. Aggressive automation against a shared SaaS platform is a platform risk, not just a personal tool risk.
- Floors prevent accidental misconfiguration from causing platform harm. The singleton pattern ensures no code path bypasses the limiter by instantiating its own instance.
- The platform bucket (originally for browser page loads, then repurposed for API calls in v0.7.x) was removed in v0.8.0 when the concurrent semaphore replaced it as the primary API concurrency control. The Monday.com bucket remains active.

---

## ADR-QC-001 (Foundation): Versioned JSON Output Schema

**Status:** Accepted (2026-03-21)

**Decision:** Every run produces a JSON report conforming to a versioned schema. The schema is the data contract for downstream agents in the agentic network.

**Rationale:**
- The filesystem-as-message-bus pattern requires a stable, machine-readable contract. Downstream agents consume this output without human mediation.
- The `schema_version` field allows downstream consumers to detect breaking changes and fail loudly rather than silently misparse data.
- CSV was rejected as lossy for nested structures (per-module results per well) and not self-describing.
