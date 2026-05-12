# Architecture

> **This is an architecture showcase repository.** Source code is private. This document describes system design from ADRs, SPEC.md, and CLAUDE.md.

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

LangGraph was chosen over a plain `asyncio` loop for the formal state contract it enforces. Every piece of data in the system must be declared in `QCAgentState` (a TypedDict) before any node can write to it. LangGraph silently discards undeclared keys during state merges -- this is a known failure mode documented in the project after several keys were lost before being discovered and added to the schema. The rule: when a node returns a new key, it must be added to `QCAgentState` in the same commit.

The conditional edge system makes loop control explicit and testable: `route_after_select_well`, `route_after_check`, and `route_after_save_well` are pure functions that read one state key and return a node name. They can be tested in isolation without running the graph.

### State Contract

All state in `QCAgentState` is JSON-serializable. Live resources (API client, rule engine, audit logger, resource cache) are held on the graph class instance, not in state. Node functions receive live resources via closure injection -- from LangGraph's perspective, each node is a plain callable.

This design means the API client connection pool stays open across all wells in a run without being passed through state, and the audit logger file handle is never serialized.

### Well Isolation

Non-Negotiable #1 is cross-operator/cross-well data isolation. Three mechanisms enforce it:

1. **`resource_cache.clear()`**: called by `save_well_results_node` after every well. The cache holds API responses within a single well's check execution. Clearing it ensures BHA data from Well A is never visible during Well B's evaluation.
2. **`completed_wells` accumulation**: only contains results for the current operator's wells. `run_all()` invokes the graph once per operator with a fresh state.
3. **Operator scope validation**: `select_well_node` compares the fetched well's operator ID against the whitelist operator ID. A mismatch logs an audit event and skips the well rather than evaluating it under the wrong operator.

### Concurrent Check Execution (v0.8.0+)

`process_check_node` processes all 30 checks per well in a single call using a two-wave pattern:

- **Wave 1**: All checks with no declared dependencies run concurrently via `asyncio.gather`. These compete for the semaphore (default: 8 concurrent).
- **Wave 2**: Checks that declare a dependency on a Wave 1 result. Before executing, each Wave 2 check inspects the accumulated Wave 1 results. If the dependency result is `INCONCLUSIVE`, the check inherits `INCONCLUSIVE` without making any API call.

Request coalescing: when concurrent checks request the same endpoint, only one in-flight fetch goes to the network. A per-endpoint `asyncio.Lock` gates subsequent callers, which read from the cache on lock release. The `_FETCH_FAILED` sentinel (not `None`) marks a failed fetch, distinguishing it from a cache miss.

Circuit breakers at two scopes:
- **Per-well**: consecutive and total timeout counters. When thresholds are reached, remaining checks are skipped and the well is marked `circuit_breaker_aborted=True`.
- **Run-level**: tracks consecutive aborted wells across the full run. When the limit is reached, the well queue is drained to halt the run.

---

## Layer 2: API Extraction Layer

### The Adapter Pattern (Option A)

When browser-based extraction was replaced with direct API calls, two migration strategies were evaluated:

- **Option A**: Keep the 29 existing evaluation functions unchanged. Add a translation layer (`api_adapter.py`) that reshapes API JSON into the `extracted_data` dicts the evaluation functions already expect.
- **Option B**: Rewrite the evaluation functions to consume API JSON directly, eliminating the translation layer.

Option A was chosen. The 29 evaluation functions and their 600+ tests represent validated, production-verified decision logic. Option B would have required rewriting all 29 functions and re-validating correctness -- high risk, no correctness benefit. Option B is tracked as a deferred task and is the preferred long-term direction.

The adapter functions are pure (no I/O, no side effects) and independently testable: `dict in, dict out`.

### JWT Lifecycle

The `APIAuth` class manages the full JWT lifecycle autonomously:
- On login, the JWT payload is decoded to determine the exact expiration time
- `get_headers()` is called before every request; if the token is expired or near expiry, re-authentication happens transparently
- The bearer token is registered with `LogSanitizer` immediately on receipt, before any attribute assignment, so token leakage via constructor exception is prevented

### Well Discovery (v0.9.0+)

Well discovery replaced a manually maintained CSV input file. An operator whitelist YAML declares operator identifiers, status ID lists (for active and historical run modes), and optional geographic filters. `discover_wells_node` runs a count pre-flight before the full search, comparing against a configurable discovery ceiling to prevent runaway runs if well counts change unexpectedly.

---

## Layer 3: Rule Engine

### Design Invariants

The rule engine is the only place where pass/fail decisions are made. Three invariants are enforced unconditionally:

1. **No network calls**: evaluation functions are pure `(dict, dict) -> CheckResult`. No I/O.
2. **No LLM inference**: every result is produced by explicit conditional logic.
3. **INCONCLUSIVE for ambiguity**: missing or unparseable data returns `INCONCLUSIVE`, never a guess.

### YAML Dispatch

Each of the 30 checks has a YAML config in `config/modules/`. The YAML defines:
- `check_name`, `check_number`, `category` -- identity
- `additive` -- if true, NO results are silently overridden to N_A
- `evaluation.function` -- dotted reference to the evaluation function (e.g., `"witsml.evaluate_witsml_connected"`)
- `evaluation.params` -- configurable thresholds (e.g., `threshold_minutes: 90` for WITSML staleness)
- `dependencies` -- optional block declaring which prior check result gates this check

The engine builds the function registry at construction via `importlib`. Adding a new check requires a new YAML file and a new evaluation function -- no changes to the engine itself.

### Interface Contract

All evaluation functions share the same signature: `(extracted_data: dict, config: dict) -> CheckResult`. This contract makes the dispatch table self-documenting and allows any evaluation function to be tested with mock data in isolation.

---

## Layer 4: Reporter

### Dual Publish Architecture

**Supabase** (score-of-record): per-well check results are written unconditionally after each well evaluates. The `--no-publish` flag suppresses only the dashboard update -- Supabase receives every result regardless. This distinction reflects their different purposes: Supabase is an audit artifact with persistent history; the dashboard is a communication artifact.

**Dashboard (GraphQL)**: a single operator-level summary row is upserted after all wells for an operator complete. The row contains score, well count, last run date, and a dashboard link. Not written for historical runs, ad-hoc single-well runs, or when `--no-publish` is set.

### Versioned Output Schema

Every run produces a JSON report conforming to a versioned schema. The schema is the data contract between the QC agent and any downstream agents in the agentic network. The `schema_version` field allows downstream consumers to detect breaking changes and fail loudly rather than silently misparse data.

---

## Cross-Cutting: Guardrails

### Security Gate

Six checks run before any node executes:
1. LangChain tracing disabled (`LANGCHAIN_TRACING_V2` absent or false)
2. No LangChain API key present
3. All required credentials set and non-empty
4. `.env` exists on disk and is listed in `.gitignore`
5. Output directory listed in `.gitignore`
6. No prohibited telemetry env vars set

Any failure raises `SecurityPolicyViolation` immediately. No partial runs.

### Static Analysis (Pre-execution)

`static_analysis.py` parses the full codebase AST before the orchestrator boots. It verifies all network calls target an exact-match domain allowlist (using `urlparse().netloc` for the comparison, not substring matching -- the substring approach is bypassable via path-segment injection). The domain allowlist is hardcoded as source-level security policy, not externalized to config, so changes require code review.

### Log Sanitizer

The `LogSanitizer` maintains a registry of secrets registered at runtime (including JWT tokens received during a run). Before any log entry is written to disk, the sanitizer scans the payload for registered secrets and replaces them with `[REDACTED]`. Evidence fields longer than 128 characters are truncated and a SHA-256 hash is stored alongside the truncated value for integrity verification.

### Audit Logger

Every node action, check result, API failure, and routing decision is written as a structured event to a per-run audit log. The `CHECK_RESULT` event for every check is logged with `check_name`, `status`, and `evidence_value`, providing a complete decision trail that can reconstruct what the engine found for every check on every well.
