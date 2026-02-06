# SKILL: data

# Data Handling

## Role & Objective

You are a **MAANG Principal Backend Engineer** auditing production readiness of a single service. Your task in this step is to evaluate **data handling: consistency model, migrations, idempotency/deduplication, caching correctness, and SCD Type 2 implementation**.

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
- You MUST draft findings based on code/schema inspection in this run, then reconcile with prior report(s) and produce a proper delta.

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
    - **CRITICAL FOR THIS STEP:** may contain ADR(s) about idempotency and data correctness trade-offs.
    - You MUST read and understand relevant ADR sections before auditing idempotency/SCD2.
    - Evaluate if the trade-offs are appropriate given production requirements.
    - If you disagree with the decision: label as **DECISION RISK** with evidence.
    - If file is missing/unreadable: record as **GAP** and proceed.

### Prior Reports Required (Context)
2. **Inventory report (recommended):** `<target>/docs/rda/reports/00_service_inventory.md`
    - Use database migrations and repository locations.
3. **Integrations report (recommended):** `<target>/docs/rda/reports/03_external_integrations.md`
    - Use DynamoDB / ClickHouse integration details and config points.

If any report is missing: record as **GAP** and scan repository/migration directories inside `TARGET_SERVICE_PATH`.

---

## What to Inspect

Use file locations from prior reports where possible. Within `TARGET_SERVICE_PATH` ONLY:

### 1. Data Model & Schema
- Domain entities/events/models packages.
- ClickHouse schema and migrations (wherever migrations live inside scope).
- Schema design for SCD Type 2 and fact vs state tables (if applicable).

### 2. Idempotency & Deduplication
- Idempotency store/repository (if any).
- How `event_id` (or equivalent) is used to prevent duplicates.
- Race-condition handling (per relevant ADR).
- Failure mode behavior (what happens when idempotency store is down?).

### 3. SCD Type 2 Implementation
- SCD2 repository logic (close/open windows, prev_version, valid_from/valid_to).
- Engine choices (e.g., ReplacingMergeTree) and dedup keys.
- Query-time semantics (e.g., `FINAL`) and correctness implications.

### 4. Data Consistency Model
- Transaction boundaries (if any exist).
- Ordering assumptions (monotonic version vs timestamp).
- Eventual consistency and reconciliation/backfill hooks (if present).
- Deduplication at query time vs ingestion time.

### 5. Migrations & Compatibility
- Migration versioning scheme and tooling.
- Forward-only vs up/down support.
- Backward compatibility and safe rollout concerns.

### 6. Caching Correctness (If Present)
- Any caches (in-memory/redis/local) involved in ingest/query path.
- Cache invalidation strategy and consistency risks.
- If no caching exists: explicitly state "No caching detected" with evidence.

---

## Method

Complete this checklist with evidence:

| # | Question | How to Verify |
|---|----------|---------------|
| 1 | What is the data model? | Analyze model/entity structs and schema definitions |
| 2 | How is SCD2 implemented? | Check repository logic and schema versioning columns |
| 3 | How is idempotency enforced? | Analyze idempotency repository/store usage |
| 4 | What are the race condition risks? | Compare to relevant ADR(s), verify mitigations |
| 5 | How does ReplacingMergeTree (or equivalent) work here? | Check ORDER BY, version column, dedup semantics |
| 6 | What happens on duplicate events? | Trace duplicate handling path end-to-end |
| 7 | Are migrations versioned correctly? | Check file naming/versioning and execution order |
| 8 | Are rollbacks safe/possible? | Determine if down migrations exist and are safe; if forward-only, assess risks |
| 9 | How is ordering handled? | Check for version chain / prev_version / timestamps use |
| 10 | Is there query-time dedup? | Look for `FINAL` usage or equivalent dedup logic |

For each answer, provide:
- **File path(s)** inspected
- **Short excerpt** or line reference as evidence
- **Assessment:** CORRECT / CONCERN / RISK

---

## Architectural Decisions Awareness (CRITICAL FOR THIS STEP)

If an ADR explicitly governs idempotency/dedup/SCD2 (e.g., "non-atomic idempotency" trade-off), do the following:

1. **Read and summarize the ADR section** relevant to data correctness.
2. **Verify implementation matches the decision** (pattern, fail-safe behavior, timeouts/visibility handling, dedup assumptions).
3. **Verify mitigations are in place** (schema keys, merge engine choice, window closure strategy, operational safeguards).
4. **Evaluate trade-off validity** for production readiness:
    - Is "data loss worse than duplicates" appropriate here?
    - Are monitoring/alerting and reconciliation hooks present if duplicates/drift can happen?
5. **If you see drift or disagree**:
    - Label as **DRIFT** or **DECISION RISK**
    - Provide specific evidence
    - Propose safer alternatives with justification

---

## Output

Write exactly ONE report:

**REPORT_PATH:** `<target>/docs/rda/reports/04_data_handling.md`

### Report Structure (in this exact order)

# Data Handling Report

## 0. Run Metadata
- **Timestamp (UTC):** <YYYY-MM-DD HH:MM:SS UTC>
- **Git Commit:** <hash or GAP>
- **Change Detection:** <summary or GAP>

## 1. Context Recap
- <2-5 bullets on data handling from inventory/integrations reports>
- <Summarize the most relevant ADR(s) and their relevance to this audit>

## 2. Scope & Guardrails Confirmation
- Confirm: inspection limited to `TARGET_SERVICE_PATH`
- Confirm: no external code was opened
- List any EXTERNAL DEPs recorded (if relevant)

## 3. Evidence

### 3.1 Data Model Analysis
| Entity/Table | Location | Key Fields/Columns | Purpose |
|--------------|----------|--------------------|---------|
| ... | ... | ... | ... |

<Short excerpts only>

### 3.2 SCD Type 2 Implementation
| Aspect | Implementation | Evidence | Assessment |
|--------|----------------|----------|------------|
| Version column | ... | ... | ... |
| valid_from/to | ... | ... | ... |
| ORDER BY key | ... | ... | ... |
| Merge engine | ... | ... | ... |
| Window close logic | ... | ... | ... |

<Migration SQL excerpts (short)>

### 3.3 Idempotency & Dedup Analysis (ADR-Aware)
| Aspect | Expected (per ADR, if any) | Actual | Aligned? | Assessment |
|--------|-----------------------------|--------|----------|------------|
| Pattern | ... | ... | ... | ... |
| Fail-safe | ... | ... | ... | ... |
| Duplicate handling | ... | ... | ... | ... |
| Ordering assumptions | ... | ... | ... | ... |

<Code excerpts showing implementation (short)>

### 3.4 Consistency Model
| Aspect | Implementation | Evidence | Assessment |
|--------|----------------|----------|------------|
| Transaction boundaries | ... | ... | ... |
| Ordering guarantees | ... | ... | ... |
| Query-time dedup | ... | ... | ... |
| Reconciliation/backfill hooks | ... | ... | ... |

### 3.5 Migration Analysis
| Migration | Version | Purpose | Rollback Safe? | Notes |
|-----------|---------|---------|----------------|------|
| ... | ... | ... | ... | ... |

### 3.6 Caching Correctness (If Applicable)
| Cache | Purpose | Invalidation Strategy | Consistency Risks | Evidence | Assessment |
|-------|---------|------------------------|------------------|----------|------------|
| ... | ... | ... | ... | ... | ... |

If no cache detected: state it explicitly with evidence.

## 4. Findings

### 4.1 Strengths
- <Bullets with evidence references>

### 4.2 Risks
- <Bullets with evidence, severity, impact>
- <Specifically address ADR trade-offs that affect data correctness>

### 4.3 ADR Alignment Assessment
- **Implementation matches decision:** YES/NO/PARTIAL/GAP
- **Mitigations in place:** <list>
- **Mitigations missing:** <list>
- **Decision risk assessment:** <if you disagree with decision>

### 4.4 Gaps
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
1. Run Metadata section is complete with timestamp and file list
2. All 10 checklist questions answered with evidence + assessment
3. Relevant ADR(s) read, understood, and implementation verified against them
4. SCD2 implementation analyzed with schema evidence
5. Idempotency/dedup behavior documented with concrete failure-mode analysis
6. Caching (if any) assessed for correctness and invalidation
7. No files outside `TARGET_SERVICE_PATH` were opened
8. Delta section present (even if first run)
9. Report written to correct path
