# SKILL: integrations

# External Integrations

## Role & Objective

You are a **MAANG Principal Backend Engineer** auditing production readiness of a single service. Your task in this step is to evaluate **external dependencies and integration correctness** — including timeouts, retries, idempotency, error mapping, connection management, and security.

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
    - Recommended example for this service in a monorepo: `<target> = services/some-service`

### Mandatory Context (Read FIRST)
1. **ARCH_DECISIONS_PATH:** `<target>/docs/rda/ARCHITECTURAL_DECISIONS.md`
    - Contains intentionally made architectural decisions with rationale.
    - Do NOT assume decisions are perfect or non-discussable.
    - You MUST consider these decisions during your audit.
    - If you believe a decision is risky/incorrect/outdated: explicitly call it out with evidence + reasoning and propose safer alternatives.
    - If file is missing/unreadable: record as **GAP** and proceed.

### Prior Reports Required (Context)
2. **Inventory report (recommended):** `<target>/docs/rda/reports/00_service_inventory.md`
    - Use external dependencies list and infrastructure requirements.
3. **Config report (recommended):** `<target>/docs/rda/reports/02_configuration_environment.md`
    - Use configuration for external systems (endpoints, credentials, timeouts).

If any report is missing: record as **GAP** and scan client/repository packages inside `TARGET_SERVICE_PATH`.

### Fast-Path for Unchanged Target

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

**Rationale:** Unchanged code → unchanged findings. Fast-path saves ~70% of agent time on reruns for stable codebases.

---

## What to Inspect

Use file locations from prior reports where possible. Within `TARGET_SERVICE_PATH` ONLY:

### 1. Integration Inventory (Must Be Evidence-Based)
- Identify all outbound integrations:
    - SDK clients (AWS, DB drivers, HTTP/gRPC clients)
    - Message queues / streams
    - Databases / caches
    - Internal services (HTTP/gRPC)
- For each, capture:
    - Where client is created (composition root / factory)
    - Where it is used (call sites)
    - Which config keys/env vars shape it (endpoints, timeouts, retries, credentials)

### 2. Timeouts & Cancellation
- Verify all external calls are bounded by timeouts and honor `context.Context`.
- For long-running operations (poll loops, batch inserts), verify timeouts vs operation semantics.

### 3. Retries & Backoff
- Identify retry mechanisms (SDK defaults, custom retry, middleware).
- Check:
    - max attempts
    - backoff strategy (exponential, jitter)
    - which errors are retried vs not retried (idempotent vs non-idempotent ops)

### 4. Idempotency & Deduplication at Integration Boundaries
- If service produces side effects externally:
    - verify idempotency keys / conditional writes / dedup tables
    - ensure retries do not duplicate effects
- For at-least-once inputs (queues), check handling of duplicates and partial failures.

### 5. Error Mapping & Observability (Integration-Facing)
- How are external errors translated to service-domain errors?
- Are errors classified (transient vs permanent)?
- Are retries / throttles / timeouts visible via logs/metrics?

### 6. Connection & Resource Management
- Pooling / reuse for DB and HTTP clients.
- Limits: max conns, max idle, per-host caps, queue poll concurrency, batch sizes.
- Reconnection behavior and failure modes.

### 7. Security
- TLS verification is enabled (no insecure skips).
- Credentials are loaded safely and are not logged.
- Least-privilege assumptions documented (if visible in config/docs).

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

See `skills/_shared/common-rules.md` lines 22-29 for full details.

---

## Method

Complete this checklist with evidence:

| # | Question | How to Verify |
|---|----------|---------------|
| 1 | What external systems are called? | Enumerate clients and their call sites inside the service |
| 2 | Are timeouts configured for all calls? | Find timeout settings and/or context timeouts around calls |
| 3 | What retry strategy is used? | Locate retry config or middleware; identify max attempts/backoff |
| 4 | How are transient errors handled? | Trace error handling on timeouts/throttling/network failures |
| 5 | Are connections pooled/reused? | Inspect client initialization and transport/pool configs |
| 6 | How is connection failure handled? | Find reconnection or fail-fast patterns and their safety |
| 7 | Is TLS properly configured? | Inspect TLS settings/cert verification behavior |
| 8 | How are credentials passed? | Trace credential injection; confirm no secret logging |
| 9 | Are errors mapped to domain errors? | Check wrapping/classification; avoid leaking provider internals |
| 10 | Is there circuit breaker logic? | Search for CB patterns; if absent, assess whether needed |

For each external system, evaluate:
- **Timeout:** Configured? Value reasonable?
- **Retry:** Strategy? Max attempts? Backoff?
- **Idempotency:** Safe under retries/duplicates?
- **Errors:** Classified/mapped? Logged? Metrics?
- **Security:** TLS? Credentials?

---

## Output

Write exactly ONE report:

**REPORT_PATH:** `<target>/docs/rda/reports/03_external_integrations.md`

### Report Structure (in this exact order)

# External Integrations Report

## 0. Run Metadata
- **Timestamp (UTC):** <YYYY-MM-DD HH:MM:SS UTC>
- **Git Commit:** <hash or GAP>

## 1. Context Recap
- <2-5 bullets on external systems from inventory/config reports>
- <Include relevant decision context from ARCH_DECISIONS_PATH>

## 2. Scope & Guardrails Confirmation
- Confirm: inspection limited to `TARGET_SERVICE_PATH`
- Confirm: no external code was opened
- List any EXTERNAL DEPs recorded

## 3. Evidence

### 3.1 Integration Inventory
| System | Client Location | Creation/Wiring | Call Sites | Config Location | Purpose |
|--------|-----------------|-----------------|-----------|-----------------|---------|
| ... | ... | ... | ... | ... | ... |

### 3.2 Per-System Analysis (Repeat per system)
| Aspect | Configuration | Evidence | Assessment |
|--------|---------------|----------|------------|
| Timeouts | ... | ... | ... |
| Retries / backoff | ... | ... | ... |
| Idempotency safety | ... | ... | ... |
| Error handling / mapping | ... | ... | ... |
| Connection mgmt | ... | ... | ... |
| Security (TLS/creds) | ... | ... | ... |

<Use short excerpts/line refs only>

### 3.3 Cross-Cutting Concerns
| Concern | Implementation | Evidence | Assessment |
|---------|----------------|----------|------------|
| Circuit breakers | ... | ... | ... |
| Error mapping | ... | ... | ... |
| Credential handling | ... | ... | ... |
| TLS verification | ... | ... | ... |
| Context propagation | ... | ... | ... |

## 4. Findings

### 4.1 Strengths
- <Bullet points with evidence references>

### 4.2 Risks
- <Bullet points with evidence, severity, impact>

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
1. Run Metadata section is complete with timestamp and file list
2. All 10 checklist questions answered with evidence
3. Each external system analyzed for timeout/retry/idempotency/error/security
4. Any retryable side effects explicitly assessed for idempotency safety
5. No files outside `TARGET_SERVICE_PATH` were opened
6. Delta section present (even if first run)
7. Report written to correct path
