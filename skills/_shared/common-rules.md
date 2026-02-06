# Rubber Duck Audit — Common Agent Rules (Shared Across All Steps)

This document defines **mandatory, shared rules** for every prompt in the Rubber Duck Audit sequence.
All step-specific prompts must **reference and follow** these rules.

---

## 1) Inputs & Scope

### Run root (`<run_root>`)
- `<run_root>` is the **directory where the prompt is executed** (the agent’s current working directory).
- All paths in outputs (reports, evidence pointers, referenced docs) MUST be **relative to `<run_root>`**.
- Absolute paths are strictly prohibited:
    - no leading `/` (e.g., `/Users/...`, `/home/...`)
    - no Windows drive prefixes (e.g., `C:\...`)
    - no machine-specific home paths
- If a tool returns paths like `./internal/app/server.go`, normalize by stripping the leading `./`.

### Target directory (`<target>`)
- The audit operates on a **single directory scope**: `<target>` is the **root boundary**.
- The agent must analyze **everything inside `<target>` and all subdirectories**.
- The agent **must not** read, search, infer, or reference files **outside** `<target>`.

### Default scope
- If `<target>` is not provided, the agent must use `.` as `<target>` **relative to `<run_root>`**.

### No scope expansion
- Do not "peek" into parent folders, sibling services, or repository-wide docs unless they are **inside `<target>`**.
- If something seems missing, call it out as a **gap** rather than expanding scope.

### Tool usage for scope enforcement
- ✅ DO: Use Glob with narrow patterns: `internal/*/`, `cmd/*/main.go`, `**/*.go` (specific extension)
- ✅ DO: Use Grep with file type filters: `--type go`, `--glob "*.yaml"`
- ✅ DO: Use Read tool directly for known file paths
- ❌ DON'T: Use Bash `ls -R`, `find .`, or other recursive listing commands
- ❌ DON'T: Use Glob `**/*` without extension filter (too broad, lists everything)
- ❌ DON'T: Read files without verifying path starts with `<target>/`

---

## 2) Mandatory Pre-Reads (Context)

Before producing any new report, the agent must read (paths are relative to `<run_root>`):

### A) Existing reports directory
- Read **all** files under:
    - `<target>/docs/rda/reports`
- Treat these reports as **context**, not absolute truth.
- Reports are **iterative** and may contain outdated statements or unresolved TODOs.

### B) Previous run report (self-report)
- The agent must find and read **its own previous report** from the prior run (if it exists) in:
    - `<target>/docs/rda/reports`
- The new report must be an **evolution** of the previous one, **not** a rewrite-from-scratch.
- The agent must:
    - preserve validated conclusions,
    - update what changed,
    - re-check **critical issues** from the previous run and state whether they are **fixed / partially fixed / not fixed**.

### C) Architectural decisions
- Read:
    - `<target>/docs/rda/ARCHITECTURAL_DECISIONS.md`
- Use it as **intent and tradeoff context** (why decisions were made).
- Do **not** treat it as final authority:
    - if a decision is harmful, risky, outdated, or inconsistent with best practice, the agent must **flag it** with concrete risks and alternatives.

---

## 3) Output Requirements

### Report location (mandatory)
- The agent must write a Markdown report to (path relative to `<run_root>`):
    - `<target>/docs/rda/reports/<REPORT_NAME>.md`

### Report naming
- Each step prompt should define `<REPORT_NAME>` deterministically (e.g., `01_inventory.md`, `02_architecture.md`, etc.).
- If the step prompt does not define it, default to a descriptive, stable name:
    - `rda_step_<N>.md` (avoid timestamps in filenames).

---

## 4) Iteration Rules (Delta-First)

The report must be written as an **iterative update**:

### A) Preserve continuity
- Keep the structure compatible with the prior report whenever possible.
- Avoid rephrasing sections that did not materially change.

### B) Focus on change + verification
- The report must explicitly include:
    - **Delta vs previous run** (what changed, what stayed the same, what is newly discovered)
    - **Status of previously critical findings** (fixed/partial/not fixed) with evidence pointers

### C) Do not “assume fixed”
- Never mark an item fixed unless there is direct evidence in `<target>` that it was fixed
  (code/config/tests/docs changes inside scope).

---

## 5) Evidence & Precision

### Evidence format (mandatory)
- **File path:** Use format `<relative-path>:<line-range>`, where `<relative-path>` is relative to `<run_root>` (e.g., `internal/app/server.go:45-48` OR `services/foo/internal/app/server.go:45-48` depending on where `<target>` is rooted).
    - Must NOT start with `/` or contain machine-specific prefixes.
    - If a tool returns `./...`, strip the leading `./`.
- **Short excerpt:** ≤3 lines ONLY, in **inline format** (never paste full functions or large blocks)
- **NO markdown code blocks:** NEVER use ` ``` ` blocks in report body (per report style constraint)
- **Correct format example:** "Evidence: config/loader.go:23-25 — validates required fields with `validator.Struct(cfg)` before use."
- For long evidence: Use "see lines 45-68" instead of pasting code
- Every non-trivial claim must include: file path + short excerpt OR line reference
- **Inline code** (single backticks like `variableName`) is acceptable for identifiers
- Any reference to `<target>`-scoped locations MUST still follow the `<run_root>`-relative rule (i.e., write `<target>/...` as a relative path, never an absolute filesystem path).

### Confidence levels (binary only)
- **VERIFIED:** Direct evidence observed in files within `<target>/`
- **GAP:** Cannot be determined within scope; mark what is missing (see RISK_RUBRIC.md)
- ❌ Do NOT use "INFERRED", "ASSUMED", or "LIKELY" — these are forms of speculation

### Avoid speculation
- If something is unclear: mark as **GAP** and continue
- **NEVER ask clarifying questions** — this is audit mode, not interactive mode
    - ❌ Wrong: "Should I use approach A or B?"
    - ✅ Correct: "Approach A is standard for MAANG services. Marked approach choice as GAP pending ADR."
- Best-effort delivery is preferred over asking questions
- List what evidence would be needed to verify (for follow-up)

---

## 6) Respecting Constraints & Tradeoffs

### Intent-aware critique
- If the system intentionally made a tradeoff (documented in `ARCHITECTURAL_DECISIONS.md`):
    - recognize it,
    - assess whether it still holds,
    - document risks and mitigations,
    - propose alternatives only when valuable.

### Best-practice bar
- The goal is **MAANG-grade** production readiness.
- The agent must flag patterns that are acceptable in small systems but risky at scale.

---

## 7) Non-Goals & Safety Rails

### No code changes unless requested
- The agent’s default output is an **audit report** and **action plan**, not a patch.
- If code changes are desirable, describe them as actionable items with scope.

### No repo-wide policy invention
- If the service has established local conventions, follow them.
- If conventions are absent, recommend reasonable defaults explicitly labeled as recommendations.

---

## 8) Subagents (Optional Parallelization)

The agent **MAY** spawn up to **15 subagents** to improve throughput and depth of analysis (e.g., parallel inspection of different areas, cross-checking prior findings, building action-item drafts).

### Hard constraints for subagents
- **Same boundary:** subagents must treat `<target>` as a hard scope boundary (no reads/search outside `<target>`).
- **Same pre-reads:** subagents must follow the same mandatory pre-read rules if their task depends on that context.
- **No file writes:** subagents must not write or modify any files. They return findings to the main agent only.
- **Evidence-first:** every subagent output must include concrete evidence pointers (paths/symbols/keys) and those paths MUST be relative to `<run_root>`.
- **No duplication:** each subagent must have a clearly scoped objective to avoid redundant work.
- **Advisory only:** subagent outputs are suggestions; the main agent must verify evidence before promoting items to P0/P1.

### Coordination rules
- The main agent must:
    - define each subagent’s objective + expected output format (short),
    - consolidate and de-duplicate subagent findings,
    - resolve contradictions by re-checking evidence,
    - remain fully accountable for the final report quality and constraint compliance.

---

## 9) Minimal Report Skeleton (Shared Template)

Every report should contain, at minimum:

1. **Run Metadata**
- `<run_root>`
- `<target>`
- audit step name
- date/time
2. **Executive Summary**
- key risks + readiness verdict (brief)
3. **Delta vs Previous Run**
4. **Findings**
- grouped by the step’s scope
- each finding includes: impact, evidence, recommendation
5. **Previously Critical Findings Status**
6. **Action Plan**
- ordered by priority (P0/P1/P2)
- effort estimate (S/M/L) optional

---

## 10) Absolute Musts (Checklist)

- [ ] Treat `<target>` as a hard boundary; analyze everything within it.
- [ ] Read all existing reports in `docs/rda/reports`.
- [ ] Read the previous self-report and evolve it; verify old critical items.
- [ ] Read `ARCHITECTURAL_DECISIONS.md` and respect intent, but critique if needed.
- [ ] Write a Markdown report into `docs/rda/reports`.
- [ ] Evidence-first, delta-first, no speculative “fixed”.
- [ ] All paths in the report are relative to `<run_root>` (no absolute paths, no `./` prefix).
- [ ] (Optional) If using subagents: max 15, same `<target>` boundary, no file writes; main agent consolidates and owns final output.
