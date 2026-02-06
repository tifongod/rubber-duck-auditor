# Rubber Duck Audit — Common Agent Rules (Shared Across All Steps)

This document defines **mandatory, shared rules** for every prompt in the Rubber Duck Audit sequence.
All step-specific prompts must **reference and follow** these rules.

---

## 1) Inputs & Scope

### Target directory (`<target>`)
- The audit operates on a **single directory scope**: `<target>` is the **root boundary**.
- The agent must analyze **everything inside `<target>` and all subdirectories**.
- The agent **must not** read, search, infer, or reference files **outside** `<target>`.

### Default scope
- If `<target>` is not provided, the agent must start from the **current working directory** (`.`) as `<target>`.

### No scope expansion
- Do not “peek” into parent folders, sibling services, or repository-wide docs unless they are **inside `<target>`**.
- If something seems missing, call it out as a **gap** rather than expanding scope.

---

## 2) Mandatory Pre-Reads (Context)

Before producing any new report, the agent must read:

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
- The agent must write a Markdown report to:
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

### Evidence-based claims
- For any non-trivial claim, the agent should provide:
  - file path(s)
  - relevant symbol(s) or config key(s)
  - short code references (avoid long copy/paste)

### Avoid speculation
- If something is unclear, say:
  - **unknown**, **not found in scope**, or **needs confirmation**
- List what evidence would confirm it.

---

## 6) Respecting Constraints & Tradeoffs

### Intent-aware critique
- If the system intentionally made a tradeoff (documented in `ARCHITECTURAL_DECISIONS.md`):
  - recognize it,
  - assess whether it still holds,
  - document risks and mitigations,
  - propose alternatives only when valuable.

### Best-practice bar
- The goal is **FAANG-grade** production readiness.
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

## 8) Minimal Report Skeleton (Shared Template)

Every report should contain, at minimum:

1. **Run Metadata**
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

## 9) Absolute Musts (Checklist)

- [ ] Treat `<target>` as a hard boundary; analyze everything within it.
- [ ] Read all existing reports in `docs/rda/reports`.
- [ ] Read the previous self-report and evolve it; verify old critical items.
- [ ] Read `ARCHITECTURAL_DECISIONS.md` and respect intent, but critique if needed.
- [ ] Write a Markdown report into `docs/rda/reports`.
- [ ] Evidence-first, delta-first, no speculative “fixed”.

