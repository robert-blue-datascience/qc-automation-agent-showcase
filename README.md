# QC Automation Agent

> **This is an architecture showcase repository.** The source code for this project is private. This repository contains architecture documentation, design decisions, and results. **No source code, client data, API credentials, or proprietary business logic appears anywhere in this repository.**

A production agentic AI system that automates data quality scoring across an active well portfolio on an industrial cloud platform. The agent discovers wells via API, evaluates data quality modules per well against deterministic rule sets, and publishes scored operator summaries to a project management dashboard via GraphQL.

---

## Data Security and Client Isolation

This project was designed from the ground up with client data protection as a non-negotiable constraint. These properties were enforced structurally (in code and tests), not by policy alone:

- **No client data was ever exposed to any LLM or external AI service.** All scoring decisions are made by a deterministic Python rule engine with zero LLM involvement. The same input always produces the same output, and every result traces to a specific condition in a specific evaluation function.
- **No client data left the local machine.** Logs, audit trails, and run output are local-only. The startup security gate blocks execution if any external telemetry or tracing service is configured.
- **Bearer tokens and credentials are scrubbed from all log output.** A log sanitizer registers every secret at the moment of receipt (before any subsequent code runs) and redacts them from all structured log entries before disk write.
- **Cross-client data isolation is enforced per-well.** API response caches are cleared between every well evaluation. Operator scope is validated on every well fetch. State from one operator's wells is never visible during another operator's evaluation.
- **Static analysis enforces a network allowlist.** An AST-based pre-execution scan verifies that all network calls in the codebase target an exact-match domain allowlist. The allowlist is hardcoded in source (not config) so changes require code review.

These properties are verified by dedicated tests, enforced by the startup security gate, and audited by an independent security reviewer (Gemini SecReview) on every release.

---

## The Problem

Manual QC scoring across hundreds of active wells and dozens of operators is unsustainable. The work is repetitive, time-sensitive, and error-prone at scale. A single run requires checking multiple data quality indicators per well, computing per-operator aggregate scores, and publishing results in a format that operations teams can act on.

The first attempt at automation used browser-based scraping (Playwright). This worked at small scale but degraded after 45+ minutes of continuous operation due to DOM state accumulation, silently producing incorrect scores without raising any error signal. Over a hundred incorrect scores were published to the dashboard before the issue was identified. This incident drove two major architectural decisions: the migration from browser extraction to a direct API adapter pattern, and the addition of a comprehension gate to the development workflow.

---

## Architecture

### Four-Layer Design

```
Orchestrator (LangGraph state machine, 10 nodes, 3 conditional edges)
    --> API Layer (async httpx, JWT auth with auto-refresh, adapter functions)
    --> Rule Engine (pure Python, deterministic, no LLM, 30 check modules)
    --> Reporter (score calculator, JSON reports, database writes, GraphQL publishing)
Cross-cutting: Guardrails (security gate, rate limiter, audit logger, log sanitizer)
```

**Layer 1: Orchestrator (LangGraph)**

The orchestrator is a typed state machine with 10 nodes and 3 conditional edges. LangGraph was chosen over a plain async loop to enforce a formal state contract: every node declares what it reads and writes, LangGraph merges partial updates, and no node can accidentally read stale state from a prior well.

Live resources (API client, rule engine, audit logger, rate limiter, resource cache) are held on the graph instance, not in the state dict. LangGraph state must be JSON-serializable; an open HTTP connection pool or file handle cannot be serialized. Closure injection makes live resources available inside each node without LangGraph seeing them.

The 10-node graph:
1. `security_gate` -- validates environment before any network activity
2. `discover_wells` -- queries search API using operator whitelist to build the well queue
3. `initialize_run` -- generates run metadata, opens audit log, builds ordered check queue
4. `select_well` -- pops the next well, fetches well detail, detects status mismatches
5. `process_check` -- concurrent two-wave evaluation of all checks per well
6. `save_well_results` -- accumulates per-well results, clears resource cache (cross-well isolation)
7. `generate_report` -- assembles versioned JSON report with per-well check results and scores
8. `publish_database` -- writes per-well results to the persistent score-of-record store
9. `publish_dashboard` -- upserts operator summary row to the dashboard via GraphQL
10. `cleanup` -- flushes audit logger

**Layer 2: API Extraction (httpx + Adapter Pattern)**

The original Playwright browser extraction layer was replaced by direct API calls via `async httpx`. A pure translation layer (`api_adapter.py`) reshapes API JSON responses into the flat dict format the rule engine already expected, preserving 100% of existing rule logic without modification.

Key API layer behaviors:
- JWT bearer token auto-refresh before expiration (transparent to orchestrator)
- Bearer token registered with log sanitizer immediately on receipt, before any log call
- Connection pool held open via async context manager for the full run duration
- Resource cache within each well avoids redundant API calls for shared endpoints (related checks share one list fetch)
- Request coalescing: when concurrent checks request the same endpoint simultaneously, only one in-flight fetch goes to the network; others wait on a per-endpoint lock

**Layer 3: Rule Engine (Pure Python, Deterministic)**

The rule engine is the only place where pass/fail decisions are made. It is intentionally isolated from all other layers: no network calls, no LLM inference, no side effects.

Architecture:
- 30 checks across 10 Python evaluation modules, covering categories such as real-time data connectivity, survey management, drilling operations, component tracking, reporting, and file management
- Each check has a corresponding YAML config defining function reference, category, parameters, and dependency declarations
- Dispatch: YAML `evaluation.function` field resolved via `importlib` at engine construction
- Dependency resolution: when Check A requires Check B's result, the engine short-circuits without calling Check A's eval function
- Additive override: checks declared `additive: true` have NO results silently overridden to N_A (the data observation and the scoring decision are separated)
- Multiple status values covering confirmed presence, absence, partial compliance, not-applicable, and inconclusive states

**Layer 4: Reporter (Database + GraphQL)**

Two publish destinations with distinct roles:

- **Score-of-record database**: per-well check results written after every well, unconditionally. `--no-publish` does not suppress database writes; it only suppresses the dashboard update. The database is the audit artifact; the dashboard is the communication artifact.
- **GraphQL dashboard**: single operator-level summary row (score, well count, last run date) upserted after all wells for that operator complete.

Per-well scoring uses a weighted average over checks that produce a numeric score (confirmed = 1.0, partial = 0.5, absent = 0.0). Not-applicable and inconclusive checks are excluded from the denominator entirely.

---

## Performance

| Execution Model | Time per Well |
|----------------|---------------|
| Sequential browser extraction (v0.6.x) | ~75-90 seconds |
| Sequential API extraction (v0.7.x) | ~1m 47s |
| Concurrent API extraction -- two-wave gather (v0.8.x+) | ~3-5 seconds |

The concurrent model uses `asyncio.gather` with two waves (wave 1: independent checks; wave 2: dependency-dependent checks) and a semaphore to cap simultaneous in-flight requests. Request coalescing prevents redundant network calls when concurrent checks share an endpoint.

---

## Results

- **30 QC modules** evaluated per well, deterministically, with zero LLM involvement in pass/fail decisions
- **800+ tests** passing across all layers (unit + integration)
- **95% reduction in per-well execution time** vs. the original browser extraction baseline
- **API-driven well discovery** replaced a manually maintained input file, eliminating discovery lag for newly spudded wells
- **Score-of-record database** enables trend analysis and deviation detection that was impossible with a flat dashboard
- **0 static analysis violations** (AST-based network allowlist enforcement)
- **Historical run mode**: reduced evaluation set for completed wells using the same rule engine with mode-specific YAML variants
