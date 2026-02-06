# SKILL: performance

# Performance & Capacity

## Role & Objective

You are a **MAANG Principal Backend Engineer** auditing production readiness of a single service. Your task in this step is to evaluate **performance and capacity readiness**: latency/throughput critical paths, batching and concurrency, memory/CPU hotspots, queue lag dynamics, ClickHouse query/write patterns, and capacity planning signals (limits, backpressure, load testability).

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

## Inputs

- `<target>` (optional): service root directory. If not provided, use `.`.

### Mandatory Context (Read FIRST)
1) `ARCH_DECISIONS_PATH = <target>/docs/rda/ARCHITECTURAL_DECISIONS.md`
- Use as context for intentional trade-offs that affect performance (batching, at-least-once semantics, idempotency, ClickHouse table engines, query patterns).
- Do NOT treat as unquestionable truth.
- If missing/unreadable: record as **GAP** and proceed.

### Prior Reports (Read SECOND)
2) Read existing reports in `<target>/docs/rda/reports/` (if present), especially:
- service inventory / architecture (critical paths, entrypoints)
- integrations (SQS/Dynamo/ClickHouse clients)
- data handling (SCD2, merge engines, dedup)
- reliability & observability (concurrency/backpressure, metrics)
  If missing: record as GAP and proceed.

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

## What to Inspect (PERF/CAPACITY-SCOPED)

Within `TARGET_SERVICE_PATH` ONLY.

### 1) Identify Critical Paths (Latency vs Throughput)
- For API servers: request path → handlers → usecases → repositories.
- For workers/consumers: poll → batch → process → write → ack.
- Document what is on the hot path and what is best-effort.

### 2) Concurrency, Batching, and Backpressure
- Worker pool sizes, goroutine fan-out, channel buffering.
- Batch size controls and flush timers.
- Rate limiting, queue polling intervals, adaptive backoff.
- Any unbounded growth risks (queues, slices, maps, caches).

### 3) Timeouts and Budgeting
- End-to-end time budgets per operation (e.g., visibility timeout vs batch processing time).
- Per-call timeouts to SQS/Dynamo/ClickHouse, retries, and their worst-case amplification.
- Head-of-line blocking risks (single writer, global locks, shared clients).

### 4) ClickHouse Performance Footprint (If Used)
- Insert patterns: batch sizes, insert frequency, async inserts, retries.
- Query patterns: use of FINAL, heavy merges, wide scans, ORDER BY keys, partitions.
- Table engines and compaction/parts growth risks.
- Schema decisions affecting performance (SCD2 history tables, dedup strategy).

### 5) Memory/CPU Hotspots (From Code Signals)
- Large allocations, per-message JSON/proto decode, repeated conversions.
- Use of reflection, regex, string concatenation in hot loops.
- Potential N+1 patterns in ingestion.
- Logging/metrics label cardinality overhead.

### 6) Capacity Signals & Operational Readiness
- Any config limits that govern capacity (max batch, max in-flight, max retries, max bytes).
- Whether there are metrics suitable for capacity planning: throughput, lag/age, batch size distribution, error/retry rates, CH insert latency, Dynamo throttles.
- Load-test harness availability: local bench, synthetic generator, replay tool.

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

Answer each checklist question with:
- **File path(s) inspected**
- **Short excerpt** or **line reference** as evidence
- **Assessment:** GOOD / CONCERN / RISK / GAP

Checklist:

| # | Question | How to Verify |
|---|----------|---------------|
| 1 | What are the critical paths and their performance goals? | Trace main flows; find any stated SLOs/budgets; otherwise GAP |
| 2 | Where are throughput bottlenecks likely to occur? | Identify single-threaded stages, locks, sequential IO, batching |
| 3 | Are concurrency and batch sizes bounded and configurable? | Locate config constants/env; check defaults |
| 4 | Is backpressure applied and safe under overload? | Look for rate limits, queue polling backoff, bounded queues |
| 5 | Are all external calls budgeted with timeouts/retries? | Inspect clients; compute worst-case amplification |
| 6 | Are ClickHouse writes/queries likely to be efficient? | Check insert/query code + migrations schema hints |
| 7 | Are there obvious allocation/CPU red flags in hot loops? | Inspect batch loops and parsing; look for avoidable churn |
| 8 | Do logs/metrics add significant overhead or high cardinality? | Inspect metrics label usage and log volume in hot paths |
| 9 | Are capacity-planning metrics present? | Look for lag, throughput, batch size, latency histograms |
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
- Relevant context from `ARCH_DECISIONS_PATH` (batching/idempotency/ClickHouse decisions, if any)

## 2. Scope & Guardrails Confirmation
- Confirm inspection limited to `TARGET_SERVICE_PATH`
- Confirm no external code was opened
- List recorded EXTERNAL DEPs (record-only)
- List notable GAPs caused by boundary

## 3. Evidence

### 3.1 Critical Paths Inventory
| Flow | Entrypoint | Steps (high-level) | Blocking IO | Perf Goal/Budget | Assessment |
|------|------------|--------------------|-------------|------------------|------------|
| Worker ingest | ... | ... | ... | ... | ... |
| API query | ... | ... | ... | ... | ... |

### 3.2 Concurrency, Batching, Backpressure
| Control | Location | Default | Configurable? | Evidence | Assessment |
|---------|----------|---------|---------------|----------|------------|
| Worker concurrency | ... | ... | ... | ... | ... |
| Batch size/flush | ... | ... | ... | ... | ... |
| Poll interval/backoff | ... | ... | ... | ... | ... |
| Bounded queues | ... | ... | ... | ... | ... |

### 3.3 Timeouts, Retries, and Worst-Case Amplification
| External Call | Timeout | Retries | Worst-Case Stall (qualitative) | Evidence | Assessment |
|--------------|---------|---------|---------------------------------|----------|------------|
| SQS receive/delete | ... | ... | ... | ... | ... |
| Dynamo put/get | ... | ... | ... | ... | ... |
| ClickHouse insert/query | ... | ... | ... | ... | ... |

### 3.4 ClickHouse Footprint (If Applicable)
| Aspect | Current Approach | Evidence | Risk | Recommendation |
|--------|------------------|----------|------|----------------|
| Inserts | ... | ... | ... | ... |
| FINAL usage | ... | ... | ... | ... |
| Parts/merges risk | ... | ... | ... | ... |
| SCD2 tables/keys | ... | ... | ... | ... |

### 3.5 Code-Level Hotspot Signals
| Pattern | Location | Why It’s Hot | Evidence | Assessment | Recommendation |
|---------|----------|--------------|----------|------------|----------------|
| Large allocs in loop | ... | ... | ... | ... | ... |
| Excess logging | ... | ... | ... | ... | ... |
| High-card labels | ... | ... | ... | ... | ... |

### 3.6 Capacity Signals & Readiness
| Metric / Signal | Present? | Evidence | Capacity Use | GAPs |
|-----------------|----------|----------|--------------|------|
| Throughput | ... | ... | ... | ... |
| Queue age/lag | ... | ... | ... | ... |
| Batch size distro | ... | ... | ... | ... |
| CH insert latency | ... | ... | ... | ... |
| Dynamo throttles | ... | ... | ... | ... |

### 3.7 Load/Perf Validation Path
- What exists in-repo (benchmarks/tools/docs)
- If missing: what minimum harness is needed and where it should live

## 4. Findings

### 4.1 Strengths
- Bullets with evidence references

### 4.2 Risks
- Bullets with evidence, severity, impact (tie to latency, throughput, cost, overload behavior)

### 4.3 Gaps
- Items not verifiable within boundary + what to check elsewhere

## 5. Action Items

### P0 (Critical)
- Item | Impact | Evidence | Verification (how to prove improved)

### P1 (Important)
- Item | Impact | Evidence | Verification

### P2 (Nice-to-have)
- Item | Impact | Evidence | Verification

Prefer actions that are measurable: “Add metric X”, “Add bound Y”, “Reduce label cardinality”, “Add load harness”.

## 6. Delta vs Previous Run
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
6) ClickHouse footprint assessed (or GAP)
7) Capacity metrics readiness assessed
8) Practical load/perf validation plan included
9) Action items are prioritized and verifiable
10) Delta section present
11) Report saved to `<target>/docs/rda/reports/09_performance_capacity.md`
