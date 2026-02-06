# SKILL: reliability

# Reliability & Failure Handling

## Role & Objective

You are a **MAANG Principal Backend Engineer** auditing production readiness of a single service. Your task in this step is to evaluate **reliability: failure handling, backpressure/concurrency limits, graceful shutdown, queue semantics, and resilience patterns**.

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
    - May contain decisions about at-least-once delivery, visibility timeout handling, and failure semantics.
    - Relevant to queue semantics and shutdown/failure handling analysis.
    - If file is missing/unreadable: record as **GAP** and proceed.

### Prior Reports Required (Context)
2. **Inventory report (recommended):** `<target>/docs/rda/reports/00_service_inventory.md`
    - Use runtime model and queue components.
3. **Integrations report (recommended):** `<target>/docs/rda/reports/03_external_integrations.md`
    - Use queue/client integration details (timeouts/retries).
4. **Data report (recommended):** `<target>/docs/rda/reports/04_data_handling.md`
    - Use idempotency and consistency model conclusions.

If any report is missing: record as **GAP** and scan queue/app/middleware packages inside `TARGET_SERVICE_PATH`.

---

## What to Inspect

Use file locations from prior reports where possible. Within `TARGET_SERVICE_PATH` ONLY:

### 1. Queue/Worker Reliability (If Applicable)
- Consumer/poller/batcher implementation.
- Message acknowledgment pattern (delete/ack timing).
- Visibility timeout extension strategy (if any).
- Poison message handling and DLQ expectations (as visible in config/code).

If service is an API server (not a worker), focus on request handling reliability (timeouts, middleware, concurrency) and mark queue parts as GAP/Not Applicable with evidence.

### 2. Failure Handling
- Error recovery in processing pipeline.
- Panic recovery.
- Partial failure handling in batches (per-message vs per-batch semantics).
- Retry loops vs fail-fast boundaries.

### 3. Backpressure & Concurrency
- Concurrency limits on workers / request handlers.
- Batch size limits and memory bounds.
- Rate limiting / token bucket / bounded channels.
- Any unbounded goroutine/channel patterns.

### 4. Graceful Shutdown
- App lifecycle and signal handling (SIGTERM/SIGINT).
- In-flight message handling on shutdown.
- Connection draining and close order.
- Shutdown timeouts and safety (avoid message loss or duplicate storms).

### 5. Delivery Semantics
- At-least-once behavior (expected redelivery on failure).
- What happens on processing timeout.
- Where idempotency is relied upon vs prevented (tie back to data report/ADR).

### 6. Resilience Patterns
- Retry with jitter/backoff and caps.
- Timeouts everywhere.
- Circuit breaker / bulkhead patterns (if present).
- Load shedding / backoff on downstream degradation.

---

## Method

Complete this checklist with evidence:

| # | Question | How to Verify |
|---|----------|---------------|
| 1 | How are messages/requests processed? | Trace poller/consumer or handler call chain |
| 2 | What is the batch or concurrency model? | Identify worker pool, batcher, or request concurrency limits |
| 3 | How is visibility timeout / deadlines managed? | Verify visibility extension or bounded processing times |
| 4 | How are processing failures handled? | Trace error paths and retry/ack behavior |
| 5 | Is there panic recovery? | Locate recovery middleware/handlers |
| 6 | What are concurrency limits? | Find explicit limits, bounded channels, semaphores |
| 7 | How is backpressure applied? | Look for rate limits, queue depth controls, batch limits |
| 8 | How is graceful shutdown handled? | Analyze shutdown sequence and timeouts |
| 9 | Are in-flight items handled on shutdown? | Confirm behavior for in-flight messages/requests |
| 10 | Is there DLQ/poison handling? | Look for DLQ logic or poison classification handling |

For each answer, provide:
- **File path(s)** inspected
- **Short excerpt** or line reference as evidence
- **Assessment:** RELIABLE / CONCERN / RISK

---

## Architectural Decisions Awareness

When findings relate to intentional decisions:
- Reference `ARCH_DECISIONS_PATH` (quote short excerpt + section heading)
- Verify queue semantics and shutdown behavior align with stated philosophy (e.g., at-least-once + idempotency reliance)
- If you see drift (code/config contradicts decisions): label as **DRIFT**, assess impact, propose corrective actions
- If you think the decision itself is poor: label as **DECISION RISK**, justify with evidence + production-readiness reasoning, propose safer alternatives

---

## Output

Write exactly ONE report:

**REPORT_PATH:** `<target>/docs/rda/reports/05_reliability_failure_handling.md`

### Report Structure (in this exact order)

# Reliability & Failure Handling Report

## 0. Run Metadata
- **Timestamp (UTC):** <YYYY-MM-DD HH:MM:SS UTC>
- **Git Commit:** <hash or GAP>
- **Change Detection:** <summary or GAP>

## 1. Context Recap
- <2-5 bullets on runtime model and reliability-critical paths from inventory report>
- <Include relevant ADR context from ARCH_DECISIONS_PATH>

## 2. Scope & Guardrails Confirmation
- Confirm: inspection limited to `TARGET_SERVICE_PATH`
- Confirm: no external code was opened
- List any EXTERNAL DEPs recorded (reliability-relevant)

## 3. Evidence

### 3.1 Processing Architecture
Provide an evidence-based flow diagram of the critical path(s) (worker or API). Example shape:

[SQS → Poller → Batcher → Consumer → Repos] or [HTTP → Middleware → Handler → Usecase → Repos]

| Component | Location | Purpose | Key Config | Assessment |
|-----------|----------|---------|------------|------------|
| ... | ... | ... | ... | ... |

### 3.2 Failure Handling Matrix
| Failure Type | Handling | Recovery / Next State | Evidence | Assessment |
|--------------|----------|------------------------|----------|------------|
| Processing error | ... | ... | ... | ... |
| Panic | ... | ... | ... | ... |
| Timeout/deadline | ... | ... | ... | ... |
| Partial batch failure | ... | ... | ... | ... |
| Downstream outage | ... | ... | ... | ... |

### 3.3 Concurrency & Backpressure
| Limit Type | Value | Configurable? | Evidence | Assessment |
|------------|-------|---------------|----------|------------|
| Worker concurrency | ... | ... | ... | ... |
| Batch size | ... | ... | ... | ... |
| Rate limit / bounded queue | ... | ... | ... | ... |
| Unbounded patterns | ... | ... | ... | ... |

### 3.4 Graceful Shutdown Analysis
| Phase | Implementation | Timeout | Evidence | Assessment |
|-------|----------------|---------|----------|------------|
| Signal capture | ... | ... | ... | ... |
| Stop accepting work | ... | ... | ... | ... |
| Drain in-flight | ... | ... | ... | ... |
| Close clients | ... | ... | ... | ... |

### 3.5 Delivery Semantics (At-Least-Once / Deadlines)
| Aspect | Implementation | Aligned with ADR? | Evidence | Assessment |
|--------|----------------|-------------------|----------|------------|
| Ack/delete timing | ... | ... | ... | ... |
| Redelivery behavior | ... | ... | ... | ... |
| Visibility/deadline extension | ... | ... | ... | ... |
| Poison handling / DLQ | ... | ... | ... | ... |

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
All assertions MUST be tied to evidence (file path + excerpt/line reference). If evidence is unavailable, label as **GAP** and continue.

---

## Completion Criteria

Before submitting:
1. Run Metadata section is complete with timestamp and evidence file list
2. All 10 checklist questions answered with evidence + assessment
3. Critical-path flow diagram is evidence-based (not generic)
4. Failure handling matrix is complete and includes partial-failure semantics
5. Concurrency/backpressure limits identified (or explicit GAP)
6. Graceful shutdown and in-flight handling analyzed with evidence
7. Delivery semantics assessed and tied back to relevant ADR(s)
8. No files outside `TARGET_SERVICE_PATH` were opened
9. Delta section present (even if first run)
10. Report written to correct path
