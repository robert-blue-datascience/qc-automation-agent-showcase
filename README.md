# QC Automation Agent

> **This is an architecture showcase repository.** The source code for this project is private. This repository contains architecture documentation, design decisions, and results.

A production agentic AI system that automates data quality scoring across an active O&G well portfolio. The agent discovers wells via API, evaluates 30 data quality modules per well against deterministic rule sets, and publishes scored operator summaries to a project management dashboard via GraphQL.

---

## The Problem

Manual QC scoring across 150+ active wells across 20+ operators is unsustainable. The work is repetitive, time-sensitive, and error-prone at scale. A single run requires checking 30 data quality indicators per well, computing per-operator aggregate scores, and publishing results in a format that operations teams can act on.

The first attempt at automation used browser-based scraping (Playwright). This worked at small scale but degraded after 45+ minutes of continuous operation due to DOM state accumulation -- silently producing incorrect scores without raising any error signal. 111 incorrect scores were published to the dashboard before the issue was identified. This incident drove two major architectural decisions: the migration from browser extraction to a direct API adapter pattern, and the addition of a comprehension gate to the development workflow.

---

## Architecture

### Four-Layer Design

```
Orchestrator (LangGraph state machine, 10 nodes, 3 conditional edges)
    --> API Layer (async httpx, JWT auth with auto-refresh, adapter functions)
    --> Rule Engine (pure Python, deterministic, no LLM, 30 check modules)
    --> Reporter (score calculator, JSON reports, Supabase writes, GraphQL publishing)
Cross-cutting: Guardrails (security gate, rate limiter, audit logger, log sanitizer)
```

**Layer 1: Orchestrator (LangGraph)**

The orchestrator is a typed state machine (`QCAgentState`) with 10 nodes and 3 conditional edges. LangGraph was chosen over a plain async loop to enforce a formal state contract: every node declares what it reads and writes, LangGraph merges partial updates, and no node can accidentally read stale state from a prior well.

Live resources (API client, rule engine, audit logger, rate limiter, resource cache) are held on the graph instance, not in the state dict. LangGraph state must be JSON-serializable; an open HTTP connection pool or file handle cannot be serialized. Closure injection makes live resources available inside each node without LangGraph seeing them.

The 10-node graph:
1. `security_gate` -- validates environment before any network activity
2. `discover_wells` -- queries search API using operator whitelist to build the well queue
3. `initialize_run` -- generates run metadata, opens audit log, builds ordered check queue
4. `select_well` -- pops the next well, fetches well detail, detects status mismatches
5. `process_check` -- concurrent two-wave evaluation of all 30 checks per well
6. `save_well_results` -- accumulates per-well results, clears resource cache (cross-well isolation)
7. `generate_report` -- assembles versioned JSON report with per-well check results and scores
8. `publish_supabase` -- writes per-well results to the persistent score-of-record store
9. `publish_monday` -- upserts operator summary row to the dashboard via GraphQL
10. `cleanup` -- flushes audit logger

**Layer 2: API Extraction (httpx + Adapter Pattern)**

The original Playwright browser extraction layer was replaced by direct API calls via `async httpx` (ADR-002). A pure translation layer (`api_adapter.py`) reshapes API JSON responses into the flat dict format the rule engine already expected, preserving 100% of existing rule logic without modification.

Key API layer behaviors:
- JWT bearer token auto-refresh before expiration (transparent to orchestrator)
- Bearer token registered with `LogSanitizer` immediately on receipt, before any log call
- Connection pool held open via async context manager for the full run duration
- `resource_cache` within each well avoids redundant API calls for shared endpoints (6 BHA checks share one list fetch, for example)
- Request coalescing: when concurrent checks request the same endpoint simultaneously, only one in-flight fetch goes to the network; others wait on a per-endpoint lock

**Layer 3: Rule Engine (Pure Python, Deterministic)**

The rule engine is the only place where pass/fail decisions are made. It is intentionally isolated from all other layers -- no network calls, no LLM inference, no side effects.

Architecture:
- 30 checks across 10 Python evaluation modules
- Each check has a corresponding YAML config in `config/modules/` defining function reference, category, parameters, and dependency declarations
- Dispatch: YAML `evaluation.function` field (e.g., `"surveys.evaluate_surveys"`) resolved via `importlib` at engine construction
- Dependency resolution: when Check A requires Check B's result, the engine short-circuits without calling Check A's eval function
- Additive override: checks declared `additive: true` have NO results silently overridden to N_A (the data observation and the scoring decision are separated)
- 7 status values: `YES`, `YES_WITSML`, `YES_EMAIL`, `NO`, `PARTIAL`, `N_A`, `INCONCLUSIVE`

**The 10 evaluation modules and the checks they cover:**

| Module | Checks Covered |
|--------|---------------|
| `witsml.py` | WITSML Connected |
| `surveys.py` | Surveys, Survey Program, Survey Corrections |
| `geosteering.py` | Live Geosteering |
| `drilling.py` | EDM Files, Well Plans |
| `bha.py` | BHA Distro, BHA Comments, BHA Uploads, BHA Failure Reports, BHA Full Components, Post Run BHAs |
| `mud.py` | Mud Report Distro, Mud Program |
| `engineering.py` | Wellbore Diagrams |
| `file_drive.py` | File Drive BHAs, File Drive Well Plans, File Drive Drill Prog, File Drive Mud Reports |
| `universal.py` | NPT Tracking, Cost Analysis, Rig Inventory Data, Tool Catalog Data, Formation Tops, Roadmaps, Engineering Scenarios, AI Drill Prog, AFE Curves |
| `location.py` | Location (historical mode only) |

**Layer 4: Reporter (Supabase + GraphQL)**

Two publish destinations with distinct roles:

- **Supabase** (score-of-record): per-well check results written after every well, unconditionally. `--no-publish` does not suppress Supabase writes -- it only suppresses the dashboard update. Supabase is the audit artifact; the dashboard is the communication artifact.
- **GraphQL dashboard**: single operator-level summary row (score, well count, last run date) upserted after all wells for that operator complete.

Per-well scoring uses a weighted average over checks that produce a numeric score (`YES`=1.0, `PARTIAL`=0.5, `NO`=0.0). `N_A` and `INCONCLUSIVE` checks are excluded from the denominator entirely.

---

## Performance

The sequential browser extraction baseline was ~75-90 seconds per well. After the API migration:

| Execution Model | Time per Well |
|----------------|---------------|
| Sequential browser extraction (v0.6.x) | ~75-90 seconds |
| Sequential API extraction (v0.7.x) | ~1m 47s |
| Concurrent API extraction -- two-wave gather (v0.8.x+) | ~3-5 seconds |

The concurrent model uses `asyncio.gather` with two waves (wave 1: independent checks; wave 2: dependency-dependent checks) and a semaphore to cap simultaneous in-flight requests. Request coalescing prevents redundant network calls when concurrent checks share an endpoint.

---

## 5-Layer Quality Gate

Every change to this project passes through a five-layer review pipeline before merging:

| Layer | Mechanism |
|-------|-----------|
| 1. `/review` | Slash command: Rob reads and understands every diff |
| 2. `/verify` | Slash command: validates that comprehension gate was passed |
| 3. `/code-review` | Gemini Code Reviewer Gem evaluates architectural integrity |
| 4. `pip-audit` | Dependency vulnerability scan (report-only; no automated fixes) |
| 5. `/sec-sweep` or `/sec-batch` | Gemini SecReview Gem full security audit on trigger or batch |

The comprehension gate (Layer 2) was added after the April 3 scoring incident: design assumptions that were not fully understood before shipping caused 111 incorrect scores to be published. The gate requires demonstrated understanding of every design decision before implementation specs ship.

---

## Three-Tool Development Workflow

This project uses three distinct AI tools with non-overlapping roles:

| Tool | Role |
|------|------|
| Claude.ai | Architect -- design decisions, ADRs, SPEC.md authorship, scope enforcement |
| Claude Code | Implementer -- code, tests, repo operations against blueprints |
| Gemini (Code Reviewer + SecReview Gems) | Reviewer -- independent code quality and security audits |

The role separation exists to prevent the implementer from reviewing its own work. Code review by the same system that wrote the code is not independent review.

---

## Security Posture

The agent enforces a startup security gate before any network activity. Checks include: telemetry env vars absent, all required credentials present, `.env` and output directories listed in `.gitignore`, no prohibited env vars set. On any failure, the gate raises an exception and the agent exits. No partial runs, no overrides.

The security audit (April 2026) covered six categories: API surface, authentication and access control, configuration and secrets management, data layer, external integrations, and rules/file handling. Static analysis runs pre-execution over the full codebase AST to verify all network calls target the domain allowlist.

---

## Results

- **30 QC modules** evaluated per well, deterministically, with zero LLM involvement in pass/fail decisions
- **899 tests** passing across all layers (unit + integration) as of v0.9.0
- **95% reduction in per-well execution time** vs. the original browser extraction baseline
- **API-driven well discovery** replaced a manually maintained input file, eliminating discovery lag for newly spudded wells
- **Supabase as score-of-record** enables trend analysis and deviation detection that was impossible with a flat dashboard
- **0 static analysis violations** (AST-based network allowlist enforcement)
- **Historical run mode** added at v0.9.0: 13-check evaluation set for completed wells using the same rule engine with mode-specific YAML variants
