# SKILL: observability

# Observability

## Role & Objective

You are a **MAANG Principal Backend Engineer** auditing production readiness of a single service. Your task in this step is to evaluate **observability** in a universal, stack-agnostic way: logging, metrics, traces, health checks, SLI/SLO readiness, and overall diagnosability.

This skill must work for **any** service (API, worker, batch, cron, mixed) and **must not assume** specific tools (Prometheus, OpenTelemetry, ClickHouse, Kubernetes, etc.). If a technology is relevant, you must first **detect it within scope** and then evaluate it.

---

## Shared Rules (MANDATORY)

Before doing anything else, read and follow:
- `skills/_shared/common-rules.md`
- `skills/_shared/GUARDRAILS.md`
- `skills/_shared/REPORT_TEMPLATE.md`
- `skills/_shared/RISK_RUBRIC.md`
- `skills/_shared/PIPELINE.md`

If anything below conflicts with shared rules, shared rules win.

---

## ⚠️ Concurrent Execution Warning

**DO NOT run this skill concurrently with itself.** Running the same step multiple times in parallel will cause last-write-wins behavior where the second run silently overwrites the first run's report with no conflict detection. This results in silent data loss.

If you need to re-run this step, wait for the current execution to complete before starting a new one.

(This warning does not apply to pipeline-orchestrated runs, which use wave-based sequencing to prevent concurrent writes.)

---

## Inputs

- `<target>` (optional): service root directory. If not provided, use `.`.

### Mandatory Context (Read FIRST)
1. **ARCH_DECISIONS_PATH:** `<target>/docs/rda/ARCHITECTURAL_DECISIONS.md`
    - May include *explicit monitoring/alerting requirements*, SLO targets, or mitigations that require observability.
    - If file is missing/unreadable: record as **GAP** and proceed.

### Prior Reports (Context; Read SECOND)
2. **Inventory report (recommended):** `<target>/docs/rda/reports/00_service_inventory.md`
    - Use runtime model, entrypoints, critical paths, dependency list.
3. **Architecture report (recommended):** `<target>/docs/rda/reports/01_architecture_design.md`
    - Use cross-cutting concerns map and composition root location.

If any report is missing: record as **GAP** and proceed by scanning observability-related packages inside `TARGET_SERVICE_PATH`.

---

## Fast-Path for Unchanged Target

If **both** of the following are true:
1. `git diff -- "${target}"` returns empty (no uncommitted changes)
2. `git status --porcelain -- "${target}"` returns empty (no untracked files)

Then you MAY take the fast-path:
- Read the previous report for this step (if exists)
- Confirm it has a "No material changes" delta section OR recent critical items
- Update **only** the Run Metadata:
    - New timestamp (UTC)
    - Same commit hash (if unchanged)
- Add note in Delta section: "Fast-path: no code changes detected since last run (commit XXXXX). Prior findings re-validated without deep re-inspection."
- **EXCEPTION:** If prior report has **P0 items marked "not fixed"**, you MUST re-inspect those specific areas (no fast-path for P0s)

If the prior report does NOT exist, or has P0s, or you are uncertain: skip fast-path and perform full inspection.

---

## Pattern Discovery (MANDATORY)

Before deep inspection, identify what observability surfaces apply to this service, with evidence:

- Runtime model: API server / worker / cron / batch / mixed
- Interfaces: HTTP / gRPC / messaging / CLI / scheduled jobs
- Signal systems detected (examples, not assumptions):
    - logging library/framework, log format (structured vs plain)
    - metrics system (Prometheus, StatsD, OTEL metrics, custom)
    - tracing system (OpenTelemetry, vendor tracer, none)
    - health endpoints/probes (HTTP endpoints, gRPC health, internal checks)
    - dashboards/runbooks/alert configs in-repo (if any)

If something is absent, explicitly mark it as **ABSENT** with evidence (paths searched + no matches).

---

## What to Inspect (Universal)

Use file locations from prior reports where possible. Within `TARGET_SERVICE_PATH` ONLY.

### 1) Logging (universal)
- Logging initialization/config (format, sinks, sampling, log level control).
- Structured vs unstructured output.
- Log levels and expectations (INFO/WARN/ERROR/DEBUG) and gating.
- Context propagation (request id / trace id / correlation id / job id / message id / tenant id) as applicable.
- Sensitive data handling (secrets/PII redaction, allowlist/denylist, payload logging policy).
- Error logging policy:
    - double-logging risks (log + return)
    - stack traces policy
    - wrap vs log boundaries
    - noisy vs actionable errors

### 2) Metrics (universal)
- Metrics registration and exposure (scrape endpoint, exporter, push gateway, etc.).
- Metric types (counter/gauge/histogram/summary) and correctness of use.
- Labels/tags and cardinality risks (tenant/user/request ids must not explode).
- Coverage across critical paths (adapt to runtime model):
    - request/handler ingress (API) OR poll loop (worker)
    - per dependency calls: latency + errors
    - retries, throttling, backoff, partial failures
    - queue lag/message age (if applicable)
    - batch sizes and processing durations (if batching exists)
    - saturation signals: concurrency, in-flight, memory, goroutines/threads (if measurable)
- ADR-specified metrics: treat as contractual; verify presence + wiring.

### 3) Health Checks (universal)
- Liveness vs readiness semantics (or equivalent).
- Dependency checks (DB/queue/cache/external services) or explicit decision to avoid them.
- Degraded mode: what counts as “ready enough”.
- Startup behavior: warmups, migrations, caches, connection pools (and how health reflects them).

### 4) Tracing (universal)
- Tracer/provider initialization and propagation.
- Span coverage around critical path segments.
- External call spans with attributes (dependency name, status, latency, error).
- Trace correlation to logs (trace id in logs or log correlation metadata).
- Sampling strategy and its effect on incident investigation.

### 5) Diagnosability & SLI/SLO Readiness (universal)
Evaluate whether signals can answer:
- “What is broken?”
- “Where is it broken?”
- “How bad is it?”
- “Since when?”
- “Is it getting worse or recovering?”

Identify explicit SLI candidates based on the service:
- API: success rate, latency percentiles, error budget burn
- Worker: throughput, lag/age, processing latency, DLQ rate, retry rate
- Any: dependency availability/latency, saturation, cost drivers (if measurable)

Also search for in-repo references:
- dashboards, alerts, runbooks, on-call docs, incident guides

---

## Tool Usage (MANDATORY)

**DO use these tools:**
- ✅ **Glob** for file patterns: `**/*.go`, `internal/*/`, `cmd/*/main.go`, `**/*.{yaml,yml,json,md}`
- ✅ **Grep** for content search:
    - Use `output_mode="files_with_matches"` for listing, `output_mode="content"` for excerpts
    - Suggested patterns (adapt as needed): `logger`, `log`, `zap`, `slog`, `zerolog`, `prometheus`, `metrics`, `otel`, `opentelemetry`, `trace`, `span`, `health`, `ready`, `live`, `probe`, `pprof`
- ✅ **Read** for known file paths (always prefer Read over cat/head/tail)

**DO NOT use:**
- ❌ Bash commands: `ls -R`, `find .`, `grep`, `cat`, `head`, `tail`, `sed`, `awk`
- ❌ Glob without extension filter: `**/*`
- ❌ Reading files outside `<target>/`

---

## Method

Complete this checklist with evidence.

For each answer, provide:
- **File path(s)** inspected
- **Short excerpt** or **line reference** as evidence
- **Assessment:** COMPLETE / PARTIAL / MISSING / N/A

Checklist:

| # | Question | How to Verify |
|---|----------|---------------|
| 1 | Is logging structured and consistent? | Inspect logger init + format + key fields |
| 2 | Are logs contextual and correlatable? | Check correlation IDs / trace IDs / job/message IDs in logs |
| 3 | Is sensitive data protected in logs? | Search for payload logging + redaction/scrubbing logic |
| 4 | What metrics are exposed and how? | Locate exporters/endpoints/registration |
| 5 | Are correctness/incident-critical metrics present (ADR-driven if any)? | Verify ADR requirements + wiring to critical paths |
| 6 | Are health checks implemented? | Find health endpoints/handlers |
| 7 | Is liveness separated from readiness (or equivalent)? | Inspect semantics and failure behavior |
| 8 | Are dependency readiness checks sensible (or explicitly avoided)? | Check dependency probes and rationale |
| 9 | Is tracing implemented end-to-end where it matters? | Tracer init + spans across call chain + external calls |
| 10 | Can incidents be diagnosed from available signals? | Evaluate coverage + context + dashboards/runbooks |

---

## Output

Write exactly ONE report:

**REPORT_PATH:** `<target>/docs/rda/reports/06_observability.md`

### Report Structure (in this exact order)

# Observability Report

## 0. Run Metadata
- **Timestamp (UTC):** <YYYY-MM-DD HH:MM:SS UTC>
- **Git Commit:** <hash or GAP>

## 1. Context Recap
- 3–7 bullets summarizing discovered observability components (logging/metrics/tracing/health)
- Include ADR monitoring requirements if present (or GAP)

## 2. Scope & Guardrails Confirmation
- Confirm: inspection limited to `TARGET_SERVICE_PATH`
- Confirm: no external code was opened
- List any EXTERNAL DEPs recorded (observability-related)

## 3. Pattern Discovery Summary
| Surface | Status (PRESENT/ABSENT/GAP) | Evidence (paths + brief note) |
|---------|------------------------------|-------------------------------|
| Logging | ... | ... |
| Metrics | ... | ... |
| Tracing | ... | ... |
| Health checks | ... | ... |
| Dashboards/alerts/runbooks in repo | ... | ... |

## 4. Evidence

### 4.1 Logging Analysis
| Aspect | Implementation | Evidence | Assessment |
|--------|----------------|----------|------------|
| Structured format | ... | ... | ... |
| Context propagation | ... | ... | ... |
| Level control & gating | ... | ... | ... |
| Sensitive data policy | ... | ... | ... |
| Error logging policy | ... | ... | ... |

### 4.2 Metrics Inventory
| Metric Name | Type | Labels | Cardinality Risk | Purpose | Evidence |
|-------------|------|--------|------------------|---------|----------|
| ... | ... | ... | ... | ... | ... |

#### ADR-Required Metrics Verification (if any)
| Requirement (ADR ref) | Present? | Evidence | Notes |
|-----------------------|----------|----------|------|
| ... | ... | ... | ... |

### 4.3 Health Checks
| Endpoint/Check | Liveness/Readiness | Dependencies Checked | Evidence | Assessment |
|----------------|---------------------|----------------------|----------|------------|
| ... | ... | ... | ... | ... |

### 4.4 Tracing Analysis
| Aspect | Implementation | Evidence | Assessment |
|--------|----------------|----------|------------|
| Trace provider | ... | ... | ... |
| Span coverage | ... | ... | ... |
| External call spans | ... | ... | ... |
| Log/trace correlation | ... | ... | ... |
| Sampling strategy | ... | ... | ... |

### 4.5 SLI/SLO Readiness & Diagnosability
| Capability | Present? | Evidence | Impact |
|------------|----------|----------|--------|
| Candidate SLIs identified | ... | ... | ... |
| “What/where/how bad/since when” answerable | ... | ... | ... |
| Dashboards references | ... | ... | ... |
| Alerts references | ... | ... | ... |
| Runbooks references | ... | ... | ... |

## 5. Findings

### 5.1 Strengths
- Bullets with evidence references

### 5.2 Risks
All risks MUST be written using `skills/_shared/RISK_RUBRIC.md` required fields (Severity S0–S3, Priority P0–P2, L/B/D, Evidence, Recommendation, Verification, etc.).  
If you cannot support a required field: mark it as GAP.

### 5.3 Gaps
- Items that could not be determined within scope

## 6. Action Items
Use the one-line format:
`<Action>` | **Priority:** P0/P1/P2 | **Impact:** ... | **Verify:** ...

## 7. Delta vs Previous Run
- If prior report existed: 3–10 bullets on differences
- If first run: "First run — no prior report to compare"
- If no material changes: "No material changes detected" + 3–5 re-checked items

---

## Completion Criteria

Before submitting:
1. Run Metadata complete
2. All 10 checklist questions answered with evidence + assessment
3. Pattern discovery table completed (PRESENT/ABSENT/GAP with evidence)
4. Metrics inventory includes types + labels + cardinality assessment
5. ADR-required monitoring items verified (or DRIFT/GAP)
6. Health semantics (liveness vs readiness or equivalent) described with evidence
7. Trace/log correlation assessed with evidence
8. Diagnosability section proposes concrete SLIs
9. Risks and Action Items comply with `skills/_shared/RISK_RUBRIC.md`
10. No files outside `TARGET_SERVICE_PATH` were opened
11. Report written to correct path
