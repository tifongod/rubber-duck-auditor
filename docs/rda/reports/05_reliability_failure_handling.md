# Step 05: Reliability & Failure Handling Report

## 0. Run Metadata
- **Timestamp (UTC):** 2026-02-07 11:33:59 UTC
- **Audit Author:** [Rubber Duck Auditor v0.1.8](https://github.com/tifongod/rubber-duck-auditor) (Claude Code plugin)
- **Git Commit:** 28f6bac837ba2d1878523cab3ff8b9f890fd7212
- **Template Version:** v1.0.0

> **⚠️ SECURITY NOTICE:** This report may contain code excerpts and file paths from the audited codebase. If the audited codebase contains committed secrets (API keys, credentials, tokens), they may appear in evidence sections. **Do NOT commit this report to public repositories without redacting sensitive data.** Audit reports are diagnostic artifacts intended for private review only.

## 1. Context Recap
- **Service:** rubber-duck-auditor (RDA) — Claude Code plugin providing production readiness audit playbooks via SKILL.md files. Pure markdown, zero executable code, no servers, no queues, no databases.
- **Scope for this step:** Reliability and failure handling — pipeline orchestration resilience, subagent failure recovery, report write correctness, fast-path cache integrity, backpressure/concurrency controls, interruption handling, and delivery semantics for the report-based state model.
- **Relevant prior reports used:** `docs/rda/reports/00_service_inventory.md` (commit 28f6bac, run 2026-02-07 15:10:00 UTC), `docs/rda/reports/03_external_integrations.md` (commit 28f6bac, run 2026-02-07 17:45:00 UTC), `docs/rda/reports/04_data_handling.md` (commit 28f6bac, run 2026-02-07 11:28:16 UTC)
- **Key components involved:**
  - Audit pipeline orchestrator with 7-wave execution model (`skills/audit-pipeline/SKILL.md:93-177`)
  - Subagent isolation and budget management (`skills/audit-pipeline/SKILL.md:84-89`)
  - Wave completion verification and pipeline quality gates (`skills/audit-pipeline/SKILL.md:271-312`)
  - Fast-path optimization with P0 exception (all 10 step skills, e.g., `skills/reliability/SKILL.md:58-76`)
  - GAP-based fail-gracefully error recovery (`skills/_shared/GUARDRAILS.md:47, 90-93, 95-98`)
  - Security hardening controls affecting input validation reliability (`skills/_shared/GUARDRAILS.md:19-29, 66-74, 103-116`)
  - Concurrent execution warnings added to all 10 step skills (lines 25-31)
  - Troubleshooting guide documenting common failure modes (`docs/rda/TROUBLESHOOTING.md`)

### Decision Context (from `docs/rda/ARCHITECTURAL_DECISIONS.md`)
- **Relevant ADRs:** ADR-003 (Wave-Based Execution Model), ADR-007 (Zero Test Policy)
- **Excerpt(s):** "Use 7-wave execution model... Completion verification gates between waves." (ADR-003:55-58). "Ship without tests. Rely on dogfooding for validation." (ADR-007:132-139).
- **File status:** File exists at correct path with 167 lines and 8 documented ADRs. Resolves prior P1 finding from commit 65b6c58.

## 2. Scope & Guardrails Confirmation
- ✅ **Inspection limited to:** `.` (repository root, relative to `<run_root>`)
- ✅ **No external code opened:** CONFIRMED
- ✅ **No files outside TARGET_SERVICE_PATH accessed:** CONFIRMED

### External Dependencies Recorded

| Import / Reference | Type | Used In | Notes |
|---|---|---|---|
| Claude Code Tools API (Glob/Grep/Read/Write/Bash) | EXTERNAL DEP / Platform | All step SKILL.md files | Tool failure modes and timeout behavior unknown; assumed fail-fast for file operations |
| Git CLI | EXTERNAL DEP / Tool | All 10 step skills (fast-path optimization, lines 61-62) | Used for change detection; failures fall back to full inspection per `GUARDRAILS.md:47` |
| Claude AI LLM Runtime | EXTERNAL DEP / Runtime | All 12 SKILL.md files | Skill execution timeout and failure modes unknown; handled by platform |

## 3. Evidence

### 3.1 Processing Architecture

This system uses a **wave-based execution model** for audit pipeline orchestration with fail-gracefully error recovery. There is no traditional runtime (no servers, no queues, no goroutines). All "processing" is LLM prompt execution writing markdown reports.

**Critical Path Flow:**

User Invocation → Audit Pipeline Orchestrator (`skills/audit-pipeline/SKILL.md`) → Wave 1: Inventory (1 subagent) → writes `00_*.md` → Wave 2: Architecture + Config (2 parallel subagents) → writes `01_*.md`, `02_*.md` → Wave 3: Integrations (1 subagent) → writes `03_*.md` → Wave 4: Data + Observability (2 parallel subagents) → writes `04_*.md`, `06_*.md` → Wave 5: Reliability (1 subagent) → writes `05_*.md` → Wave 6: Quality + Security + Performance (3 parallel subagents) → writes `07_*.md`, `08_*.md`, `09_*.md` → Wave 7: Executive Summary (1 subagent) → writes `summary/00_summary.md`

| Component | Location | Purpose | Key Config | Assessment |
|-----------|----------|---------|------------|------------|
| Wave Orchestrator | `skills/audit-pipeline/SKILL.md:93-177` | Enforces dependency order via 7 waves; spawns subagents per step | Max 11 subagents total, max 3 concurrent per wave | CORRECT — prevents missing context via sequential wave gates |
| Subagent Budget Cap | `skills/audit-pipeline/SKILL.md:84-89` | Limits total spawned subagents to prevent resource exhaustion | 11 total max per line 86, 3 concurrent max per line 87; total needed: 1+2+1+2+1+3+1=11 per line 177 | CORRECT — math now aligns (cap updated from prior 10 to 11) |
| Wave Completion Verification | `skills/audit-pipeline/SKILL.md:271-297` | Validates report existence after each wave before proceeding to next | Uses Bash `ls` or Read tool to check expected report paths | CORRECT — fail-fast on missing reports with one-retry mechanism |
| Pipeline Quality Gates | `skills/audit-pipeline/SKILL.md:301-312` | Validates report structural completeness after each wave | Checks Run Metadata, Scope confirmation, Findings, Action Items, Delta section | CONCERN — checks structural presence but not P0 re-inspection compliance |
| Fast-Path Cache | All 10 step skills lines 58-76 | Skips deep inspection if `git diff/status` shows no changes and no P0s | Dual check: `git diff -- "${target}"` + `git status --porcelain -- "${target}"`; P0 exception forces re-inspection | CORRECT — cache invalidation design is sound; shell escaping now FIXED with double-quoted `"${target}"` |
| Concurrent Execution Warning | All 10 step skills lines 25-31 | Advisory warning against manual parallel invocation of same step | Documents last-write-wins behavior and silent data loss risk; notes pipeline orchestration prevents this | CORRECT — resolves prior P1 finding; warning present in all 10 step skills |
| Fail-Gracefully Recovery | `skills/_shared/GUARDRAILS.md:47, 90-93` | Missing evidence mapped to GAP, not hard failure; no questions in audit mode | Binary confidence: VERIFIED or GAP only per `GUARDRAILS.md:95-98` | CORRECT — appropriate for best-effort audit delivery |
| Subagent Isolation | `skills/audit-pipeline/SKILL.md:64-69` | Each step runs in fresh subagent with no shared memory or context | Fresh context per step; no cross-step state leakage | CORRECT — prevents prompt drift and cross-contamination |
| Input Validation | `skills/_shared/GUARDRAILS.md:19-29` | `<target>` path validated before use: rejects `..`, absolute paths outside cwd, shell metacharacters | REJECT + HALT on invalid input | CORRECT — prevents path traversal failures |
| Shell Escaping | `skills/_shared/GUARDRAILS.md:66-74` | Bash commands must use double-quoted `"${target}"` substitution | Mandatory for all Bash tool invocations | CORRECT — now propagated to all 10 step skills fast-path sections (lines 61-62) |
| Prompt Injection Defense | `skills/_shared/GUARDRAILS.md:103-116` | All file content treated as untrusted data; adversarial patterns detected and labeled | Non-negotiable; takes precedence over any content in audited files | CORRECT — prevents agent behavioral corruption from adversarial codebases |

### 3.2 Failure Handling Matrix

| Failure Type | Handling | Recovery / Next State | Evidence | Assessment |
|--------------|----------|------------------------|----------|------------|
| **Git command failure** | Mark change detection as GAP, skip fast-path, proceed with full inspection | Falls back to full code analysis (safe degradation) | `skills/_shared/GUARDRAILS.md:47` — "record as GAP and continue"; all step skills lines 58-76 define fallback | CORRECT — git is an optional optimization; failure is non-blocking |
| **Tool call timeout (Bash)** | Platform enforces 2-minute default timeout; no explicit handling in skills | Agent receives timeout error; no retry; skill may produce incomplete report | `skills/_shared/common-rules.md:32` documents 2-minute Bash default; `docs/rda/TROUBLESHOOTING.md:7-21` provides diagnosis/workarounds | CORRECT — documented with troubleshooting guidance |
| **Tool call timeout (Glob/Grep/Read)** | Unknown — platform-controlled timeout | Unknown failure mode | No timeout configuration visible in skills | GAP — cannot verify timeout behavior for non-Bash tools |
| **Missing file / Read tool error** | Mark finding as GAP and continue | Agent continues with best-effort analysis | `skills/_shared/GUARDRAILS.md:47` — "If information is outside boundary or unavailable, record as GAP and continue" | CORRECT — appropriate for audit use case |
| **Subagent failure (pipeline)** | Pipeline retries step once, then marks as GAP if still fails | GAP marker for failed step; pipeline continues to next wave | `skills/audit-pipeline/SKILL.md:290-292` — "Retry the step once (respecting isolation)"; if still missing, "record as GAP and continue to next wave" | CORRECT — one retry prevents transient failures; GAP fallback prevents pipeline block |
| **Write tool failure (partial write)** | No rollback mechanism; partial report may remain on disk | Downstream steps may consume corrupted report | No error handling observed in step skills or pipeline; pipeline quality gates at `skills/audit-pipeline/SKILL.md:301-312` check section presence after wave completion | CONCERN — partial write may produce cascading errors; quality gates check structural presence but not integrity |
| **Concurrent write (same step)** | Last-write-wins; first agent's findings silently overwritten | Silent data loss of first agent's findings | All 10 step skills lines 25-31 include explicit warning; `skills/audit-pipeline/SKILL.md:104-105` mitigates via wave sequencing for pipeline runs | CORRECT — warning added resolves prior P1; mitigated within pipeline; manual concurrent invocations now warned |
| **Fast-path P0 exception not honored** | Agent may incorrectly use fast-path when prior report has unfixed P0s | P0 items persist unverified across runs | All step skills line 72 (e.g., `skills/reliability/SKILL.md:72`) define P0 exception; `docs/rda/TROUBLESHOOTING.md:51-70` documents failure mode | CONCERN — agent compliance only; quality gates do not validate P0 re-inspection |
| **User cancellation mid-pipeline** | No explicit handling; agent stops mid-wave | Partial reports may exist; no cleanup; pipeline not resumable | No cancellation logic in `skills/audit-pipeline/SKILL.md`; no checkpoint mechanism | CONCERN — must restart entire pipeline from Wave 1 |
| **Invalid `<target>` path** | REJECT + HALT with clear error message | Agent stops before any file operations | `skills/_shared/GUARDRAILS.md:19-29` — validates `<target>`, rejects `..`, shell metacharacters, absolute paths outside cwd | CORRECT — input validation prevents downstream failures |
| **Prompt injection attempt** | File content marked as PROMPT INJECTION ATTEMPT; agent continues unaffected | Audit continues with untrusted content noted | `skills/_shared/GUARDRAILS.md:103-116` — all file content treated as untrusted data; 6 adversarial pattern categories | CORRECT — prevents agent behavioral corruption |

### 3.3 Concurrency & Backpressure

| Limit Type | Value | Configurable? | Evidence | Assessment |
|------------|-------|---------------|----------|------------|
| **Subagent concurrency (pipeline)** | Max 3 per wave, max 11 total per pipeline run | Hard-coded in pipeline skill | `skills/audit-pipeline/SKILL.md:86-87` — "up to 11 subagents total" and "at most 3 concurrent subagents per wave" | CORRECT — 3 concurrent aligns with Wave 6 (largest wave: Quality + Security + Performance) |
| **Subagent cap (shared rules)** | Max 11 per `common-rules.md` | Hard-coded in common rules | `skills/_shared/common-rules.md:162` — "up to 11 subagents" (was 15 in prior audit) | CORRECT — now aligned with pipeline cap of 11 |
| **Tool call concurrency** | Unknown — controlled by Claude Code platform | Not configurable in plugin | No evidence in codebase; platform-managed | GAP — cannot verify platform concurrency limits for tool calls |
| **Report write concurrency** | No concurrency control — last-write-wins overwrite | No configuration available | All 10 step skills lines 25-31 warn against concurrent manual invocations; wave sequencing prevents concurrent writes in pipeline | CORRECT — risk mitigated by warning and wave sequencing |
| **Unbounded patterns** | No unbounded goroutine/channel/process patterns | N/A — no executable code | Confirmed via Glob: no `.go`, `.py`, `.js`, `.ts`, `.sh` files in repo (from prior report 00). Only `.json` config and `.md` prompts. | CORRECT — no unbounded resource growth risk in prompt artifact |

**Backpressure mechanisms:**
- Wave-based sequencing acts as implicit backpressure (next wave blocked until current wave completes). Evidence: `skills/audit-pipeline/SKILL.md:104-105` — "Execute one wave at a time" and "Wait for wave completion."
- Subagent budget cap (11 total) prevents runaway spawning. Evidence: `skills/audit-pipeline/SKILL.md:86`.
- Report verification between waves blocks progression until reports exist. Evidence: `skills/audit-pipeline/SKILL.md:295-297` — "Do NOT proceed to next wave until: All expected reports from current wave exist OR are explicitly marked as GAP."
- No explicit backpressure on tool call rate (relies on platform limits).

### 3.4 Graceful Shutdown Analysis

**FINDING:** No traditional graceful shutdown exists. This is a stateless prompt system, not a runtime service. The relevant "shutdown" concept is **pipeline interruption handling**.

| Phase | Implementation | Timeout | Evidence | Assessment |
|-------|----------------|---------|----------|------------|
| **User cancellation** | No explicit handling — agent stops mid-wave | Unknown — platform-controlled | No cancellation logic in `skills/audit-pipeline/SKILL.md`; `docs/rda/TROUBLESHOOTING.md:25-48` documents missing report recovery | GAP — partial reports may exist with no cleanup |
| **Skill execution timeout** | Platform enforces session timeout (unknown value) | Unknown — platform-controlled | `.claude-plugin/plugin.json` has no timeout config | GAP — long-running skills may timeout with incomplete output |
| **Partial wave completion** | Pipeline checks report existence after each wave; retries missing reports once | No explicit timeout for wave completion check | `skills/audit-pipeline/SKILL.md:273-297` — wave completion verification is mandatory; `skills/audit-pipeline/SKILL.md:290-292` — retry once then GAP | CORRECT — report verification prevents proceeding with incomplete wave |
| **In-flight tool calls** | No explicit handling — tool calls assumed to complete or timeout | Bash: 2 min default per `common-rules.md:32`; others unknown | `docs/rda/reports/03_external_integrations.md` assessed Bash default 120,000ms | GAP — cannot verify cleanup behavior if agent stops mid-tool-call |

**Pipeline resumability assessment:** If a pipeline run stops mid-execution (cancellation, timeout, crash), **no checkpoint/resume mechanism exists**. Pipeline must be rerun from Wave 1. The fast-path optimization partially mitigates this: if code is unchanged, previously completed steps can use fast-path (skip deep inspection), reducing re-execution overhead to metadata-only updates for completed waves. However, this only works if prior reports from the interrupted run were successfully written.

### 3.5 Delivery Semantics (At-Least-Once / Deadlines)

| Aspect | Implementation | Aligned with ADR? | Evidence | Assessment |
|--------|----------------|-------------------|----------|------------|
| **Report write semantics** | At-most-once overwrite — last write wins if concurrent | ALIGNED (ADR-003 wave sequencing prevents concurrent writes in pipeline) | `docs/rda/reports/04_data_handling.md` assessed overwrite semantics; all 10 step skills lines 25-31 warn against manual concurrent invocations | CORRECT for pipeline runs (wave sequencing prevents concurrent writes); manual concurrent invocations now warned |
| **Retry behavior (pipeline)** | One retry per failed step, then GAP fallback | ALIGNED (ADR-003 wave execution model) | `skills/audit-pipeline/SKILL.md:290-292` — "Retry the step once (respecting isolation)" | CORRECT — balances reliability (one retry catches transient failures) with fail-fast (no infinite loops) |
| **Idempotency** | Report overwrites are idempotent (deterministic paths, full content replacement) | ALIGNED | All step skills write to stable paths per `skills/_shared/PIPELINE.md:78-89`; re-running same step overwrites prior report | CORRECT — safe to re-run; no side effects beyond filesystem write |
| **Queue/redelivery behavior** | Not applicable — not a queue-based system | N/A | No message queues, workers, or redelivery semantics in this prompt-based plugin | CORRECT — no redelivery handling needed |
| **Deadline/timeout handling** | No explicit deadlines; fast-path has no timeout; full inspection has no timeout | GAP (no ADR) | All step skills lines 58-76 define fast-path with no timeout; no execution budget in skills; Bash default 2-min documented in `common-rules.md:32` | CONCERN — very large codebases may cause indefinite inspection time; Bash timeout now documented |
| **Poison handling / DLQ** | Not applicable — not a queue-based system | N/A | No poison message concept applicable to prompt-based audit | CORRECT — no DLQ handling needed |

### 3.6 Resilience Patterns

| Pattern | Implementation | Evidence | Assessment |
|---------|----------------|----------|------------|
| **Retry with backoff** | Partial — single retry with no backoff delay (pipeline only) | `skills/audit-pipeline/SKILL.md:290-292` — "Retry the step once" (no delay or backoff mentioned) | ACCEPTABLE — single retry sufficient for transient tool failures; backoff unnecessary for local file I/O |
| **Timeouts** | Partial — Bash tool has platform default 2-min timeout documented; other tools unknown | `skills/_shared/common-rules.md:32` documents Bash timeout; `docs/rda/TROUBLESHOOTING.md:7-21` provides git timeout guidance | CORRECT — timeout documented with troubleshooting guidance |
| **Circuit breaker** | NOT IMPLEMENTED — no circuit breaker for repeated tool failures | No evidence of circuit breaker patterns in any skill file | ACCEPTABLE — not needed for local file I/O; fails fast with no cascading failure risk |
| **Bulkhead isolation** | YES — wave-based execution with subagent isolation per step | `skills/audit-pipeline/SKILL.md:64-69` — "each step runs in a fresh subagent" with no shared context | CORRECT — step isolation prevents cross-contamination; failure in one step cannot corrupt another |
| **Load shedding** | YES — subagent budget cap (11 max) prevents overload | `skills/audit-pipeline/SKILL.md:86` — hard cap on subagent count | CORRECT — prevents runaway resource consumption |
| **Fail-gracefully** | YES — GAP-based error recovery for all missing/unavailable evidence | `skills/_shared/GUARDRAILS.md:47` — "record as GAP and continue"; `GUARDRAILS.md:90-93` — "NEVER ask clarifying questions"; `GUARDRAILS.md:95-98` — binary VERIFIED/GAP only | CORRECT — appropriate for best-effort audit delivery |
| **Input validation** | YES — `<target>` path validated before any file operations | `skills/_shared/GUARDRAILS.md:19-29` — rejects `..`, shell metacharacters, absolute paths outside cwd; HALT on failure | CORRECT — prevents downstream failures from invalid input |
| **Prompt injection defense** | YES — adversarial file content detected, labeled, and ignored | `skills/_shared/GUARDRAILS.md:103-116` — all file content treated as untrusted data; 6 adversarial pattern categories | CORRECT — prevents agent behavioral corruption |

**Key finding:** System consistently uses **fail-gracefully** pattern (mark as GAP and continue) rather than fail-fast for evidence-gathering failures. This is appropriate for an audit tool where partial results are more valuable than complete failure. The single-retry mechanism for subagent failures (`skills/audit-pipeline/SKILL.md:290-292`) adds a targeted reliability layer for the pipeline mode. Input validation, prompt injection defense, concurrent execution warnings, and shell escaping add additional resilience patterns.

## 4. Findings

### 4.1 Strengths
- **FIXED: Shell escaping propagated to all fast-path sections** — All 10 step skills now use properly quoted `git diff -- "${target}"` (line 61) and `git status --porcelain -- "${target}"` (line 62). Resolves prior P1 security risk about command injection and data correctness risk about silent git failures. Evidence: `skills/reliability/SKILL.md:61-62`, verified across all 10 step skills.

- **FIXED: Concurrent execution warnings added** — All 10 step skills now include explicit warning at lines 25-31: "DO NOT run this skill concurrently with itself... This results in silent data loss." Resolves prior P1 finding about missing advisory warning. Evidence: `skills/reliability/SKILL.md:25-31`, verified across all 10 step skills.

- **FIXED: Subagent cap aligned across documents** — `skills/_shared/common-rules.md:162` now says "up to 11 subagents" matching `skills/audit-pipeline/SKILL.md:86`. Prior contradiction (15 vs 10) has been resolved. Evidence: `skills/_shared/common-rules.md:162`, `skills/audit-pipeline/SKILL.md:86`.

- **FIXED: Bash timeout documented** — `skills/_shared/common-rules.md:32` now documents the 2-minute default Bash tool timeout and provides guidance for long operations. Evidence: `skills/_shared/common-rules.md:32`.

- **NEW: Troubleshooting guide added** — `docs/rda/TROUBLESHOOTING.md` provides diagnosis and solutions for 8 common failure modes: git timeouts, missing reports, fast-path P0 failures, absolute paths, ADR errors, exec-summary failures, reports with questions, subagent budget exceeded. Evidence: `docs/rda/TROUBLESHOOTING.md:1-188`.

- **Wave-based execution with dependency gates prevents missing context** — 7 dependency waves ensure later steps consume completed earlier reports. Wave completion verification (`skills/audit-pipeline/SKILL.md:273-297`) blocks progression until reports exist or are explicitly marked as GAP. Evidence: `skills/audit-pipeline/SKILL.md:93-177` defines waves; `skills/audit-pipeline/SKILL.md:295-297` enforces gate.

- **Subagent isolation prevents cross-contamination** — Each step runs in a fresh subagent with no shared memory or context. Failure in one step cannot corrupt another step's findings. Evidence: `skills/audit-pipeline/SKILL.md:64-69` — "each step runs in a fresh subagent with no reliance on prior step chat context."

- **Hard resource caps prevent runaway execution** — Max 11 subagents total, max 3 concurrent per wave. Prevents resource exhaustion if agent misinterprets instructions. Evidence: `skills/audit-pipeline/SKILL.md:86-87`.

- **Fail-gracefully strategy enables best-effort delivery** — When git unavailable, files missing, or evidence outside scope, system marks as GAP and continues. Binary confidence levels (VERIFIED/GAP only) prevent speculative findings. Evidence: `skills/_shared/GUARDRAILS.md:47, 90-93, 95-98`.

- **Fast-path cache with correctness guards** — Dual check (`git diff` + `git status --porcelain`) with proper shell escaping ensures cache invalidation on code changes. P0 exception forces re-inspection of critical items. Explicit disclosure in delta section (fast-path note) prevents consumer confusion. Evidence: all step skills lines 58-76 (e.g., `skills/reliability/SKILL.md:58-76`).

- **One-retry mechanism reduces transient failures in pipeline** — Pipeline retries failed steps once before marking as GAP. Balances reliability with fail-fast (no infinite retry loops). Evidence: `skills/audit-pipeline/SKILL.md:290-292`.

- **Input validation and prompt injection defense** — `<target>` path validation rejects traversal attempts and shell metacharacters with HALT on failure (`skills/_shared/GUARDRAILS.md:19-29`). Prompt injection defense treats all file content as untrusted data (`skills/_shared/GUARDRAILS.md:103-116`). Both prevent classes of reliability failures.

- **Pipeline quality gates validate report structure** — Post-wave validation checks reports for Run Metadata, Scope confirmation, Findings, Action Items, and Delta section. Evidence: `skills/audit-pipeline/SKILL.md:301-312`.

- **Idempotent report writes enable safe re-runs** — Re-running any step or full pipeline overwrites prior report at deterministic path with no side effects. Evidence: all step skills write to stable paths per `skills/_shared/PIPELINE.md:78-89`.

- **ADR-aware design** — All architectural decisions now documented with explicit status, context, consequences, and rationale. ADR-003 documents wave-based execution. ADR-007 documents zero test policy. Evidence: `docs/rda/ARCHITECTURAL_DECISIONS.md:1-167`.

### 4.2 Risks

#### P0 (Critical)
None identified. This is a static prompt engineering artifact with no runtime production risk. Failure modes affect audit quality (incomplete reports, missing findings) but do not cause service outages, data loss, or security breaches.

#### P1 (Important)
- **Fast-path P0 exception not programmatically enforced** — Category: `reliability`. Severity: S2. Likelihood: L2 (agent may misinterpret prior report or skip P0 check). Blast Radius: B2 (P0 items may persist unverified across runs). Detection: D2 (detectable via manual review of delta section). Fast-path rules (all step skills line 72, e.g., `skills/reliability/SKILL.md:72`) require re-inspection if prior report has P0 items, but compliance is agent-behavioral only. Pipeline quality gates at `skills/audit-pipeline/SKILL.md:301-312` check structural completeness but not P0 re-inspection evidence. `docs/rda/TROUBLESHOOTING.md:51-70` documents the failure mode. Evidence: all step skills line 72; `skills/audit-pipeline/SKILL.md:301-312`; `docs/rda/TROUBLESHOOTING.md:51-70`. Recommendation: Extend quality gates to validate: if prior report had P0 items AND new report uses fast-path delta note, fail validation. Verification: Quality gate rejects fast-path reports with unresolved prior P0s. **Status: NOT FIXED since prior run.**

- **No rollback mechanism for partial write failures** — Category: `reliability/data`. Severity: S2. Likelihood: L1 (rare — filesystem issues uncommon). Blast Radius: B2 (affects downstream steps consuming corrupted report). Detection: D2 (readable via Read tool but corruption may not be obvious). Write tool may fail mid-write, leaving incomplete report content. Downstream steps would consume malformed markdown. Evidence: no error handling or rollback in step skills or pipeline quality gates; Write tool atomicity unknown (platform-managed). `docs/rda/TROUBLESHOOTING.md:25-48` documents missing report recovery but not partial write detection. Recommendation: Add post-write structural validation: read back written report and confirm required sections present before marking wave complete. Verification: Quality gate reads and validates report structure after each step write. **Status: NOT FIXED since prior run.**

#### P2 (Nice-to-have)
- **No circuit breaker for repeated file read failures** — Category: `reliability`. Severity: S3. Likelihood: L1 (extremely rare for local filesystem). Blast Radius: B1 (single audit run). Detection: D2 (agent retries until skill timeout). If filesystem becomes unavailable mid-audit, agent repeatedly calls Read tool and fails. Low likelihood for local filesystem. Evidence: no circuit breaker pattern in skills. Recommendation: Add advisory note in pipeline to fail fast after 10 consecutive Read tool errors. **Status: NOT FIXED since prior run.**

- **No pipeline resumability after interruption** — Category: `operability`. Severity: S3. Likelihood: L2 (users may cancel long-running pipelines). Blast Radius: B1 (user must rerun from Wave 1). Detection: D1 (immediate user observation). If pipeline stops mid-execution, no checkpoint/resume mechanism exists. Must rerun from Wave 1. Fast-path optimization partially mitigates by allowing completed steps to skip deep inspection on rerun. Evidence: no checkpoint logic in `skills/audit-pipeline/SKILL.md`. `docs/rda/TROUBLESHOOTING.md:25-48` documents missing report recovery but not resumability. Recommendation: Add wave completion markers to enable resume from last completed wave. **Status: NOT FIXED since prior run.**

- **Git command exit code not validated in fast-path** — Category: `correctness`. Severity: S3. Likelihood: L1 (rare — corrupted git repo). Blast Radius: B1 (single audit run). Detection: D2 (audit may incorrectly activate fast-path on git error). Fast-path checks if `git diff` and `git status --porcelain` return empty strings, but does not validate exit code. If git command fails with non-zero exit, empty stderr/stdout may be misinterpreted as "no changes". Impact: rare edge case where corrupted git repo triggers fast-path when full inspection is needed. Evidence: all step skills lines 61-62 check output emptiness, not exit code. Recommendation: Update fast-path to require exit code 0 AND empty output. Verification: Test with corrupted git repo; confirm fallback to full inspection. **Status: NOT FIXED since prior run.**

### 4.3 Decision Alignment Assessment
- **Decision:** ADR-003 (Wave-Based Execution Model)
    - **Expected (ADR):** Use 7-wave execution model with explicit data flow via report file paths
    - **Actual:** `skills/audit-pipeline/SKILL.md:93-177` implements 7 waves with completion verification gates between waves; `skills/_shared/PIPELINE.md:93-107` defines preconditions per step
    - **Assessment:** ALIGNED — Implementation matches ADR intent. Report file I/O integration is structured per wave sequencing requirements.
    - **Evidence:** `skills/audit-pipeline/SKILL.md:93-177`, ADR-003:55-66

- **Decision:** ADR-007 (Zero Test Policy)
    - **Expected (ADR):** Ship without tests, rely on dogfooding for validation
    - **Actual:** Multiple reliability improvements (shell escaping, concurrent warnings, subagent cap alignment, Bash timeout docs) were applied between commit f192c31 and 28f6bac after being flagged by dogfooding audits
    - **Assessment:** ALIGNED — ADR policy is working as intended. Dogfooding detected 4 P1 issues which have now been FIXED. Demonstrates validation approach effectiveness.
    - **Evidence:** Shell escaping fixed (all step skills lines 61-62), concurrent warnings added (all step skills lines 25-31), subagent cap aligned (`common-rules.md:162`), Bash timeout documented (`common-rules.md:32`); ADR-007:132-139

### 4.4 Gaps
- **GAP: Claude Code tool timeout behavior unknown** — Skills use Glob, Grep, Read extensively, but timeout/retry/error behavior is not documented in this codebase. Missing: platform documentation on tool failure modes. Why it matters: large codebases (100k+ files) may trigger tool timeouts during Glob or Grep operations.

- **GAP: Platform session timeout unknown** — Skills are prompts executed by Claude AI runtime. Timeout for skill execution (per invocation) not visible in plugin metadata. Missing: platform documentation on session timeout or token budget limits. Why it matters: long-running skills (e.g., exec-summary consolidating 10 reports, large codebase audits) may timeout with incomplete output.

- **GAP: Write tool atomicity unknown** — Claude Code Write tool behavior is external to codebase. Cannot verify if writes are atomic (all-or-nothing) or may leave partial content on failure. Missing: platform documentation on Write tool guarantees. Why it matters: partial writes corrupt reports and cascade errors to downstream steps.

- **GAP: Subagent isolation mechanism unknown** — Audit pipeline spawns up to 11 subagents, but isolation guarantees (separate context windows? shared filesystem views? memory isolation?) not visible in codebase. Missing: platform documentation on subagent sandboxing. Why it matters: if subagents share state, race conditions could corrupt parallel audit steps within the same wave.

## 5. Action Items

### P0 (Critical)
None — static prompt artifact with no production runtime risk. Failure modes affect audit quality but do not cause outages, data loss, or security breaches.

### P1 (Important)
- **Extend pipeline quality gates to validate P0 re-inspection** | Impact: Ensures fast-path P0 exception is enforced — agent must re-inspect P0 items, not just update metadata | Verify: Update `skills/audit-pipeline/SKILL.md:301-312` to check: if prior report had P0 items AND new report delta says "fast-path", flag as quality gate failure

- **Add post-write report validation after each wave** | Impact: Catches corrupted reports from partial write failures before downstream steps consume them | Verify: Update pipeline wave verification to read each completed report and confirm structural completeness; retry on failure

### P2 (Nice-to-have)
- **Add circuit breaker guidance for filesystem failures** | Impact: Improves UX by failing fast after repeated Read tool errors | Verify: Pipeline SKILL.md includes fail-fast guidance after consecutive errors

- **Implement wave checkpoint/resume mechanism** | Impact: Enables resuming pipeline from last completed wave after interruption | Verify: Add checkpoint logic to pipeline; test resume from Wave 3 after interruption

- **Validate git command exit codes in fast-path** | Impact: Prevents false fast-path activation if git command fails silently | Verify: Fast-path condition checks exit code 0 AND empty output

## 6. Delta vs Previous Run
- **Previous Report:** `docs/rda/reports/05_reliability_failure_handling.md` at commit `65b6c585b8b8ca8f5ca32e119c212253f649b1a8` dated 2026-02-07 19:45:00 UTC
- **Material changes detected.** Four commits since prior report: `cd55ce0` (added instructions to README.md + output path fixes), `b8c5cb0` (dogfooding), `d817044` (removed analytics-specific rules from data skill), `28f6bac` (fixed low-hanging fruits from audits).

1. **FIXED: Shell escaping propagated to all step skills (P1)** — Prior report identified this as NOT FIXED. Verified now: all 10 step skills use properly quoted `git diff -- "${target}"` (line 61) and `git status --porcelain -- "${target}"` (line 62). Evidence: `skills/reliability/SKILL.md:61-62`, manual verification confirms 0 unquoted instances in fast-path sections.

2. **FIXED: Concurrent execution warnings added (P1)** — Prior report identified this as NOT FIXED. Verified now: all 10 step skills include explicit warning at lines 25-31. Evidence: `skills/reliability/SKILL.md:25-31`, verified across all 10 step skills.

3. **FIXED: Subagent cap aligned (P1)** — Prior report identified contradiction (15 vs 10). Verified now: `skills/_shared/common-rules.md:162` says "up to 11 subagents" matching `skills/audit-pipeline/SKILL.md:86`. Evidence: `skills/_shared/common-rules.md:162`, `skills/audit-pipeline/SKILL.md:86`.

4. **FIXED: Bash timeout documented (P2)** — Prior report identified this as NOT FIXED. Verified now: `skills/_shared/common-rules.md:32` documents the 2-minute default Bash tool timeout. Evidence: `skills/_shared/common-rules.md:32`.

5. **NEW: Troubleshooting guide added** — `docs/rda/TROUBLESHOOTING.md` (188 lines) provides diagnosis and solutions for 8 common failure modes. Not present in prior report. Evidence: `docs/rda/TROUBLESHOOTING.md:1-188`.

6. **NEW: ADR documentation complete** — `docs/rda/ARCHITECTURAL_DECISIONS.md` now exists with 167 lines and 8 documented ADRs including ADR-003 (Wave-Based Execution Model) and ADR-007 (Zero Test Policy) relevant to reliability. Prior report noted file was missing. Evidence: `docs/rda/ARCHITECTURAL_DECISIONS.md:1-167`.

7. **NOT FIXED: Fast-path P0 exception not programmatically enforced (P1)** — Prior report identified this as NOT FIXED. Status unchanged. Pipeline quality gates still check only structural presence, not P0 re-inspection evidence. `docs/rda/TROUBLESHOOTING.md:51-70` now documents the failure mode but does not implement enforcement. Evidence: `skills/audit-pipeline/SKILL.md:301-312`, `docs/rda/TROUBLESHOOTING.md:51-70`.

8. **NOT FIXED: No rollback mechanism for partial write failures (P1)** — Prior report identified this as NOT FIXED. Status unchanged. No error handling, retry, or post-write validation observed. `docs/rda/TROUBLESHOOTING.md:25-48` documents missing report recovery but not partial write detection. Evidence: no changes to write workflow in step skills.

9. **NOT FIXED: No circuit breaker for repeated tool failures (P2)** — Prior report identified this as NOT FIXED. Status unchanged. Evidence: no circuit breaker pattern in skills.

10. **NOT FIXED: No pipeline resumability (P2)** — Prior report identified this as NOT FIXED. Status unchanged. `docs/rda/TROUBLESHOOTING.md:25-48` documents missing report recovery but not checkpoint/resume capability. Evidence: no checkpoint logic in pipeline.

11. **NOT FIXED: Git exit code not validated (P2)** — Prior report identified this as NOT FIXED. Status unchanged. Fast-path still checks output emptiness only. Evidence: `skills/reliability/SKILL.md:61-62` check output, not exit code.

12. **RESOLVED: Prior report issues with markdown code blocks and absolute paths** — Prior report violated formatting rules (`GUARDRAILS.md:151` — no markdown code blocks) and path hygiene rules (`common-rules.md:10-17` — no absolute paths). This run complies with both. Evidence: inline text flow throughout, relative path `.` in scope section.

13. **RESOLVED: Prior report identified subagent math inconsistency (11 vs stated 10)** — Pipeline cap was 10 at line 86 in prior report, but wave execution required 11 (1+2+1+2+1+3+1). Cap has been updated to 11 at `skills/audit-pipeline/SKILL.md:86`. Evidence: `skills/audit-pipeline/SKILL.md:86` now says "up to 11 subagents total".

14. **Summary:** 4 of 5 prior P1 NOT FIXED findings have been FIXED (shell escaping, concurrent warnings, subagent cap alignment, Bash timeout docs). 2 P1s remain open (P0 enforcement, rollback mechanism). All 3 prior P2s remain open (circuit breaker, resumability, exit code validation). All FIXED items were reliability/correctness improvements applied after dogfooding detected them (per ADR-007).

---

<sub>Generated by [Rubber Duck Auditor v0.1.8](https://github.com/tifongod/rubber-duck-auditor) — a Claude Code plugin for MAANG-grade production readiness audits | Install: `/plugin marketplace add tifongod/rubber-duck-auditor && /plugin install rda@rubber-duck-auditor`</sub>
