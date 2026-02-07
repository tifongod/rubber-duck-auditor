# SKILL: performance

# Performance & Capacity

## Role & Objective

You are a **MAANG Principal Backend Engineer** auditing production readiness of a single service. Your task in this step is to evaluate **performance and capacity readiness** in a stack-agnostic way: latency/throughput critical paths, batching and concurrency, memory/CPU hotspots, queue lag dynamics (if applicable), **data-store query/write patterns (if applicable)**, and capacity planning signals (limits, backpressure, load testability).

**Do not assume** any specific datastore/queue/infra (e.g., ClickHouse, Kafka, Kubernetes). If a technology matters, you must first **detect it within scope** and then evaluate it.

This step must produce a practical plan: what to measure, what to optimize first, and how to validate improvements.

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
1) `ARCH_DECISIONS_PATH = <target>/docs/rda/ARCHITECTURAL_DECISIONS.md`
- Use as context for intentional trade-offs that affect performance (batching, retries, at-least-once semantics, backpressure strategy, data-store engine choices, query patterns).
- Do NOT treat as unquestionable truth.
- If missing/unreadable: record as **GAP** and proceed.

### Prior Reports (Read SECOND)
2) Read existing reports in `<target>/docs/rda/reports/` (if present), especially:
- service inventory / architecture (critical paths, entrypoints)
- integrations (queues/brokers, databases, caches, external APIs)
- data handling (versioning/temporal patterns, dedup, migrations)
- reliability & observability (concurrency/backpressure, metrics)
  If missing: record as GAP and proceed.

---

## Fast-Path for Unchanged Target

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

## Pattern Discovery (MANDATORY)

Before deep inspection, quickly identify which performance surfaces apply to this service and keep evidence (paths + short excerpts):
- Runtime model: API server / worker / cron / mixed
- External IO: databases, caches, queues/brokers, object storage, external HTTP/gRPC
- Data-store type(s), if any: relational, key-value, document, search, OLAP/warehouse, etc.
- Hot-loop signals: decoding/encoding, mapping/transformations, logging/metrics in loops
- Backpressure controls: bounded queues, rate limiting, circuit breakers, concurrency limits

If something is ABSENT (e.g., no queue usage, no cache usage), explicitly state "Not detected" with discovery evidence later.

---

## What to Inspect (PERF/CAPACITY-SCOPED)

Within `TARGET_SERVICE_PATH` ONLY.

### 1) Identify Critical Paths (Latency vs Throughput)
- For API servers: request path → handlers → usecases → repositories.
- For workers/consumers: poll → batch → process → write → ack.
- Document what is on the hot path and what is best-effort.

### 2) Concurrency, Batching, and Backpressure
- Worker pool sizes, goroutine fan-out, channel buffering.
- Batch size controls and flush timers.
- Rate limiting, polling intervals, adaptive backoff.
- Any unbounded growth risks (queues, slices, maps, caches).

### 3) Timeouts and Budgeting
- End-to-end time budgets per operation (e.g., request deadlines; queue visibility vs batch processing time).
- Per-call timeouts to external systems, retries, and their worst-case amplification.
- Head-of-line blocking risks (single writer, global locks, shared clients).

### 4) Data Store Footprint (If Used)
Evaluate performance characteristics based on what stores are detected.
- Write patterns: batch sizes, write frequency, upserts, transactions, retries.
- Read/query patterns: scans vs indexed lookups, pagination, sorting/grouping, wide queries.
- Schema/index decisions affecting performance (indexes, partitions, clustering keys, TTL).
- Store-specific semantics that can cause heavy work:
  - compaction/merges, read amplification, write amplification
  - forced-consistency reads / forced-merge reads (store-specific; only if detected)
  - dedup-at-read vs dedup-at-write
- Connection pooling, concurrency, and backpressure around the store clients.

### 5) Memory/CPU Hotspots (From Code Signals)
- Large allocations, per-message JSON/proto decode, repeated conversions.
- Use of reflection, regex, string concatenation in hot loops.
- Potential N+1 patterns in ingestion or query fan-out.
- Logging/metrics label cardinality overhead.

### 6) Capacity Signals & Operational Readiness
- Config limits governing capacity (max batch, max in-flight, max retries, max bytes, timeouts).
- Metrics suitable for capacity planning: throughput, lag/age, batch size distribution, error/retry rates, store latency, throttling/limits, saturation signals.
- Load-test/perf validation harness availability: bench, synthetic generator, replay tool, docker-compose, docs.

---

## Tool Usage (MANDATORY)

**DO use these tools:**
- ✅ **Glob** for file patterns: `**/*.go`, `internal/*/`, `cmd/*/main.go`, `**/*.{yaml,yml,json,sql,md}`
- ✅ **Grep** for content search:
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

Answer each checklist question with:
- **File path(s) inspected**
- **Short excerpt** or **line reference** as evidence
- **Assessment:** GOOD / CONCERN / RISK / GAP / N/A

When you report *Risks* and *Action Items*, you MUST follow `skills/_shared/RISK_RUBRIC.md`:
- Risks must include all required fields (Severity S0–S3, Priority P0–P2, Likelihood/Blast/Detection, Evidence, Recommendation, Verification).
- If any required field cannot be supported within scope: mark it as **GAP** (do not speculate).
- Action items must be specific, bounded, and verifiable.

Checklist:

| # | Question | How to Verify |
|---|----------|---------------|
| 1 | What are the critical paths and their performance goals? | Trace main flows; find any stated SLOs/budgets; otherwise GAP |
| 2 | Where are throughput bottlenecks likely to occur? | Identify single-threaded stages, locks, sequential IO, batching |
| 3 | Are concurrency and batch sizes bounded and configurable? | Locate config constants/env; check defaults |
| 4 | Is backpressure applied and safe under overload? | Look for rate limits, polling backoff, bounded queues |
| 5 | Are all external calls budgeted with timeouts/retries? | Inspect clients; compute qualitative worst-case amplification |
| 6 | Are data-store writes/queries likely to be efficient (if any store is used)? | Check query/write code + schema/migrations hints + store-specific risks |
| 7 | Are there obvious allocation/CPU red flags in hot loops? | Inspect batch loops and parsing; look for avoidable churn |
| 8 | Do logs/metrics add significant overhead or high cardinality? | Inspect metrics label usage and log volume in hot paths |
| 9 | Are capacity-planning metrics present? | Look for lag, throughput, batch size, latency histograms, saturation |
| 10 | Is there a credible load/perf validation approach? | Look for benchmarks, replay tools, docker-compose, docs |

Additionally:
- Where possible, compute *qualitative* worst-case behavior (no microbenchmarks), e.g. “retries × batch size × timeout → max stall”.

---

## Output

Write exactly ONE report:

- `REPORT_PATH = <target>/docs/rda/reports/09_performance_capacity.md`

Report must follow this structure and order:

# Performance & Capacity Report

## 0. Run Metadata
- Timestamp (UTC)
- Git Commit (or GAP)

## 1. Context Recap
- 3–7 bullets: runtime model, primary flows, and why performance matters here
- Relevant context from `ARCH_DECISIONS_PATH` (batching/retries/backpressure/data-store decisions, if any)

## 2. Scope & Guardrails Confirmation
- Confirm inspection limited to `TARGET_SERVICE_PATH`
- Confirm no external code was opened
- List recorded EXTERNAL DEPs (record-only)
- List notable GAPs caused by boundary

## 3. Pattern Discovery Summary
| Surface | Status (PRESENT/ABSENT/GAP) | Evidence (paths + brief note) |
|---------|------------------------------|-------------------------------|
| API server critical path | ... | ... |
| Worker/consumer critical path | ... | ... |
| Queues/brokers | ... | ... |
| Data stores | ... | ... |
| Caches | ... | ... |
| Backpressure controls | ... | ... |

## 4. Evidence

### 4.1 Critical Paths Inventory
| Flow | Entrypoint | Steps (high-level) | Blocking IO | Perf Goal/Budget | Assessment |
|------|------------|--------------------|-------------|------------------|------------|
| ... | ... | ... | ... | ... | ... |

### 4.2 Concurrency, Batching, Backpressure
| Control | Location | Default | Configurable? | Evidence | Assessment |
|---------|----------|---------|---------------|----------|------------|
| ... | ... | ... | ... | ... | ... |

### 4.3 Timeouts, Retries, and Worst-Case Amplification
| External Call | Timeout | Retries | Worst-Case Stall (qualitative) | Evidence | Assessment |
|--------------|---------|---------|---------------------------------|----------|------------|
| Queue/broker calls (if any) | ... | ... | ... | ... | ... |
| Primary datastore calls (if any) | ... | ... | ... | ... | ... |
| Cache calls (if any) | ... | ... | ... | ... | ... |
| External HTTP/gRPC (if any) | ... | ... | ... | ... | ... |
| Other critical dependencies (if any) | ... | ... | ... | ... | ... |

### 4.4 Data Store Footprint (If Applicable)
| Aspect | Current Approach | Evidence | Risk | Recommendation |
|--------|------------------|----------|------|----------------|
| Writes (batching/upserts/tx) | ... | ... | ... | ... |
| Reads/queries (scans/index use) | ... | ... | ... | ... |
| Schema/index/partition choices | ... | ... | ... | ... |
| Store-specific heavy ops (compaction/merges/forced reads) | ... | ... | ... | ... |
| Fragmentation/parts growth risk (if applicable) | ... | ... | ... | ... |

### 4.5 Code-Level Hotspot Signals
| Pattern | Location | Why It’s Hot | Evidence | Assessment | Recommendation |
|---------|----------|--------------|----------|------------|----------------|
| ... | ... | ... | ... | ... | ... |

### 4.6 Capacity Signals & Readiness
| Metric / Signal | Present? | Evidence | Capacity Use | GAPs |
|-----------------|----------|----------|--------------|------|
| ... | ... | ... | ... | ... |

### 4.7 Load/Perf Validation Path
- What exists in-repo (benchmarks/tools/docs)
- If missing: what minimum harness is needed and where it should live

## 5. Findings

### 5.1 Strengths
- Bullets with evidence references

### 5.2 Risks
All risks MUST be written using `skills/_shared/RISK_RUBRIC.md` required fields (Category, Severity S0–S3, Priority P0–P2, Impact, Likelihood L1–L3, Blast Radius B1–B3, Detection D1–D3, Evidence, Recommendation, Verification).  
If any field cannot be supported: mark it as GAP.

### 5.3 Gaps
- Items not verifiable within boundary + what to check elsewhere

## 6. Action Items
Use the one-line format from `skills/_shared/RISK_RUBRIC.md`:
`<Action>` | **Priority:** P0/P1/P2 | **Impact:** ... | **Verify:** ...

Prefer actions that are measurable: “Add metric X”, “Add bound Y”, “Reduce label cardinality”, “Add load harness”.

## 7. Delta vs Previous Run
- If prior report existed: 3–10 bullets on differences
- If first run: "First run — no prior report to compare"
- If no material changes: "No material changes detected" + 3–5 re-checked items

---

## Completion Criteria
1) Run Metadata complete
2) All 10 checklist questions answered with evidence + assessment
3) Critical paths documented with clear bottleneck hypotheses
4) Concurrency/batching/backpressure boundedness assessed
5) Timeouts/retries worst-case behavior reviewed
6) Data-store footprint assessed (or N/A with evidence)
7) Capacity metrics readiness assessed
8) Practical load/perf validation plan included
9) Risks and action items comply with `skills/_shared/RISK_RUBRIC.md`
10) Delta section present
11) Report saved to `<target>/docs/rda/reports/09_performance_capacity.md`
