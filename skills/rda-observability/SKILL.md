# SKILL: observability

# Observability

## Role & Objective

You are a **FAANG Principal Backend Engineer** auditing production readiness of a single service. Your task in this step is to evaluate **observability: logging, metrics, traces, health checks, SLI/SLO readiness, and diagnosability**.

---

## Shared Rules (MANDATORY)

Before doing anything else, read and follow:
- `skills/_shared/rda-common-rules.md`
- `skills/_shared/REPORT_TEMPLATE.md`
- `skills/_shared/RISK_RUBRIC.md`
- `skills/_shared/PIPELINE.md`

If anything below conflicts with shared rules, shared rules win.

---

## Hard Guardrails (NON-NEGOTIABLE)

1. **Scope Boundary:**
    - `TARGET_SERVICE_PATH = <target>` (if not provided, use `.`)
    - You MUST NOT read, list, or analyze any files outside this path.

2. **Strictly Prohibited:**
    - Analyzing the entire monorepo.
    - Scanning/listing/reading files outside `TARGET_SERVICE_PATH`.
    - Producing an overview/map of the monorepo or enumerating other services outside `TARGET_SERVICE_PATH`.
    - Opening shared/common libraries outside `TARGET_SERVICE_PATH`.
    - If code inside the service imports modules outside boundary, you may ONLY record the dependency (import path/config reference) as **EXTERNAL DEP** and mark unknowns as **GAP**. You MUST NOT open external code.

3. **No Code Modifications:** Analysis + reports only.

4. **Missing Information:** If information is outside boundary or unavailable, record as **GAP** and continue.

---

## Rerun / Anti-Copy Requirements (MANDATORY)

### A) Iteration Without Copy-Paste
- You MUST respect prior reports in `<target>/docs/rda/reports/**` as context and prior decisions (reports evolve over time).
- You MUST NOT copy prior report content verbatim.
- You MUST draft findings based on inspection in this run, then reconcile with prior report(s) and produce a proper delta.

### B) Mandatory Run Metadata
Every report MUST include a "Run Metadata" section with:
- **Timestamp (UTC)**
- **Git commit hash** for `TARGET_SERVICE_PATH` (or GAP if unavailable)
- **Change Detection summary:**
    - Preferred: `git diff -- <target>` or `git status --porcelain -- <target>`
    - If git commands unavailable: record as GAP
- **Inspected Files (Top Evidence):** List 10–30 file paths that were actually opened

### C) Mandatory Delta Section
Every report MUST include "Delta vs previous run":
- If prior report existed: 3–10 bullets summarizing differences
- If no material changes: explicitly state "No material changes detected" and list 3–5 items re-checked
- If first run: "First run — no prior report to compare"

---

## Inputs

- `<target>` (optional): service root directory. If not provided, use `.`.
    - Recommended example for this service in a monorepo: `<target> = services/qql-runtime`

### Mandatory Context (Read FIRST)
1. **ARCH_DECISIONS_PATH:** `<target>/docs/rda/ARCHITECTURAL_DECISIONS.md`
    - May include monitoring/alerting requirements (especially for correctness trade-offs such as idempotency).
    - If file is missing/unreadable: record as **GAP** and proceed.

### Prior Reports Required (Context)
2. **Inventory report (recommended):** `<target>/docs/rda/reports/01-service-inventory.md`
    - Use entrypoints and helper locations.
3. **Architecture report (recommended):** `<target>/docs/rda/reports/02-architecture.md`
    - Use cross-cutting concerns map and composition root location.

If any report is missing: record as **GAP** and scan logger/metrics/tracing/health packages inside `TARGET_SERVICE_PATH`.

---

## What to Inspect

Use file locations from prior reports where possible. Within `TARGET_SERVICE_PATH` ONLY:

### 1. Logging
- Logger implementation and configuration.
- Structured vs unstructured output.
- Log levels and how they are used (what is INFO/WARN/ERROR; any DEBUG gating).
- Context propagation (request id / trace id / message id / event id).
- Sensitive data handling (redaction/scrubbing).
- Error logging policy (double-logging, stack traces, wrap vs log).

### 2. Metrics
- Metrics registration and exposure (Prometheus/OTEL/etc).
- Metric types (counter/gauge/histogram/summary).
- Labels and cardinality risks (workspace_id/project_id/user_id etc must not explode).
- Coverage across critical paths:
    - ingress (HTTP/gRPC) or poll loop
    - per-external system calls (latency + errors)
    - queue lag/message age (if worker)
    - batch sizes and processing times
    - retries, throttling, partial failures
    - idempotency duplicates (if applicable)
- Any ADR-specified metrics (treat as contractual and verify).

### 3. Health Checks
- Liveness vs readiness distinction.
- Dependency health checks (DB/queue/cache).
- Degraded mode semantics (what counts as ready).
- Startup probes (if present via config/docs).

### 4. Tracing
- Tracer provider and propagation.
- Span creation around critical path segments.
- External call spans and attributes.
- Trace correlation to logs (shared trace id).

### 5. Diagnosability & SLO Readiness
- Can you answer: "what is broken, where, how bad, since when?"
- Error messages include enough context (operation, dependency, ids).
- Explicit SLI candidates:
    - success rate, latency percentiles, queue lag, batch processing p99, error budgets.
- Runbooks / dashboards references (if present in repo).

---

## Method

Complete this checklist with evidence:

| # | Question | How to Verify |
|---|----------|---------------|
| 1 | Is logging structured? | Inspect logger init and output format |
| 2 | Are logs contextual and correlatable? | Look for request/trace/message IDs in log fields |
| 3 | Is sensitive data protected? | Search for secrets/PII logging and redaction logic |
| 4 | What metrics are exposed? | List metric registrations and exporters |
| 5 | Are correctness-critical metrics present (ADR-driven if any)? | Verify ADR-required metrics exist and are wired to critical paths |
| 6 | Are health checks implemented? | Find health endpoints/handlers |
| 7 | Is liveness separated from readiness? | Inspect endpoint semantics |
| 8 | Do readiness checks verify dependencies sensibly? | Check DB/queue/clickhouse checks or explicit decision to avoid them |
| 9 | Is tracing implemented end-to-end? | Look for tracer init + spans across call chain |
| 10 | Can incidents be diagnosed from signals? | Evaluate coverage of critical path, error context, dashboards/runbooks |

For each answer, provide:
- **File path(s)** inspected
- **Short excerpt** or line reference as evidence
- **Assessment:** COMPLETE / PARTIAL / MISSING

---

## Architectural Decisions Awareness

When findings relate to intentional decisions:
- Reference `ARCH_DECISIONS_PATH` (quote short excerpt + section heading)
- If ADRs define monitoring requirements (metrics/alerts thresholds), treat them as expected invariants:
    - Verify metrics exist and are emitted on the intended code paths
    - Verify alerts/runbooks exist if ADR demands them (or record as DRIFT/GAP)
- If missing: label as **DRIFT**, assess impact, propose corrective actions

---

## Output

Write exactly ONE report:

**REPORT_PATH:** `<target>/docs/rda/reports/07-observability.md`

### Report Structure (in this exact order)

# Observability Report

## 0. Run Metadata
- **Timestamp (UTC):** <YYYY-MM-DD HH:MM:SS UTC>
- **Git Commit:** <hash or GAP>
- **Change Detection:** <summary or GAP>
- **Inspected Files (Top Evidence):** <list 10-30 files>

## 1. Context Recap
- <2-5 bullets on observability components discovered>
- <Include ADR monitoring requirements if present>

## 2. Scope & Guardrails Confirmation
- Confirm: inspection limited to `TARGET_SERVICE_PATH`
- Confirm: no external code was opened
- List any EXTERNAL DEPs recorded (observability-related)

## 3. Evidence

### 3.1 Logging Analysis
| Aspect | Implementation | Evidence | Assessment |
|--------|----------------|----------|------------|
| Structured format | ... | ... | ... |
| Context propagation | ... | ... | ... |
| Log levels & gating | ... | ... | ... |
| Sensitive data | ... | ... | ... |
| Error logging policy | ... | ... | ... |

<Short excerpts only>

### 3.2 Metrics Inventory
| Metric Name | Type | Labels | Cardinality Risk | Purpose | Evidence |
|-------------|------|--------|------------------|---------|----------|
| ... | ... | ... | ... | ... | ... |

#### ADR-Required Metrics Verification (if any)
| Requirement (ADR excerpt ref) | Present? | Evidence | Notes |
|-------------------------------|----------|----------|------|
| ... | ... | ... | ... |

### 3.3 Health Checks
| Endpoint/Check | Liveness/Readiness | Dependencies Checked | Evidence | Assessment |
|----------------|---------------------|----------------------|----------|------------|
| ... | ... | ... | ... | ... |

### 3.4 Tracing Analysis
| Aspect | Implementation | Evidence | Assessment |
|--------|----------------|----------|------------|
| Trace provider | ... | ... | ... |
| Span coverage | ... | ... | ... |
| External call spans | ... | ... | ... |
| Log/trace correlation | ... | ... | ... |

### 3.5 SLI/SLO Readiness & Diagnosability
| Capability | Present? | Evidence | Impact |
|------------|----------|----------|--------|
| Candidate SLIs (success/latency/lag) | ... | ... | ... |
| Dashboards references | ... | ... | ... |
| Alerts references | ... | ... | ... |
| Runbooks references | ... | ... | ... |
| Incident drill-down possible? | ... | ... | ... |

## 4. Findings

### 4.1 Strengths
- <Bullets with evidence references>

### 4.2 Risks
- <Bullets with evidence, severity, impact>

### 4.3 Gaps
- <Items that could not be determined>

## 5. Action Items

### P0 (Critical)
- <Item> | Impact: <X> | Verify: <how>

### P1 (Important)
- <Item> | Impact: <X> | Verify: <how>

### P2 (Nice-to-have)
- <Item> | Impact: <X> | Verify: <how>

## 6. Delta vs Previous Run
- <If prior report existed: 3-10 bullets on differences>
- <If first run: "First run — no prior report to compare">
- <If no material changes: "No material changes detected" + list 3-5 re-checked items>

---

## Prioritization Rubric
- **P0:** Production risk / correctness / security / data loss / outage class
- **P1:** Significant reliability / operability / maintainability improvements
- **P2:** Nice-to-have / cleanup / future-proofing

---

## No Speculation Rule
All assertions MUST be tied to evidence (file path + excerpt/line reference). If evidence is unavailable, label as **INFERRED** or **GAP** and continue.

---

## Completion Criteria

Before submitting:
1. Run Metadata section is complete with timestamp and evidence file list
2. All 10 checklist questions answered with evidence + assessment
3. Metrics inventory includes types + labels + cardinality risk
4. ADR-required monitoring items verified (or marked DRIFT/GAP)
5. Liveness/readiness semantics are clearly described
6. Trace/log correlation assessed with evidence
7. Diagnosability section identifies concrete SLI/SLO candidates
8. No files outside `TARGET_SERVICE_PATH` were opened
9. Delta section present (even if first run)
10. Report written to correct path
