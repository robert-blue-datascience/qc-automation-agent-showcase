# Architecture

> **This is an architecture showcase repository.** Source code is private. No client data, API credentials, or proprietary business logic appears in this document.

---

## System Overview

The QC Automation Agent is a four-layer agentic system that discovers wells via API, evaluates 30 data quality modules per well using deterministic rules, and publishes scored results to a persistent database and a project management dashboard.

The layers are strictly decoupled:
- The orchestrator knows about state management and graph topology, not data formats
- The API layer knows about network calls and JSON translation, not business rules
- The rule engine knows about pass/fail logic, not where data came from
- The reporter knows about scoring and publishing, not orchestration

---

## Layer 1: Orchestrator (LangGraph StateGraph)

### Why LangGraph

LangGraph was chosen over a plain `asyncio` loop for the formal state contract it enforces. Every piece of data in the system must be declared in the state schema (a TypedDict) before any node can write to it. LangGraph silently discards undeclared keys during state merges, which is a known failure mode documented in the project. The rule: when a node returns a new key, it must be added to the state schema in the same commit.

The conditional edge system makes loop control explicit and testable: routing functions are pure functions that read one state key and return a node name. They can be tested in isolation without running the graph.

### State Contract

All state is JSON-serializable. Live resources (API client, rule engine, audit logger, resource cache) are held on the graph class instance, not in state. Node functions receive live resources via closure injection. This keeps the API client connection pool open across all wells without serialization, and prevents file handles from entering the state dict.

### Well Isolation

Cross-operator and cross-well data isolation is a non-negotiable. Three mechanisms enforce it:

1. **Resource cache clearing**: called after every well. The cache holds API responses within a single well's check execution. Clearing it ensures data from Well A is never visible during Well B's evaluation.
2. **Per-operator graph invocation**: the orchestrator is invoked once per operator with a fresh state.
3. **Operator scope validation**: each well fetch is validated against the expected operator. A mismatch logs an audit event and skips the well.

### Concurrent Check Execution

All 30 checks per well are processed in a single call using a two-wave pattern:

- **Wave 1**: All checks with no declared dependencies run concurrently via `asyncio.gather` with a semaphore cap.
- **Wave 2**: Checks that declare a dependency on a Wave 1 result. If the dependency result is inconclusive, the check inherits that status without making any API call.

Request coalescing prevents redundant network calls: when concurrent checks request the same endpoint, only one in-flight fetch goes to the network. A per-endpoint lock gates subsequent callers, which read from the cache on lock release. A sentinel value (not `None`) marks a failed fetch, distinguishing it from a cache miss.

Circuit breakers operate at two scopes: per-well (consecutive timeout counters trigger remaining-check skip) and run-level (consecutive aborted wells trigger queue drain to halt the run).

---

## Layer 2: API Extraction Layer

### The Adapter Pattern

When browser-based extraction was replaced with direct API calls, two migration strategies were evaluated:

- **Option A (chosen)**: Keep existing evaluation functions unchanged. Add a pure translation layer that reshapes API JSON into the dict format the evaluation functions already expect.
- **Option B (deferred)**: Rewrite evaluation functions to consume API JSON directly.

Option A was chosen because the evaluation functions and their hundreds of tests represent validated, production-verified decision logic. Rewriting them would require full re-validation with no correctness benefit. The adapter functions are pure (no I/O, no side effects) and independently testable.

### JWT Lifecycle

The auth class manages the full JWT lifecycle autonomously: token payload is decoded for exact expiration time, `get_headers()` transparently re-authenticates when needed, and the bearer token is registered with the log sanitizer immediately on receipt (before any attribute assignment) so token leakage via constructor exception is prevented.

### Well Discovery

Well discovery replaced a manually maintained CSV input file. An operator whitelist YAML declares operator identifiers and status filters. A count pre-flight runs before the full search to prevent runaway runs if well counts change unexpectedly.

---

## Layer 3: Rule Engine

### Design Invariants

The rule engine is the only place where pass/fail decisions are made. Three invariants are enforced unconditionally:

1. **No network calls**: evaluation functions are pure (dict in, result out). No I/O.
2. **No LLM inference**: every result is produced by explicit conditional logic.
3. **Inconclusive for ambiguity**: missing or unparseable data returns an inconclusive status, never a guess.

### YAML Dispatch

Each of the 30 checks has a YAML config defining identity, function reference, configurable thresholds, and dependency declarations. The engine builds the function registry at construction via `importlib`. Adding a new check requires a new YAML file and a new evaluation function; no changes to the engine itself.

### Interface Contract

All evaluation functions share the same signature: `(extracted_data: dict, config: dict) -> CheckResult`. This makes the dispatch table self-documenting and allows any evaluation function to be tested with mock data in isolation.

---

## Layer 4: Reporter

### Dual Publish Architecture

**Score-of-record database**: per-well check results are written unconditionally after each well. The `--no-publish` flag suppresses only the dashboard update. This reflects their different purposes: the database is an audit artifact with persistent history; the dashboard is a communication artifact.

**Dashboard (GraphQL)**: a single operator-level summary row is upserted after all wells for an operator complete. Not written for historical runs, ad-hoc single-well runs, or when `--no-publish` is set.

### Versioned Output Schema

Every run produces a JSON report conforming to a versioned schema. The `schema_version` field allows downstream consumers to detect breaking changes and fail loudly rather than silently misparse data.

---

## Cross-Cutting: Guardrails

### Security Gate

Six checks run as the first node before any network activity. Any failure raises an exception immediately. No partial runs, no overrides:

1. External tracing integration disabled (checked by value, not just presence)
2. No external tracing API key present
3. All required credentials set and non-empty
4. `.env` exists on disk and is listed in `.gitignore`
5. Output directory listed in `.gitignore`
6. No prohibited telemetry environment variables set

### Static Analysis (Pre-execution)

A pre-execution AST scan verifies all network calls target an exact-match domain allowlist using `urlparse().netloc` for comparison (not substring matching, which is bypassable via path-segment injection). The allowlist is hardcoded in source so changes require code review.

### Log Sanitizer

The log sanitizer maintains a registry of secrets registered at runtime (including JWT tokens received during a run). Before any log entry is written to disk, registered secrets are replaced with `[REDACTED]`. Evidence fields longer than 128 characters are truncated with a SHA-256 hash stored alongside for integrity verification.

### Audit Logger

Every node action, check result, API failure, and routing decision is written as a structured event to a per-run audit log. The complete decision trail can reconstruct what the engine found for every check on every well.
