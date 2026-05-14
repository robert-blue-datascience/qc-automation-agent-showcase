# Architecture Decision Records

> **This is an architecture showcase repository.** Source code is private. No client data or proprietary business logic appears in this document.

---

## Why LangGraph for Orchestration

**Decision:** Use LangGraph `StateGraph` as the orchestration framework rather than a plain `asyncio` loop or task queue.

**Rationale:**
- LangGraph enforces a typed state contract (TypedDict) that requires all data to be declared before any node can write it. This eliminates a class of bugs where nodes read stale or undeclared state.
- The conditional edge system makes loop control an explicit, testable part of the graph definition rather than buried control flow.
- Graph topology (nodes, edges, conditional edges) is defined once and tested independently of node logic.

**Tradeoff:** LangGraph state must be JSON-serializable. This forced live resources (API client, file handles, rule engine) to be held on the graph class instance rather than in state, using closure injection.

---

## Why API Adapter Pattern Over Direct API Consumption

**Decision:** Migrate from browser extraction to API calls by adding a pure translation layer that reshapes API JSON into the existing dict format, rather than rewriting the evaluation functions to consume API JSON directly.

**Rationale:**
- The evaluation functions and their hundreds of tests represent validated, production-verified logic. Rewriting them would require full re-validation with no correctness benefit.
- The adapter functions have a clear failure boundary: they either produce the correct dict shape or they do not. Failures surface as inconclusive results with audit log entries, not silent wrong answers.
- Direct API consumption is the correct long-term design and is tracked as a deferred task.

**Context:** The migration was triggered by a production incident where browser session degradation after extended operation caused over a hundred incorrect scores to be published. Session degradation was traced to browser memory state accumulation, a fundamental failure mode (not a timing or selector bug). The API path removes this failure mode entirely.

---

## Why Two-Wave Concurrent Execution

**Decision:** Execute checks in two waves using `asyncio.gather`. Wave 1 runs all independent checks concurrently; Wave 2 runs dependency-dependent checks after Wave 1 completes.

**Rationale:**
- Sequential execution was acceptable but left no headroom as the portfolio grew. Some API endpoints are known to hang for extended periods; in sequential execution, any one of these blocks all subsequent checks.
- The two-wave model achieves maximum concurrency for the 25+ independent checks while guaranteeing that dependency conditions are evaluated against completed Wave 1 results.
- A semaphore caps simultaneous in-flight requests more accurately than a token bucket rate limiter.

**Key design elements:**
- Request coalescing via per-endpoint lock: shared endpoints are fetched once regardless of how many concurrent checks need them.
- Per-well circuit breaker: consecutive and total timeout counters prevent a slow well from blocking the full run.
- Run-level circuit breaker: consecutive aborted wells trigger queue drain.

---

## Why a Persistent Database as Score-of-Record

**Decision:** Write per-well check results to a persistent database after every well, unconditionally. The dashboard receives only an operator-level summary.

**Rationale:**
- The dashboard is not a database. It has no query API suitable for trend analysis, no row-level history, and no mechanism to detect score drift between runs.
- Per-well history requires a queryable store. Questions like "what was the score trend for wells drilled in Q1?" are unanswerable against flat JSON reports on disk.
- `--no-publish` governs the dashboard only. Suppressing database writes would defeat the score-of-record purpose.

---

## Startup Security Gate

**Decision:** A security gate runs as the first node in the graph, before any network activity. Six checks verify the security environment. Any failure exits immediately.

**Rationale:**
- The orchestration framework's tracing integration phones home to an external service by default. The network policy allows only declared domains. A passive environment check is insufficient; the gate must block execution, not warn.
- Credential leakage via logs is a known failure mode in automation tools. Defense in depth requires both preventing leakage at the source (security gate) and scrubbing it if it reaches the logger (log sanitizer).
- Under time pressure, warnings are ignored. Hard gates are not.

---

## Token Bucket Rate Limiter with Hard Floors

**Decision:** A shared singleton rate limiter controls all outbound requests. Hard floors and ceilings are enforced in code and cannot be overridden by configuration.

**Rationale:**
- The cloud platform being automated is a production system shared with real users. Aggressive automation against a shared platform is a platform risk, not just a personal tool risk.
- Floors prevent accidental misconfiguration. The singleton pattern ensures no code path bypasses the limiter by instantiating its own instance.
- Configuration files get edited. Environment variables get changed under time pressure. Hard floors in code cannot be accidentally bypassed.

---

## Versioned JSON Output Schema

**Decision:** Every run produces a JSON report conforming to a versioned schema. The schema is the data contract for downstream agents in the agentic network.

**Rationale:**
- The filesystem-as-message-bus pattern requires a stable, machine-readable contract. Downstream agents consume this output without human mediation.
- The `schema_version` field allows downstream consumers to detect breaking changes and fail loudly rather than silently misparse data.
