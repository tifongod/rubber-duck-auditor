# SKILL: data

# Data Handling

## Role & Objective

You are a **MAANG Principal Backend Engineer** auditing production readiness of a single service. Your task in this step is to evaluate **data handling: consistency model, migrations, idempotency/deduplication, caching correctness, and SCD Type 2 implementation**.

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

## Inputs

- `<target>` (optional): service root directory. If not provided, use `.`.
    - Recommended example for this service in a monorepo: `<target> = services/some-service`

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
- Add note in Delta section: "Fast-path: no code changes detected since last run (commit XXXXX). Prior findings re-validated without deep re-inspection."
- **EXCEPTION:** If prior report has **P0 items marked "not fixed"**, you MUST re-inspect those specific areas (no fast-path for P0s)

If the prior report does NOT exist, or has P0s, or you are uncertain: skip fast-path and perform full inspection.

**Rationale:** Unchanged code → unchanged findings. Fast-path saves ~70% of agent time on reruns for stable codebases.

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

## Output

Write exactly ONE report:

**REPORT_PATH:** `<target>/docs/rda/reports/04_data_handling.md`

### Report Structure (in this exact order)

# Data Handling Report

## 0. Run Metadata
- **Timestamp (UTC):** <YYYY-MM-DD HH:MM:SS UTC>
- **Git Commit:** <hash or GAP>

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
