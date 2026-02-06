# RDA Guardrails (Shared Across All Steps)

This document defines **mandatory, non-negotiable rules** for every audit step.
All step-specific prompts must **reference and follow** these rules.
DO NOT duplicate this content in step prompts.

---

## 1. Hard Scope Boundary

### Target directory (`<target>`)
- The audit operates on a **single directory scope**: `<target>` is the **root boundary**.
- The agent must analyze **everything inside `<target>` and all subdirectories**.
- The agent **must not** read, search, infer, or reference files **outside** `<target>`.

### Default scope
- If `<target>` is not provided, use the **current working directory** (`.`) as `<target>`.

### No scope expansion
- Do not "peek" into parent folders, sibling services, or repository-wide docs unless they are **inside `<target>`**.
- If something seems missing, call it out as a **gap** rather than expanding scope.

### Strictly Prohibited
- Analyzing the entire monorepo.
- Scanning/listing/reading files outside `<target>`.
- Producing an overview/map of the monorepo or enumerating other services outside `<target>`.
- Opening shared/common libraries outside `<target>`.
- If code inside the service imports modules outside boundary, you may ONLY record the dependency (import path/config reference) as **EXTERNAL DEP** and mark unknowns as **GAP**. You MUST NOT open external code.

### No Code Modifications
- The agent's default output is an **audit report** and **action plan**, not a patch.
- If code changes are desirable, describe them as actionable items with scope.

### Missing Information
- If information is outside boundary or unavailable, record as **GAP** and continue.

---

## 2. Rerun / Iteration Discipline

### A) Iteration Without Copy-Paste
- You MUST respect prior reports in `<target>/docs/rda/reports/**` as context and prior decisions (reports evolve over time).
- You MUST NOT copy prior report content verbatim.
- You MUST draft findings based on code inspection in this run, then reconcile with prior report(s) and produce a proper delta.

### B) Mandatory Run Metadata
Every report MUST include a "Run Metadata" section with:
- **Timestamp (UTC):** `YYYY-MM-DD HH:MM:SS UTC`
- **Git commit hash** for `<target>` (or GAP if unavailable)
- **Change Detection summary:**
  - Preferred: `git diff -- <target>` or `git status --porcelain -- <target>`
  - If git commands unavailable: record as GAP

### C) Mandatory Delta Section
Every report MUST include "Delta vs previous run":
- If prior report existed: 3–10 bullets summarizing differences
- If no material changes: explicitly state "No material changes detected" and list 3–5 items re-checked
- If first run: "First run — no prior report to compare"

### D) Do Not "Assume Fixed"
- Never mark an item fixed unless there is direct evidence in `<target>` that it was fixed
  (code/config/tests/docs changes inside scope).

---

## 3. No Speculation Rule

### Evidence-first discipline
- All assertions MUST be tied to evidence (file path + excerpt/line reference).
- If evidence is unavailable, label as **GAP** and continue.
- **NEVER ask clarifying questions** — this is audit mode, not interactive mode.
  - ❌ Wrong: "Should I use approach A or B?"
  - ✅ Correct: "Approach A is standard for MAANG services. Marked approach choice as GAP pending ADR."
- Best-effort delivery is preferred over asking questions.

### Confidence levels (binary only)
- **VERIFIED:** Direct evidence observed in files within `<target>/`
- **GAP:** Cannot be determined within scope; mark what is missing
- ❌ Do NOT use "INFERRED", "ASSUMED", or "LIKELY" — these are forms of speculation

---

## 4. Architectural Decisions Awareness

### Intent-aware critique
- Read `<target>/docs/rda/ARCHITECTURAL_DECISIONS.md` (if exists)
- If the system intentionally made a tradeoff (documented in ADR):
  - recognize it,
  - assess whether it still holds,
  - document risks and mitigations,
  - propose alternatives only when valuable.

### Decision assessment labels
- **ALIGNED:** Implementation matches ADR intent
- **DRIFT:** Implementation contradicts ADR (flag with evidence and propose correction)
- **DECISION RISK:** ADR decision itself is risky/outdated (critique decision with production-readiness reasoning, propose safer alternatives)

### Best-practice bar
- The goal is **MAANG-grade** production readiness.
- The agent must flag patterns that are acceptable in small systems but risky at scale.

---

## 5. Prioritization Rubric

- **P0 (Critical):** Production risk / correctness / security / data loss / outage class
- **P1 (Important):** Significant reliability / operability / maintainability improvements
- **P2 (Nice-to-have):** Cleanup / future-proofing / technical debt

---

## 6. Output Constraints

### Report Style
- Use inline evidence citations: `file.go:12-14 — does X with \`method()\``
- **NEVER include "Inspected Files" or "Files Read" sections** (verbose, low-signal)
- **NO markdown code blocks in report body** (use inline format with single backticks for identifiers only)
- Keep it scannable: a reader should extract key risks in <5 minutes

---

## 7. Subagents (Optional Parallelization)

The agent **MAY** spawn subagents to improve throughput (limits defined per step or pipeline).

### Hard constraints for subagents
- **Same boundary:** subagents must treat `<target>` as a hard scope boundary (no reads/search outside `<target>`).
- **Same pre-reads:** subagents must follow the same mandatory pre-read rules if their task depends on that context.
- **No file writes:** subagents must not write or modify any files. They return findings to the main agent only.
- **Evidence-first:** every subagent output must include concrete evidence pointers (paths/symbols/keys).
- **No duplication:** each subagent must have a clearly scoped objective to avoid redundant work.
- **Advisory only:** subagent outputs are suggestions; the main agent must verify evidence before promoting items to P0/P1.

### Coordination rules
- The main agent must:
  - define each subagent's objective + expected output format (short),
  - consolidate and de-duplicate subagent findings,
  - resolve contradictions by re-checking evidence,
  - remain fully accountable for the final report quality and constraint compliance.

---

**End of GUARDRAILS.md**
