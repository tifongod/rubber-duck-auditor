# SKILL: rda-observability

# Observability

## Role & Objective

You are a **MAANG Principal Backend Engineer** auditing production readiness of a single service. Your task in this step is to evaluate **observability: logging, metrics, traces, health checks, SLI/SLO readiness, and diagnosability**.

---

## Shared Rules (MANDATORY)

Before doing anything else, read and follow:
- `skills/_shared/rda-common-rules.md`
- `skills/_shared/GUARDRAILS.md`
- `skills/_shared/REPORT_TEMPLATE.md`
- `skills/_shared/RISK_RUBRIC.md`
- `skills/_shared/PIPELINE.md`

If anything below conflicts with shared rules, shared rules win.

---

## Inputs

- `<target>` (optional): service root directory. If not provided, use `.`.
    - Recommended example for this service in a monorepo: `<target> = services/some-service`

### Mandatory Context (Read FIRST)
1. **ARCH_DECISIONS_PATH:** `<target>/docs/rda/ARCHITECTURAL_DECISIONS.md`
    - May include monitoring/alerting requirements (especially for correctness trade-offs such as idempotency).
    - If file is missing/unreadable: record as **GAP** and proceed.

### Prior Reports Required (Context)
2. **Inventory report (recommended):** `<target>/docs/rda/reports/00_service_inventory.md`
    - Use entrypoints and helper locations.
3. **Architecture report (recommended):** `<target>/docs/rda/reports/01_architecture_design.md`
    - Use cross-cutting concerns map and composition root location.

If any report is missing: record as **GAP** and scan logger/metrics/tracing/health packages inside `TARGET_SERVICE_PATH`.

### Fast-Path for Unchanged Target

If **both** of the following are true:
1. `git diff -- <target>` returns empty (no uncommitted changes)
2. `git status --porcelain -- <target>` returns empty (no untracked files)

Then you MAY take the fast-path:
- Read the previous report for this step (if exists)
- Confirm it has a "No material changes" delta section OR recent critical items
- Update **only** the Run Metadata:
  - New timestamp (UTC)
  - Same commit hash (if unchanged)
  - Change Detection: "No code changes detected via git diff/status"
- Add note in Delta section: "Fast-path: no code changes detected since last run (commit XXXXX). Prior findings re-validated without deep re-inspection."
- **EXCEPTION:** If prior report has **P0 items marked "not fixed"**, you MUST re-inspect those specific areas (no fast-path for P0s)

If the prior report does NOT exist, or has P0s, or you are uncertain: skip fast-path and perform full inspection.

**Rationale:** Unchanged code → unchanged findings. Fast-path saves ~70% of agent time on reruns for stable codebases.

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

## Tool Usage (MANDATORY)

**DO use these tools:**
- ✅ **Glob** for file patterns: `**/*.go`, `internal/*/`, `cmd/*/main.go`, `**/*.{yaml,yml}`
- ✅ **Grep** for content search:
  - `pattern="timeout"`, `--type go`, `--glob "*.yaml"`
  - Use `output_mode="files_with_matches"` for listing, `output_mode="content"` for excerpts
- ✅ **Read** for known file paths (always prefer Read over cat/head/tail)

**DO NOT use:**
- ❌ Bash commands: `ls -R`, `find .`, `grep`, `cat`, `head`, `tail`, `sed`, `awk`
- ❌ Glob without extension filter: `**/*` (too broad, lists everything)
- ❌ Reading files outside `<target>/` (verify path starts with target first)

**Rationale:** Dedicated tools provide better permission handling, are faster, and are auditable by the user.

See `skills/_shared/rda-common-rules.md` lines 22-29 for full details.

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

## Output

Write exactly ONE report:

**REPORT_PATH:** `<target>/docs/rda/reports/06_observability.md`

### Report Structure (in this exact order)

# Observability Report

## 0. Run Metadata
- **Timestamp (UTC):** <YYYY-MM-DD HH:MM:SS UTC>
- **Git Commit:** <hash or GAP>
- **Change Detection:** <summary or GAP>

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
