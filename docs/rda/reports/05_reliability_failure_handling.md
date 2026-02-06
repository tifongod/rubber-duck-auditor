# Step 05: Reliability & Failure Handling Report

## 0. Run Metadata
- **Timestamp (UTC):** 2026-02-07 19:45:00 UTC
- **Audit Author:** [Rubber Duck Auditor v0.1.5](https://github.com/tifongod/rubber-duck-auditor) (Claude Code plugin)
- **Git Commit:** 65b6c585b8b8ca8f5ca32e119c212253f649b1a8

> **SECURITY NOTICE:** This report may contain code excerpts and file paths from the audited codebase. If the audited codebase contains committed secrets (API keys, credentials, tokens), they may appear in evidence sections. **Do NOT commit this report to public repositories without redacting sensitive data.** Audit reports are diagnostic artifacts intended for private review only.

## 1. Context Recap
- **Service:** rubber-duck-auditor (RDA) -- Claude Code plugin providing production readiness audit playbooks via SKILL.md files. Pure markdown, zero executable code, no servers, no queues, no databases.
- **Scope for this step:** Reliability and failure handling -- pipeline orchestration resilience, subagent failure recovery, report write correctness, fast-path cache integrity, backpressure/concurrency controls, interruption handling, and delivery semantics for the report-based state model.
- **Relevant prior reports used:** `docs/rda/reports/00_service_inventory.md` (current run, commit 65b6c58), `docs/rda/reports/03_external_integrations.md` (current run, commit 65b6c58), `docs/rda/reports/04_data_handling.md` (current run, commit 65b6c58), `docs/rda/reports/05_reliability_failure_handling.md` (prior run, commit f192c31)
- **Key components involved:**
  - Audit pipeline orchestrator with 7-wave execution model (`skills/audit-pipeline/SKILL.md:93-177`)
  - Subagent isolation and budget management (`skills/audit-pipeline/SKILL.md:84-89`)
  - Wave completion verification and pipeline quality gates (`skills/audit-pipeline/SKILL.md:271-312`)
  - Fast-path optimization with P0 exception (all 10 step skills, e.g., `skills/reliability/SKILL.md:45-62`)
  - GAP-based fail-gracefully error recovery (`skills/_shared/GUARDRAILS.md:47, 90-93, 95-98`)
  - Security hardening controls affecting input validation reliability (`skills/_shared/GUARDRAILS.md:19-29, 63-71, 100-113`)
  - `<run_root>` path hygiene enforcement (`skills/_shared/common-rules.md:10-17`)

### Decision Context (from `docs/rda/ARCHITECTURAL_DECISIONS.md`)
- **GAP** -- File does not exist at `docs/rda/ARCHITECTURAL_DECISIONS.md`. An empty file exists at `docs/ARCHITECTURAL_DECISIONS.md` (wrong path, 0 bytes). Cannot verify intentional tradeoffs for failure handling strategy (e.g., why GAP-based recovery vs hard failures, why single-retry vs exponential backoff, why 10-subagent cap, why no pipeline resumability). Carried forward from prior run; partially addressed (file created but empty and mislocated).

## 2. Scope & Guardrails Confirmation
- Inspection limited to: `.` (repository root, relative to `<run_root>`)
- No external code opened: CONFIRMED
- No files outside TARGET_SERVICE_PATH accessed: CONFIRMED

### External Dependencies Recorded

| Import / Reference | Type | Used In | Notes |
|---|---|---|---|
| Claude Code Tools API (Glob/Grep/Read/Write/Bash) | EXTERNAL DEP / Platform | All step SKILL.md files | Tool failure modes and timeout behavior unknown; assumed fail-fast for file operations |
| Git CLI | EXTERNAL DEP / Tool | All 10 step skills (fast-path optimization, lines 45-62) | Used for change detection; failures fall back to GAP marking per `GUARDRAILS.md:47` |
| Claude AI LLM Runtime | EXTERNAL DEP / Runtime | All 12 SKILL.md files | Skill execution timeout and failure modes unknown; handled by platform |

## 3. Evidence

### 3.1 Processing Architecture

This system uses a **wave-based execution model** for audit pipeline orchestration with fail-gracefully error recovery. There is no traditional runtime (no servers, no queues, no goroutines). All "processing" is LLM prompt execution writing markdown reports.

**Critical Path Flow:**

User Invocation --> Audit Pipeline Orchestrator (`skills/audit-pipeline/SKILL.md`) --> Wave 1: Inventory (1 subagent) --> writes `00_*.md` --> Wave 2: Architecture + Config (2 parallel subagents) --> writes `01_*.md`, `02_*.md` --> Wave 3: Integrations (1 subagent) --> writes `03_*.md` --> Wave 4: Data + Observability (2 parallel subagents) --> writes `04_*.md`, `06_*.md` --> Wave 5: Reliability (1 subagent) --> writes `05_*.md` --> Wave 6: Quality + Security + Performance (3 parallel subagents) --> writes `07_*.md`, `08_*.md`, `09_*.md` --> Wave 7: Executive Summary (1 subagent) --> writes `summary/00_summary.md`

| Component | Location | Purpose | Key Config | Assessment |
|-----------|----------|---------|------------|------------|
| Wave Orchestrator | `skills/audit-pipeline/SKILL.md:93-177` | Enforces dependency order via 7 waves; spawns subagents per step | Max 10 subagents total, max 3 concurrent per wave | CORRECT -- prevents missing context via sequential wave gates |
| Subagent Budget Cap | `skills/audit-pipeline/SKILL.md:84-89` | Limits total spawned subagents to prevent resource exhaustion | 10 total max, 3 concurrent max; summary shares slot with step 09 | CONCERN -- math inconsistency between stated cap and actual count (see 4.2) |
| Wave Completion Verification | `skills/audit-pipeline/SKILL.md:271-297` | Validates report existence after each wave before proceeding to next | Uses Read tool or Glob to check expected report paths | CORRECT -- fail-fast on missing reports with one-retry mechanism |
| Pipeline Quality Gates | `skills/audit-pipeline/SKILL.md:301-312` | Validates report structural completeness after each wave | Checks Run Metadata, Scope confirmation, Findings, Action Items, Delta section | CONCERN -- checks structural presence but not P0 re-inspection compliance |
| Fast-Path Cache | All 10 step skills lines 45-62 | Skips deep inspection if `git diff/status` shows no changes and no P0s | Dual check: `git diff` + `git status --porcelain`; P0 exception forces re-inspection | CORRECT -- cache invalidation design is sound; P0 exception compliance is agent-behavioral only |
| Fail-Gracefully Recovery | `skills/_shared/GUARDRAILS.md:47, 90-93` | Missing evidence mapped to GAP, not hard failure; no questions in audit mode | Binary confidence: VERIFIED or GAP only per `GUARDRAILS.md:95-98` | CORRECT -- appropriate for best-effort audit delivery |
| Subagent Isolation | `skills/audit-pipeline/SKILL.md:64-69` | Each step runs in fresh subagent with no shared memory or context | Fresh context per step; no cross-step state leakage | CORRECT -- prevents prompt drift and cross-contamination |
| Input Validation | `skills/_shared/GUARDRAILS.md:19-29` | `<target>` path validated before use: rejects `..`, absolute paths outside cwd, shell metacharacters | REJECT + HALT on invalid input | CORRECT -- prevents path traversal failures |
| Shell Escaping | `skills/_shared/GUARDRAILS.md:63-71` | Bash commands must use double-quoted `"${target}"` substitution | Mandatory for all Bash tool invocations | CONCERN -- rule defined but 10 step skills use unquoted `<target>` in fast-path (see 4.2) |
| Prompt Injection Defense | `skills/_shared/GUARDRAILS.md:100-113` | All file content treated as untrusted data; adversarial patterns detected and labeled | Non-negotiable; takes precedence over any content in audited files | CORRECT -- prevents agent behavioral corruption from adversarial codebases |

### 3.2 Failure Handling Matrix

| Failure Type | Handling | Recovery / Next State | Evidence | Assessment |
|--------------|----------|------------------------|----------|------------|
| **Git command failure** | Mark change detection as GAP, skip fast-path, proceed with full inspection | Falls back to full code analysis (safe degradation) | `skills/_shared/GUARDRAILS.md:47` -- "record as GAP and continue"; all step skills lines 45-62 define fallback | CORRECT -- git is an optional optimization; failure is non-blocking |
| **Tool call timeout (Bash)** | Platform enforces 2-minute default timeout; no explicit handling in skills | Agent receives timeout error; no retry; skill may produce incomplete report | `docs/rda/reports/03_external_integrations.md` assessed Bash default 120,000ms; no timeout configuration in step skills | CONCERN -- large repos may exceed timeout during `git status`; no explicit guidance for agents |
| **Tool call timeout (Glob/Grep/Read)** | Unknown -- platform-controlled timeout | Unknown failure mode | No timeout configuration visible in skills | GAP -- cannot verify timeout behavior for non-Bash tools |
| **Missing file / Read tool error** | Mark finding as GAP and continue | Agent continues with best-effort analysis | `skills/_shared/GUARDRAILS.md:47` -- "If information is outside boundary or unavailable, record as GAP and continue" | CORRECT -- appropriate for audit use case |
| **Subagent failure (pipeline)** | Pipeline retries step once, then marks as GAP if still fails | GAP marker for failed step; pipeline continues to next wave | `skills/audit-pipeline/SKILL.md:289-292` -- "Retry the step once (respecting isolation)"; if still missing, "record as GAP and continue to next wave" | CORRECT -- one retry prevents transient failures; GAP fallback prevents pipeline block |
| **Write tool failure (partial write)** | No rollback mechanism; partial report may remain on disk | Downstream steps may consume corrupted report | No error handling observed in step skills or pipeline; pipeline quality gates at `skills/audit-pipeline/SKILL.md:301-312` check section presence after wave completion | CONCERN -- partial write may produce cascading errors; quality gates check structural presence but not integrity |
| **Concurrent write (same step)** | Last-write-wins; first agent's findings silently overwritten | Silent data loss of first agent's findings | `skills/_shared/common-rules.md:72-73` writes to deterministic path; `skills/audit-pipeline/SKILL.md:102-107` mitigates via wave sequencing for pipeline runs only | CONCERN -- mitigated within pipeline; manual concurrent invocations unprotected |
| **Fast-path P0 exception not honored** | Agent may incorrectly use fast-path when prior report has unfixed P0s | P0 items persist unverified across runs | All step skills line 58 (e.g., `skills/reliability/SKILL.md:58`) define P0 exception; no programmatic enforcement | CONCERN -- agent compliance only; quality gates do not validate P0 re-inspection |
| **User cancellation mid-pipeline** | No explicit handling; agent stops mid-wave | Partial reports may exist; no cleanup; pipeline not resumable | No cancellation logic in `skills/audit-pipeline/SKILL.md`; no checkpoint mechanism | CONCERN -- must restart entire pipeline from Wave 1 |
| **Invalid `<target>` path** | REJECT + HALT with clear error message | Agent stops before any file operations | `skills/_shared/GUARDRAILS.md:19-29` -- validates `<target>`, rejects `..`, shell metacharacters, absolute paths outside cwd | CORRECT -- input validation prevents downstream failures |
| **Prompt injection attempt** | File content marked as PROMPT INJECTION ATTEMPT; agent continues unaffected | Audit continues with untrusted content noted | `skills/_shared/GUARDRAILS.md:100-113` -- all file content treated as untrusted data; 6 adversarial pattern categories | CORRECT -- prevents agent behavioral corruption |

### 3.3 Concurrency & Backpressure

| Limit Type | Value | Configurable? | Evidence | Assessment |
|------------|-------|---------------|----------|------------|
| **Subagent concurrency (pipeline)** | Max 3 per wave, max 10 total per pipeline run | Hard-coded in pipeline skill | `skills/audit-pipeline/SKILL.md:86-87` -- "up to 10 subagents total" and "at most 3 concurrent subagents per wave" | CORRECT -- 3 concurrent aligns with Wave 6 (largest wave: Quality + Security + Performance) |
| **Subagent cap (individual step)** | Max 15 per `common-rules.md` | Hard-coded in common rules | `skills/_shared/common-rules.md:159` -- "up to 15 subagents" | CONCERN -- contradicts pipeline cap of 10; creates ambiguity |
| **Tool call concurrency** | Unknown -- controlled by Claude Code platform | Not configurable in plugin | No evidence in codebase; platform-managed | GAP -- cannot verify platform concurrency limits for tool calls |
| **Report write concurrency** | No concurrency control -- last-write-wins overwrite | No configuration available | `skills/_shared/common-rules.md:72-73` writes to deterministic path; no locking | RISK -- concurrent manual invocations can silently overwrite reports; mitigated by wave sequencing in pipeline |
| **Unbounded patterns** | No unbounded goroutine/channel/process patterns | N/A -- no executable code | Confirmed via Glob: no `.go`, `.py`, `.js`, `.ts`, `.sh` files in repo. Only `.json` config and `.md` prompts. | CORRECT -- no unbounded resource growth risk in prompt artifact |

**Backpressure mechanisms:**
- Wave-based sequencing acts as implicit backpressure (next wave blocked until current wave completes). Evidence: `skills/audit-pipeline/SKILL.md:104-105` -- "Execute one wave at a time" and "Wait for wave completion."
- Subagent budget cap (10 total) prevents runaway spawning. Evidence: `skills/audit-pipeline/SKILL.md:86`.
- Report verification between waves blocks progression until reports exist. Evidence: `skills/audit-pipeline/SKILL.md:295-297` -- "Do NOT proceed to next wave until: All expected reports from current wave exist OR are explicitly marked as GAP."
- No explicit backpressure on tool call rate (relies on platform limits).

### 3.4 Graceful Shutdown Analysis

**FINDING:** No traditional graceful shutdown exists. This is a stateless prompt system, not a runtime service. The relevant "shutdown" concept is **pipeline interruption handling**.

| Phase | Implementation | Timeout | Evidence | Assessment |
|-------|----------------|---------|----------|------------|
| **User cancellation** | No explicit handling -- agent stops mid-wave | Unknown -- platform-controlled | No cancellation logic in `skills/audit-pipeline/SKILL.md` | GAP -- partial reports may exist with no cleanup |
| **Skill execution timeout** | Platform enforces session timeout (unknown value) | Unknown -- platform-controlled | `.claude-plugin/plugin.json` has no timeout config | GAP -- long-running skills may timeout with incomplete output |
| **Partial wave completion** | Pipeline checks report existence after each wave; retries missing reports once | No explicit timeout for wave completion check | `skills/audit-pipeline/SKILL.md:271-297` -- wave completion verification is mandatory; `skills/audit-pipeline/SKILL.md:289-292` -- retry once then GAP | CORRECT -- report verification prevents proceeding with incomplete wave |
| **In-flight tool calls** | No explicit handling -- tool calls assumed to complete or timeout | Bash: 2 min default; others unknown | Prior report `docs/rda/reports/03_external_integrations.md` assessed Bash default 120,000ms | GAP -- cannot verify cleanup behavior if agent stops mid-tool-call |

**Pipeline resumability assessment:** If a pipeline run stops mid-execution (cancellation, timeout, crash), **no checkpoint/resume mechanism exists**. Pipeline must be rerun from Wave 1. The fast-path optimization partially mitigates this: if code is unchanged, previously completed steps can use fast-path (skip deep inspection), reducing re-execution overhead to metadata-only updates for completed waves. However, this only works if prior reports from the interrupted run were successfully written.

### 3.5 Delivery Semantics (At-Least-Once / Deadlines)

| Aspect | Implementation | Aligned with ADR? | Evidence | Assessment |
|--------|----------------|-------------------|----------|------------|
| **Report write semantics** | At-most-once overwrite -- last write wins if concurrent | GAP (no ADR) | `docs/rda/reports/04_data_handling.md` assessed overwrite semantics; `skills/_shared/common-rules.md:72-73` deterministic paths | CORRECT for pipeline runs (wave sequencing prevents concurrent writes); RISK for manual concurrent invocations |
| **Retry behavior (pipeline)** | One retry per failed step, then GAP fallback | GAP (no ADR) | `skills/audit-pipeline/SKILL.md:289-292` -- "Retry the step once (respecting isolation)" | CORRECT -- balances reliability (one retry catches transient failures) with fail-fast (no infinite loops) |
| **Idempotency** | Report overwrites are idempotent (deterministic paths, full content replacement) | GAP (no ADR) | All step skills write to stable paths per `skills/_shared/PIPELINE.md:78-89`; re-running same step overwrites prior report | CORRECT -- safe to re-run; no side effects beyond filesystem write |
| **Queue/redelivery behavior** | Not applicable -- not a queue-based system | N/A | No message queues, workers, or redelivery semantics in this prompt-based plugin | CORRECT -- no redelivery handling needed |
| **Deadline/timeout handling** | No explicit deadlines; fast-path has no timeout; full inspection has no timeout | GAP (no ADR) | All step skills lines 45-62 define fast-path with no timeout; no execution budget in skills | CONCERN -- very large codebases may cause indefinite inspection time |
| **Poison handling / DLQ** | Not applicable -- not a queue-based system | N/A | No poison message concept applicable to prompt-based audit | CORRECT -- no DLQ handling needed |

### 3.6 Resilience Patterns

| Pattern | Implementation | Evidence | Assessment |
|---------|----------------|----------|------------|
| **Retry with backoff** | Partial -- single retry with no backoff delay (pipeline only) | `skills/audit-pipeline/SKILL.md:289-292` -- "Retry the step once" (no delay or backoff mentioned) | ACCEPTABLE -- single retry sufficient for transient tool failures; backoff unnecessary for local file I/O |
| **Timeouts** | Partial -- Bash tool has platform default 2-min timeout; other tools unknown | No explicit timeout configuration in skills; assessed in `docs/rda/reports/03_external_integrations.md` | CONCERN -- no explicit timeout guidance for large repos |
| **Circuit breaker** | NOT IMPLEMENTED -- no circuit breaker for repeated tool failures | No evidence of circuit breaker patterns in any skill file | ACCEPTABLE -- not needed for local file I/O; fails fast with no cascading failure risk |
| **Bulkhead isolation** | YES -- wave-based execution with subagent isolation per step | `skills/audit-pipeline/SKILL.md:64-69` -- "each step runs in a fresh subagent" with no shared context | CORRECT -- step isolation prevents cross-contamination; failure in one step cannot corrupt another |
| **Load shedding** | YES -- subagent budget cap (10 max) prevents overload | `skills/audit-pipeline/SKILL.md:86` -- hard cap on subagent count | CORRECT -- prevents runaway resource consumption |
| **Fail-gracefully** | YES -- GAP-based error recovery for all missing/unavailable evidence | `skills/_shared/GUARDRAILS.md:47` -- "record as GAP and continue"; `GUARDRAILS.md:90-93` -- "NEVER ask clarifying questions"; `GUARDRAILS.md:95-98` -- binary VERIFIED/GAP only | CORRECT -- appropriate for best-effort audit delivery |
| **Input validation** | YES -- `<target>` path validated before any file operations | `skills/_shared/GUARDRAILS.md:19-29` -- rejects `..`, shell metacharacters, absolute paths outside cwd; HALT on failure | CORRECT -- prevents downstream failures from invalid input |
| **Prompt injection defense** | YES -- adversarial file content detected, labeled, and ignored | `skills/_shared/GUARDRAILS.md:100-113` -- all file content treated as untrusted data; 6 adversarial pattern categories | CORRECT -- prevents agent behavioral corruption |

**Key finding:** System consistently uses **fail-gracefully** pattern (mark as GAP and continue) rather than fail-fast for evidence-gathering failures. This is appropriate for an audit tool where partial results are more valuable than complete failure. The single-retry mechanism for subagent failures (`skills/audit-pipeline/SKILL.md:289-292`) adds a targeted reliability layer for the pipeline mode. Input validation and prompt injection defense add two additional resilience patterns.

## 4. Findings

### 4.1 Strengths
- **Wave-based execution with dependency gates prevents missing context** -- 7 dependency waves ensure later steps consume completed earlier reports. Wave completion verification (`skills/audit-pipeline/SKILL.md:271-297`) blocks progression until reports exist or are explicitly marked as GAP. Evidence: `skills/audit-pipeline/SKILL.md:93-177` defines waves; `skills/audit-pipeline/SKILL.md:295-297` enforces gate.

- **Subagent isolation prevents cross-contamination** -- Each step runs in a fresh subagent with no shared memory or context. Failure in one step cannot corrupt another step's findings. Evidence: `skills/audit-pipeline/SKILL.md:64-69` -- "each step runs in a fresh subagent with no reliance on prior step chat context."

- **Hard resource caps prevent runaway execution** -- Max 10 subagents total, max 3 concurrent per wave. Prevents resource exhaustion if agent misinterprets instructions. Evidence: `skills/audit-pipeline/SKILL.md:86-87`.

- **Fail-gracefully strategy enables best-effort delivery** -- When git unavailable, files missing, or evidence outside scope, system marks as GAP and continues. Binary confidence levels (VERIFIED/GAP only) prevent speculative findings. Evidence: `skills/_shared/GUARDRAILS.md:47, 90-93, 95-98`.

- **Fast-path cache with correctness guards** -- Dual check (`git diff` + `git status --porcelain`) ensures cache invalidation on code changes. P0 exception forces re-inspection of critical items. Explicit disclosure in delta section (fast-path note) prevents consumer confusion. Evidence: all step skills lines 45-62 (e.g., `skills/reliability/SKILL.md:45-62`).

- **One-retry mechanism reduces transient failures in pipeline** -- Pipeline retries failed steps once before marking as GAP. Balances reliability with fail-fast (no infinite retry loops). Evidence: `skills/audit-pipeline/SKILL.md:289-292`.

- **Input validation and prompt injection defense** -- `<target>` path validation rejects traversal attempts and shell metacharacters with HALT on failure (`skills/_shared/GUARDRAILS.md:19-29`). Prompt injection defense treats all file content as untrusted data (`skills/_shared/GUARDRAILS.md:100-113`). Both prevent classes of reliability failures.

- **Pipeline quality gates validate report structure** -- Post-wave validation checks reports for Run Metadata, Scope confirmation, Findings, Action Items, and Delta section. Evidence: `skills/audit-pipeline/SKILL.md:301-312`.

- **Idempotent report writes enable safe re-runs** -- Re-running any step or full pipeline overwrites prior report at deterministic path with no side effects. Evidence: all step skills write to stable paths per `skills/_shared/PIPELINE.md:78-89`.

### 4.2 Risks

#### P0 (Critical)
None identified. This is a static prompt engineering artifact with no runtime production risk. Failure modes affect audit quality (incomplete reports, missing findings) but do not cause service outages, data loss, or security breaches. The prior run also had no P0s; this assessment is unchanged.

#### P1 (Important)
- **Shell escaping rule not propagated to step skill fast-path sections** -- Category: `reliability/security`. Severity: S1. Likelihood: L2 (possible when `<target>` contains spaces or special characters). Blast Radius: B2 (all 10 step skills affected). Detection: D2 (silent failure or incorrect fast-path activation). `skills/_shared/GUARDRAILS.md:65-67` mandates `git diff -- "${target}"` (double-quoted), but all 10 step skills use `git diff -- <target>` and `git status --porcelain -- <target>` (unquoted) in fast-path sections. Confirmed via Grep: 10 unquoted `git diff` occurrences at `skills/inventory/SKILL.md:48`, `skills/architecture/SKILL.md:45`, `skills/config/SKILL.md:45`, `skills/integrations/SKILL.md:48`, `skills/data/SKILL.md:48`, `skills/reliability/SKILL.md:48`, `skills/observability/SKILL.md:45`, `skills/quality/SKILL.md:49`, `skills/security/SKILL.md:50`, `skills/performance/SKILL.md:47`; plus 10 unquoted `git status` occurrences at the subsequent line in each. If git command fails silently due to unquoted path with spaces, empty output may be misinterpreted as "no changes," incorrectly activating fast-path and producing a stale report. This is both a security risk (command injection via malicious `<target>`) and a reliability risk (stale cache). Recommendation: Update all 10 step skills to use `"${target}"` quoting consistent with `GUARDRAILS.md:65-67`. Verification: Grep for unquoted `<target>` in git commands returns zero matches. **Status: NOT FIXED since prior run (also flagged in `docs/rda/reports/03_external_integrations.md` and `docs/rda/reports/04_data_handling.md`).**

- **Subagent cap contradiction between common-rules (15) and pipeline (10)** -- Category: `correctness/reliability`. Severity: S2. Likelihood: L2 (agents follow whichever limit they encounter first). Blast Radius: B2 (all pipeline runs and individual step runs affected). Detection: D2 (discovered when agent exceeds 10 or avoids exceeding 15). `skills/_shared/common-rules.md:159` says "up to 15 subagents" while `skills/audit-pipeline/SKILL.md:86` says "up to 10 subagents total." Individual step runs (governed by common-rules) believe they can use 15, while pipeline runs (governed by pipeline SKILL.md) cap at 10. If a step within the pipeline spawns 15 micro-subagents, it exceeds the pipeline budget. Impact: unpredictable subagent behavior; potential resource exhaustion or artificial limitation. Evidence: `skills/_shared/common-rules.md:159`, `skills/audit-pipeline/SKILL.md:86`. Recommendation: Align both documents to single consistent cap (e.g., 10 for pipeline, parametric per-step). Verification: Grep for subagent cap numbers returns consistent values. **Status: NOT FIXED since prior run.**

- **Concurrent write risk outside pipeline orchestration** -- Category: `data/reliability`. Severity: S1. Likelihood: L2 (possible when users manually invoke same step twice). Blast Radius: B1 (single report affected). Detection: D3 (silent loss -- no error, no log, no conflict marker). If two agents execute the same step simultaneously outside the audit pipeline, last-write-wins overwrites first agent's findings with no conflict detection. Impact: silent data loss. Mitigation: wave-based execution prevents concurrent writes within pipeline. Evidence: `skills/_shared/common-rules.md:72-73` writes to deterministic path; `skills/audit-pipeline/SKILL.md:102-107` mitigates via wave sequencing for pipeline runs only. Recommendation: Add advisory WARNING to all step skills: "Do not run this step concurrently with itself." Verification: Warning present in all 10 SKILL.md files. **Status: NOT FIXED since prior run.**

- **Fast-path P0 exception not programmatically enforced** -- Category: `reliability`. Severity: S2. Likelihood: L2 (agent may misinterpret prior report or skip P0 check). Blast Radius: B2 (P0 items may persist unverified across runs). Detection: D2 (detectable via manual review of delta section). Fast-path rules (all step skills line 58, e.g., `skills/reliability/SKILL.md:58`) require re-inspection if prior report has P0 items, but compliance is agent-behavioral only. Pipeline quality gates at `skills/audit-pipeline/SKILL.md:301-312` check structural completeness but not P0 re-inspection evidence. Evidence: all step skills line 58; `skills/audit-pipeline/SKILL.md:301-312`. Recommendation: Extend quality gates to validate: if prior report had P0 items AND new report uses fast-path delta note, fail validation. Verification: Quality gate rejects fast-path reports with unresolved prior P0s. **Status: NOT FIXED since prior run.**

- **No rollback mechanism for partial write failures** -- Category: `reliability/data`. Severity: S2. Likelihood: L1 (rare -- filesystem issues uncommon). Blast Radius: B2 (affects downstream steps consuming corrupted report). Detection: D2 (readable via Read tool but corruption may not be obvious). Write tool may fail mid-write, leaving incomplete report content. Downstream steps would consume malformed markdown. Evidence: no error handling or rollback in step skills or pipeline quality gates; Write tool atomicity unknown (platform-managed). Recommendation: Add post-write structural validation: read back written report and confirm required sections present before marking wave complete. Verification: Quality gate reads and validates report structure after each step write. **Status: NOT FIXED since prior run.**

- **No timeout configuration for git operations on large repos** -- Category: `reliability`. Severity: S2. Likelihood: L2 (possible on large monorepos with 10k+ files). Blast Radius: B1 (single audit run hangs). Detection: D2 (user observes hang with no progress indication). Git commands via Bash inherit 2-minute default timeout. Large repos may exceed this during `git status`, causing unclear failure. Impact: audit hang or timeout with no actionable error message. Evidence: all step skills lines 48-49 use `git diff/status` with no timeout args. Recommendation: Add explicit Bash timeout parameter (e.g., 30 seconds) or document expected behavior on large repos. Verification: Test on repo with 10,000+ files. **Status: NOT FIXED since prior run.**

- **Report filename inconsistencies across definition files** -- Category: `correctness/reliability`. Severity: S2. Likelihood: L3 (every pipeline run encounters these). Blast Radius: B1 (reports written to wrong location; downstream consumers may not find them). Detection: D1 (immediate -- report file missing from expected location). Three inconsistencies confirmed via Grep: (a) `skills/audit-pipeline/SKILL.md:225` defines security report as `08_security_and_privacy.md` while `skills/_shared/PIPELINE.md:58,87`, `skills/security/SKILL.md:167`, and `skills/audit-pipeline/SKILL.md:142,286` use `08_security_privacy.md`. (b) `skills/audit-pipeline/SKILL.md:229` defines performance report as `09_performance_and_capacity.md` while `skills/_shared/PIPELINE.md:63,88` and `skills/performance/SKILL.md:155` use `09_performance_capacity.md`. Actual file on disk is `docs/rda/reports/09_performance_and_capacity.md` (follows the non-canonical name). (c) Summary path conflict: `skills/exec-summary/SKILL.md:21` defines `<TARGET>/docs/rda/reports/SUMMARY.md`; `skills/audit-pipeline/SKILL.md:147,234,336` references `<target>/docs/rda/summary/00_summary.md`; `skills/_shared/PIPELINE.md:68,89` lists `SUMMARY.md` in reports directory. Impact: wave verification may fail if reports are written to paths that differ from what pipeline checks for. Recommendation: Align all files to canonical names: `08_security_privacy.md`, `09_performance_capacity.md`, `docs/rda/summary/00_summary.md`. Verification: Grep for each report filename returns identical path across all referencing files. **Status: NOT FIXED since prior run (also flagged in `docs/rda/reports/03_external_integrations.md` and `docs/rda/reports/04_data_handling.md`).**

- **Pipeline subagent math inconsistency (11 vs stated 10)** -- Category: `correctness`. Severity: S2. Likelihood: L2. Blast Radius: B1. Detection: D2. `skills/audit-pipeline/SKILL.md:86` caps at 10 subagents. Line 88 acknowledges "10 steps + 1 summary = 11, but summary can share with step 09." However, the wave execution example at lines 149-177 shows 7 waves spawning 1+2+1+2+1+3+1 = 11 subagents independently (Wave 6 runs steps 07, 08, 09 and Wave 7 runs summary as separate subagent). Line 177 claims "Total subagents: 10 (within budget)" -- this count is incorrect; the example spawns 11. Impact: agents may hit budget cap before summary or skip summary to stay within budget. Recommendation: Either raise the cap to 11 or restructure Wave 7 to reuse a Wave 6 slot (e.g., run summary after step 09 completes in Wave 6, sharing its subagent). Verification: Subagent count in wave example matches stated cap. **Status: NOT FIXED since prior run.**

#### P2 (Nice-to-have)
- **No circuit breaker for repeated file read failures** -- If filesystem becomes unavailable mid-audit, agent repeatedly calls Read tool and fails. Low likelihood for local filesystem. Evidence: no circuit breaker pattern in skills. Recommendation: Add advisory note in pipeline to fail fast after 10 consecutive Read tool errors. **Status: NOT FIXED since prior run.**

- **No pipeline resumability after interruption** -- If pipeline stops mid-execution, no checkpoint/resume mechanism exists. Must rerun from Wave 1. Fast-path optimization partially mitigates by allowing completed steps to skip deep inspection on rerun. Evidence: no checkpoint logic in `skills/audit-pipeline/SKILL.md`. Recommendation: Add wave completion markers to enable resume from last completed wave. **Status: NOT FIXED since prior run.**

- **Bash tool timeout not documented in skills** -- Skills reference Bash tool for git commands but do not mention default timeout (2 minutes) or how to extend it. Evidence: no timeout documentation in `skills/_shared/common-rules.md:31-37` or step skills "Tool Usage" sections. Recommendation: Add timeout guidance to shared rules. **Status: NOT FIXED since prior run.**

- **`exec-summary` does not reference GUARDRAILS.md** -- `skills/exec-summary/SKILL.md:7-9` references `common-rules.md`, `RISK_RUBRIC.md`, and `PIPELINE.md` but omits `GUARDRAILS.md`. Instead, it inlines its own guardrails (lines 25-42). While exec-summary scope is reports-only (lower risk), security rules (prompt injection defense, path validation) are not inherited from shared source. 11 of 12 skills reference `GUARDRAILS.md`; `exec-summary` is the only exception. Evidence: `skills/exec-summary/SKILL.md:7-9`. Recommendation: Add `GUARDRAILS.md` to exec-summary shared rules. **Status: NOT FIXED since prior run (also flagged in `docs/rda/reports/03_external_integrations.md`).**

- **Git command exit code not validated in fast-path** -- Fast-path checks if `git diff` and `git status --porcelain` return empty strings, but does not validate exit code. If git command fails with non-zero exit and empty stderr, empty output may be interpreted as "no changes." Evidence: all step skills lines 48-49 check output emptiness, not exit code. Recommendation: Update fast-path to require exit code 0 AND empty output. **Status: NOT FIXED since prior run.**

### 4.3 Decision Alignment Assessment
- **GAP** -- Cannot perform decision alignment assessment because `docs/rda/ARCHITECTURAL_DECISIONS.md` does not exist. File exists at wrong path (`docs/ARCHITECTURAL_DECISIONS.md`) and is empty (0 bytes). Cannot determine if reliability tradeoffs (GAP-based recovery, single-retry, no pipeline resumability, 10-subagent cap, no concurrent write protection) are intentional design choices or technical debt. Carried forward from prior run; partially addressed but still non-functional.

### 4.4 Gaps
- **GAP: Claude Code tool timeout behavior unknown** -- Skills use Glob, Grep, Read extensively, but timeout/retry/error behavior is not documented in this codebase. Missing: platform documentation on tool failure modes. Why it matters: large codebases (100k+ files) may trigger tool timeouts during Glob or Grep operations.

- **GAP: Platform session timeout unknown** -- Skills are prompts executed by Claude AI runtime. Timeout for skill execution (per invocation) not visible in plugin metadata. Missing: platform documentation on session timeout or token budget limits. Why it matters: long-running skills (e.g., exec-summary consolidating 10 reports, large codebase audits) may timeout with incomplete output.

- **GAP: Write tool atomicity unknown** -- Claude Code Write tool behavior is external to codebase. Cannot verify if writes are atomic (all-or-nothing) or may leave partial content on failure. Missing: platform documentation on Write tool guarantees. Why it matters: partial writes corrupt reports and cascade errors to downstream steps.

- **GAP: Subagent isolation mechanism unknown** -- Audit pipeline spawns up to 10 subagents, but isolation guarantees (separate context windows? shared filesystem views? memory isolation?) not visible in codebase. Missing: platform documentation on subagent sandboxing. Why it matters: if subagents share state, race conditions could corrupt parallel audit steps within the same wave.

- **GAP: Architectural decisions document missing** -- `docs/rda/ARCHITECTURAL_DECISIONS.md` does not exist (empty file at wrong path). Cannot validate whether reliability tradeoffs are intentional. Why it matters: all audit steps mark tradeoff assessments as GAP.

## 5. Action Items

### P0 (Critical)
None -- static prompt artifact with no production runtime risk. Failure modes affect audit quality but do not cause outages, data loss, or security breaches.

### P1 (Important)
- **Propagate shell escaping to all step skill fast-path sections** | Impact: Closes reliability/security gap where 10 step skills provide unquoted git command examples contradicting `GUARDRAILS.md:65-67` shell escaping mandate; prevents incorrect fast-path activation and command injection | Verify: Update all 10 step skills lines 48-49 to use `git diff -- "${target}"` and `git status --porcelain -- "${target}"`; Grep for unquoted `<target>` in git commands returns zero matches

- **Resolve subagent cap contradiction (15 vs 10) and fix pipeline math (11 vs 10)** | Impact: Eliminates ambiguity for agents deciding how many subagents to spawn; corrects inaccurate budget claim in wave example | Verify: Align `common-rules.md:159` with pipeline cap; update `audit-pipeline/SKILL.md:177` to reflect actual count; Grep for subagent cap numbers returns consistent values

- **Add concurrent execution warning to all step skills** | Impact: Prevents silent data loss when users invoke same step twice in parallel outside pipeline orchestration | Verify: Add WARNING to all 10 step SKILL.md files; confirm warning present

- **Extend pipeline quality gates to validate P0 re-inspection** | Impact: Ensures fast-path P0 exception is enforced -- agent must re-inspect P0 items, not just update metadata | Verify: Update `audit-pipeline/SKILL.md:301-312` to check: if prior report had P0 items AND new report uses fast-path, fail validation

- **Add post-write report validation after each wave** | Impact: Catches corrupted reports from partial write failures before downstream steps consume them | Verify: Update pipeline wave verification to read each completed report and confirm structural completeness; retry on failure

- **Add git operation timeout guidance** | Impact: Prevents audit hangs on large repos | Verify: Add timeout parameter recommendation in step skills fast-path sections (e.g., 30-second Bash timeout) or document expected behavior

- **Align report filenames across all definition files** | Impact: Prevents wave verification failures and downstream data flow errors from path mismatches | Verify: Security report uses `08_security_privacy.md` everywhere; performance report uses one canonical name everywhere; summary uses `docs/rda/summary/00_summary.md` everywhere

### P2 (Nice-to-have)
- **Add circuit breaker guidance for filesystem failures** | Impact: Improves UX by failing fast after repeated Read tool errors | Verify: Pipeline SKILL.md includes fail-fast guidance after consecutive errors

- **Implement wave checkpoint/resume mechanism** | Impact: Enables resuming pipeline from last completed wave after interruption | Verify: Add checkpoint logic to pipeline; test resume from Wave 3 after interruption

- **Document Bash tool timeout in shared rules** | Impact: Sets user expectations for git command timeout behavior | Verify: Timeout documentation present in shared rules with default value

- **Add GUARDRAILS.md reference to exec-summary skill** | Impact: Ensures all 12 skills inherit security rules from shared file | Verify: `exec-summary/SKILL.md` references `GUARDRAILS.md` in shared rules list

- **Validate git command exit codes in fast-path** | Impact: Prevents false fast-path activation if git command fails silently | Verify: Fast-path condition checks exit code 0 AND empty output

## 6. Delta vs Previous Run
- **Previous Report:** `docs/rda/reports/05_reliability_failure_handling.md` at commit `f192c31` dated 2026-02-07 00:30:00 UTC
- **Material changes detected.** Two commits since prior report: `9a9146a` (removed "rda-" prefix from skill directories) and `65b6c58` (added absolute path rules and `<run_root>` concept).

1. **FIXED: Scope section now uses relative path** -- Prior report line 27 used absolute path `/Users/dkolpakov/GolandProjects/rubber-duck-auditor/`. This run uses `.` (repository root) per new `<run_root>` rules in `skills/_shared/common-rules.md:10-17`.

2. **FIXED: Prior report contained markdown code blocks** -- Prior report used triple-backtick code blocks in section 3.1 (processing architecture flow diagram), violating `GUARDRAILS.md:151` ("NO markdown code blocks in report body"). This run uses inline text flow throughout.

3. **UPDATED: Skill directory paths updated throughout** -- All evidence references now use `skills/<name>/SKILL.md` format (was `skills/rda-<name>/SKILL.md` in prior report). Commit 9a9146a renamed directories. E.g., `skills/reliability/SKILL.md` (was `skills/rda-reliability/SKILL.md`).

4. **UPDATED: Shared rules file references corrected** -- `common-rules.md` (was `rda-common-rules.md`). All evidence pointers updated. Commit 9a9146a renamed the file.

5. **NEW: Pipeline subagent math inconsistency identified (P1)** -- Wave execution example at `skills/audit-pipeline/SKILL.md:149-177` spawns 11 subagents (1+2+1+2+1+3+1) but claims "Total subagents: 10" at line 177. The cap at line 86 is 10. Flagged in prior `03_external_integrations.md` as P2 but elevated to P1 here due to direct reliability impact (budget exhaustion before summary).

6. **NEW: `exec-summary` GUARDRAILS.md omission identified (P2)** -- `skills/exec-summary/SKILL.md:7-9` omits `GUARDRAILS.md` from shared rules list, inlining its own guardrails instead. Identified via cross-reference with `docs/rda/reports/03_external_integrations.md`.

7. **NEW: Git exit code validation gap identified (P2)** -- Fast-path checks output emptiness but not git command exit code. Cross-referenced from `docs/rda/reports/04_data_handling.md`.

8. **NOT FIXED: Shell escaping contradiction in all 10 step skills** -- Prior P1. All step skills still use unquoted `git diff -- <target>` at fast-path lines 48-49. Evidence: Grep confirmed 10 unquoted `git diff` and 10 unquoted `git status` occurrences.

9. **NOT FIXED: Subagent cap contradiction (15 vs 10)** -- Prior P1. `skills/_shared/common-rules.md:159` still says "up to 15 subagents"; `skills/audit-pipeline/SKILL.md:86` still says "up to 10."

10. **NOT FIXED: Concurrent write risk outside pipeline** -- Prior P1. No advisory warning added to step skills.

11. **NOT FIXED: Fast-path P0 exception not programmatically enforced** -- Prior P1. Pipeline quality gates still check structural presence only, not P0 re-inspection compliance.

12. **NOT FIXED: No rollback for partial write failures** -- Prior P1. No post-write validation added.

13. **NOT FIXED: No timeout configuration for git operations** -- Prior P1. No explicit timeout in step skills or shared rules.

14. **NOT FIXED: Report filename inconsistencies** -- Prior P1. `skills/audit-pipeline/SKILL.md:225` still uses `08_security_and_privacy.md`; line 229 still uses `09_performance_and_capacity.md`. Summary path conflict persists. On-disk performance report file is `09_performance_and_capacity.md` (does not match canonical `09_performance_capacity.md` from `PIPELINE.md:88`).

15. **NOT FIXED: No circuit breaker, no pipeline resumability, no Bash timeout docs** -- Prior P2s. All carried forward unchanged.

---

<sub>Generated by [Rubber Duck Auditor v0.1.5](https://github.com/tifongod/rubber-duck-auditor) -- a Claude Code plugin for MAANG-grade production readiness audits | Install: `/plugin marketplace add tifongod/rubber-duck-auditor && /plugin install rda@rubber-duck-auditor`</sub>
