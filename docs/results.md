# Results and Outcomes

> **This is an architecture showcase repository.** Source code is private. Results are reported in general terms without client-identifiable data.

---

## Performance Outcomes

| Metric | Before (Browser) | After (API + Concurrent) | Improvement |
|--------|-----------------|--------------------------|-------------|
| Per-well execution time | 75-90 seconds | 3-5 seconds | ~95% reduction |
| Full portfolio run time | 172 minutes (first production run) | Substantially reduced | Scales linearly with well count |
| Silent failure mode | Browser degradation after 45+ min, no error signal | API failures return INCONCLUSIVE per check, run continues | Eliminated |

The first full production run (April 2026) processed 111 wells across 21 operators in 172 minutes using the browser extraction path. After the API migration and concurrent execution, the same portfolio runs in a fraction of the time with no browser degradation risk.

---

## Quality Outcomes

**Test coverage (v0.9.0):** 899 tests passing across all layers.

| Layer | Test Scope |
|-------|-----------|
| Rule engine (unit) | All 30 checks: happy path, failure cases, edge cases, dependency resolution, additive override |
| Rule engine (integration) | All 30 checks through real YAML configs with mock extracted_data |
| API adapter (unit) | All adapter functions: correct dict shapes for each check's strategy |
| Orchestrator (unit) | All 10 nodes and 3 routing functions in isolation |
| Guardrails (unit) | Security gate, rate limiter, audit logger, log sanitizer, static analysis |

**Zero LLM involvement in pass/fail decisions.** Every QC score is produced by deterministic conditional logic. The same input always produces the same result. Every result traces to a specific condition in a specific evaluation function.

**Zero static analysis violations.** AST-based network call analysis verifies all HTTP requests target the exact-match domain allowlist. This runs pre-execution before the graph boots.

---

## Operational Outcomes

**API-driven well discovery (v0.9.0):** Replaced a manually maintained CSV input file. The platform's search API is now the authoritative source for active wells. Newly spudded wells appear in QC runs automatically; there is no discovery lag between a well going active on the platform and entering the evaluation queue.

**Supabase as score-of-record (v0.9.0):** Per-well check results are now queryable across runs. Trend analysis and deviation detection (flagging wells whose scores changed between runs) are possible with indexed database queries rather than re-parsing JSON files on disk.

**Historical run mode (v0.9.0):** The same pipeline evaluates completed wells using a 13-check subset with mode-specific YAML variants. This reuses 100% of the rule engine and orchestrator with only configuration changes.

---

## Security Outcomes

**Security audit (April 2026)** covered 6 categories: API surface, authentication and access control, configuration and secrets management, data layer, external integrations, and rules/file handling. The audit was conducted via Gemini SecReview Gem in batch mode, producing structured findings documents per category.

**Key security properties as of v0.9.0:**
- Startup security gate blocks execution on any environment policy violation
- Bearer tokens registered with log sanitizer before any log call (no token leakage window)
- Credential registry (`credentials.py`) is the single source of truth for all required and secret env vars -- both the security gate and the log sanitizer import from the same set
- Domain allowlist uses exact `netloc` comparison via `urlparse`, preventing path-segment bypass attacks (e.g., `https://evil.com/allowed-domain.com` is correctly rejected)
- Evidence values in audit logs are truncated at 128 chars with SHA-256 hash for integrity verification, limiting PII exposure in local logs

---

## Lessons Learned

**The April 3 incident (111 incorrect scores published):** Browser session degradation after 45+ minutes of continuous operation produced incorrect scores without raising any error signal. The agent appeared to run successfully while silently producing wrong answers. This drove three permanent changes:
1. Full migration from browser extraction to API calls (eliminates the degradation failure mode)
2. The comprehension gate in the development workflow (prevents shipping design assumptions not fully understood)
3. Operator scope validation in `select_well_node` (explicit check that fetched well belongs to the expected operator)

**The `pip-audit --fix` incident (April 2026):** Running `pip-audit --fix` automatically broke `pydantic/pydantic-core` version coupling, killing the scheduled task and blocking all runs for two days. The safe workflow is now documented: `pip-audit` (report only), manually upgrade the flagged package, run the full test suite, freeze dependencies. `pip-audit --fix` is prohibited in the project conventions.

**LangGraph silent state drop:** Keys returned by a node that are not declared in `QCAgentState` are silently discarded during the state merge. Multiple keys were lost this way before the rule was established: when a node returns a new key, it must be added to `QCAgentState` in the same commit.
