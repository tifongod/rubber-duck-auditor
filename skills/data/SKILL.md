# SKILL: data

# Data Handling

## Role & Objective

You are a **MAANG Principal Backend Engineer** auditing production readiness of a **single service**.  
Your task in this step is to evaluate **data handling**: data model & schema, consistency model, migrations & compatibility, idempotency/deduplication, caching correctness, and **any versioned/temporal data patterns (if present)**.

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
    - Recommended example for a monorepo: `<target> = services/some-service`

### Mandatory Context (Read FIRST)
1. **ARCH_DECISIONS_PATH:** `<target>/docs/rda/ARCHITECTURAL_DECISIONS.md`
    - **CRITICAL FOR THIS STEP:** may contain ADR(s) about data correctness, idempotency, migrations, or performance trade-offs.
    - You MUST read and understand relevant ADR sections before auditing idempotency and correctness.
    - Evaluate if the trade-offs are appropriate given production requirements.
    - If you disagree with the decision: label as **DECISION RISK** with evidence.
    - If file is missing/unreadable: record as **GAP** and proceed.

### Prior Reports Required (Context)
2. **Inventory report (recommended):** `<target>/docs/rda/reports/00_service_inventory.md`
    - Use database/migration locations, repositories, and entity/model pointers.
3. **Integrations report (recommended):** `<target>/docs/rda/reports/03_external_integrations.md`
    - Use data stores, brokers/queues, caches, and external persistence integration details and config points.

If any report is missing: record as **GAP** and scan repository/migration directories inside `TARGET_SERVICE_PATH`.

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
- Add note in Delta section:
    - "Fast-path: no code changes detected since last run (commit XXXXX). Prior findings re-validated without deep re-inspection."
- **EXCEPTION:** If prior report has **P0 items marked "not fixed"**, you MUST re-inspect those specific areas (no fast-path for P0s)

If the prior report does NOT exist, or has P0s, or you are uncertain: skip fast-path and perform full inspection.

**Rationale:** Unchanged code → unchanged findings. Fast-path saves agent time on reruns for stable codebases.

---

## Pattern Discovery (MANDATORY)

Before deep inspection, quickly determine which data patterns apply to this service.  
Classify each as: **PRESENT / ABSENT / UNCLEAR (GAP)** and keep **evidence** (paths + short excerpts).

Use narrow Glob/Grep inside `TARGET_SERVICE_PATH` to detect patterns such as:

### Versioned / Temporal Data
- Keywords: `valid_from`, `valid_to`, `effective_at`, `as_of`, `version`, `revision`, `prev_`, `history`, `audit`, `snapshot`, `bitemporal`, `tombstone`
- Semantics: historical correctness, time-travel queries, “current state” derived from history, snapshots, audit trails

### Idempotency / Deduplication
- Keywords: `idempot*`, `dedup*`, `event_id`, `message_id`, `inbox`, `outbox`, `processed_*`, `exactly_once`, `at_least_once`

### Migrations / Schema Management
- Keywords/dirs: `migrations`, `schema`, `ddl`, `sql`, `goose`, `migrate`, `flyway`, `liquibase`

### Caching
- Keywords: `redis`, `memcache`, `ristretto`, `cache`, `ttl`, `invalidate`, `etag`, `stale`

### Ordering / Streams
- Keywords: `offset`, `cursor`, `sequence`, `monotonic`, `partition`, `consumer group`, `ordering`, `out_of_order`

If a pattern is **ABSENT**, you MUST later state "Not detected" and include the discovery evidence (what you searched and where).

---

## What to Inspect

Use file locations from prior reports where possible. Within `TARGET_SERVICE_PATH` ONLY:

### 1. Data Model & Schema
- Domain entities/events/models packages.
- Store schema and migrations (wherever migrations live inside scope).
- If applicable: append-only facts vs state snapshots, audit/history tables, soft-delete/tombstones, temporal/version columns.
- Constraints & invariants: uniqueness, foreign keys, check constraints, application-level enforcement.

### 2. Idempotency & Deduplication
- Idempotency store/repository (if any).
- How `event_id` / `message_id` (or equivalents) prevent duplicates.
- Race-condition handling (per relevant ADR).
- Failure-mode behavior: what happens when idempotency store is degraded/unavailable?
- Retries: client retries, queue retries, internal retries, and replay/backfill behavior.

### 3. Versioning & Temporal Semantics (If Present)
- Any implementation of historical correctness / “time travel”:
    - history windows (valid_from/valid_to or equivalent)
    - audit logs (append-only), event logs
    - snapshot + delta patterns
    - soft-delete/tombstones + restore semantics
- How “current state” is derived:
    - materialized views, query-time selection, background compaction, reconciliation jobs, etc.
- Store/engine-specific correctness implications (if any).

If not detected: mark this section as N/A with discovery evidence.

### 4. Data Consistency Model
- Transaction boundaries and isolation (if any).
- Ordering assumptions (version chains vs timestamps, offsets, partitions).
- Eventual consistency and reconciliation/backfill hooks (if present).
- Deduplication at write time vs read time.
- Store/engine-specific behavior affecting correctness (upserts, compaction, read replicas, merge/dedup semantics).

### 5. Migrations & Compatibility
- Migration versioning scheme and tooling.
- Forward-only vs up/down support.
- Backward compatibility and safe rollout concerns.
- Data migrations vs schema migrations, online migration strategy if needed.

### 6. Caching Correctness (If Present)
- Any caches (in-memory/redis/local) involved in ingest/query path.
- Cache keying strategy, TTLs, and invalidation.
- Consistency risks: stale reads, write-through/write-behind, cache stampede, partial invalidation.
- If no caching exists: explicitly state "No caching detected" with evidence.

### 7. Domain-Specific Correctness Invariants (MANDATORY)
Identify any business invariants encoded in persistence or workflows (examples):
- “exactly one active X per Y”
- monotonic counters/balances
- no overlapping intervals/reservations
- uniqueness under concurrency and retries
  Verify how they are enforced (DB constraints vs application logic) and what breaks under retries/out-of-order.

---

## Tool Usage (MANDATORY)

**DO use these tools:**
- ✅ **Glob** for narrow patterns: `**/*.go`, `**/*.sql`, `**/*.{yaml,yml,json}`, `cmd/*/main.*`, `internal/**`
- ✅ **Grep** for content search (agent tooling equivalent, not shell):
    - Use `output_mode="files_with_matches"` for listing
    - Use `output_mode="content"` for short excerpts
- ✅ **Read** for known file paths (always prefer Read over shell tools)

**DO NOT use:**
- ❌ Bash commands: `ls -R`, `find .`, `grep`, `cat`, `head`, `tail`, `sed`, `awk`
- ❌ Glob without extension filter: `**/*` (too broad)
- ❌ Reading files outside `<target>/` (verify path starts with target first)

**Rationale:** Dedicated tools provide better permission handling, are faster, and are auditable by the user.  
See `skills/_shared/common-rules.md` for full details.

---

## Method

Complete this checklist with evidence:

| # | Question | How to Verify |
|---|----------|---------------|
| 1 | What is the data model and main entities? | Analyze models/entities + schema definitions |
| 2 | Which versioned/temporal pattern (if any) exists, and what correctness guarantees does it provide? | Use Pattern Discovery + inspect schema/repositories/jobs |
| 3 | How is idempotency enforced (if at all)? | Trace idempotency store usage, keys, and write/read gates |
| 4 | What are the main race-condition risks? | Compare to ADR(s), validate mitigations in code |
| 5 | What store/engine-specific semantics affect correctness? | Check upserts/compaction/transactions/replicas/merge behavior |
| 6 | What happens on duplicate events/requests? | Trace end-to-end handling (skip/merge/reject) |
| 7 | Are migrations versioned and applied safely? | Check naming/versioning/execution ordering and tooling |
| 8 | Are rollbacks safe/possible? | Determine down migrations or forward-only strategy and risks |
| 9 | How is ordering handled? | Check version chains, offsets, timestamps, partitioning, assumptions |
| 10 | Is there read-time dedup/reconciliation? | Look for “latest version” selection, repair jobs, compaction views |

For each answer, provide:
- **File path(s)** inspected
- **Short excerpt** (only what’s necessary)
- **Assessment:** CORRECT / CONCERN / RISK / GAP / N/A

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
- <2–5 bullets on data handling from inventory/integrations reports>
- <Summarize the most relevant ADR(s) and their relevance to this audit>

## 2. Scope & Guardrails Confirmation
- Confirm: inspection limited to `TARGET_SERVICE_PATH`
- Confirm: no external code was opened
- List any EXTERNAL DEPs recorded (if relevant)

## 3. Pattern Discovery Summary
| Pattern | Status (PRESENT/ABSENT/GAP) | Evidence (paths + brief note) |
|--------|------------------------------|-------------------------------|
| Versioned/temporal | ... | ... |
| Idempotency/dedup | ... | ... |
| Migrations/schema | ... | ... |
| Caching | ... | ... |
| Ordering/streams | ... | ... |

## 4. Evidence

### 4.1 Data Model Analysis
| Entity/Table | Location | Key Fields/Columns | Purpose |
|--------------|----------|--------------------|---------|
| ... | ... | ... | ... |

<Short excerpts only>

### 4.2 Versioning & Temporal Semantics (If Present)
| Aspect | Implementation | Evidence | Assessment |
|--------|----------------|----------|------------|
| Temporal columns / versioning | ... | ... | ... |
| Current-state derivation | ... | ... | ... |
| Window/interval logic (if any) | ... | ... | ... |
| Repair/reconciliation | ... | ... | ... |
| Store-specific implications | ... | ... | ... |

If ABSENT:
- State: "No explicit versioned/temporal pattern detected"
- Include discovery evidence
- Mark as N/A for deep analysis

### 4.3 Idempotency & Dedup Analysis (ADR-Aware)
| Aspect | Expected (per ADR, if any) | Actual | Aligned? | Assessment |
|--------|-----------------------------|--------|----------|------------|
| Pattern | ... | ... | ... | ... |
| Keys | ... | ... | ... | ... |
| Duplicate handling | ... | ... | ... | ... |
| Fail-safe | ... | ... | ... | ... |
| Ordering assumptions | ... | ... | ... | ... |

<Code excerpts (short)>

### 4.4 Consistency Model
| Aspect | Implementation | Evidence | Assessment |
|--------|----------------|----------|------------|
| Transaction boundaries | ... | ... | ... |
| Isolation / consistency | ... | ... | ... |
| Ordering guarantees | ... | ... | ... |
| Read-time dedup/reconciliation | ... | ... | ... |
| Backfill/repair hooks | ... | ... | ... |

### 4.5 Migration Analysis
| Migration | Version | Purpose | Rollback Safe? | Notes |
|-----------|---------|---------|----------------|------|
| ... | ... | ... | ... | ... |

### 4.6 Caching Correctness (If Applicable)
| Cache | Purpose | Invalidation Strategy | Consistency Risks | Evidence | Assessment |
|-------|---------|------------------------|------------------|----------|------------|
| ... | ... | ... | ... | ... | ... |

If no cache detected: state it explicitly with evidence.

### 4.7 Domain-Specific Invariants
| Invariant | Where enforced (DB/app/both) | Failure under retries/out-of-order? | Evidence | Assessment |
|----------|-------------------------------|------------------------------------|----------|------------|
| ... | ... | ... | ... | ... |

---

## 5. Findings

### 5.1 Strengths
- <Bullets with evidence references>

### 5.2 Risks
- <Bullets with evidence, severity, impact>
- <Specifically address ADR trade-offs that affect data correctness>

### 5.3 ADR Alignment Assessment
- **Implementation matches decision:** YES/NO/PARTIAL/GAP
- **Mitigations in place:** <list>
- **Mitigations missing:** <list>
- **Decision risk assessment:** <if you disagree with decision>

### 5.4 Gaps
- <Items that could not be determined>

## 6. Action Items

### P0 (Critical)
- <Item> | Impact: <X> | Verify: <how>

### P1 (Important)
- <Item> | Impact: <X> | Verify: <how>

### P2 (Nice-to-have)
- <Item> | Impact: <X> | Verify: <how>

## 7. Delta vs Previous Run
- <If prior report existed: 3–10 bullets on differences>
- <If first run: "First run — no prior report to compare">
- <If no material changes: "No material changes detected" + list 3–5 re-checked items>

---

## Completion Criteria

Before submitting:
1. Run Metadata section is complete with timestamp and commit (or GAP)
2. All 10 checklist questions answered with evidence + assessment
3. Relevant ADR(s) read, understood, and verified against implementation
4. Pattern Discovery completed and reflected in report (with evidence)
5. If a versioned/temporal pattern is PRESENT: analyzed with schema + code evidence; if ABSENT: marked N/A with evidence
6. Idempotency/dedup behavior documented with concrete failure-mode analysis
7. Caching (if any) assessed for correctness and invalidation; if absent, stated with evidence
8. Delta section present (even if first run)
9. No files outside `TARGET_SERVICE_PATH` were opened
10. Report written to correct path
