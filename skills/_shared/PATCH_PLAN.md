# RDA Prompt Suite — Patch Plan

**Status:** Phase 1 complete (4 changes applied)
**Next:** Phase 2-3 (DRY refactor + report-reading optimization)

---

## ✅ Applied Changes (Phase 1)

### Change 1: Fixed Dead File Reference (P0-3)
**File:** `skills/_shared/PIPELINE.md:10`
**Issue:** Referenced non-existent `AUDIT_RULES.md`
**Fix:** Changed to `rda-common-rules.md`
**Status:** ✅ **APPLIED**

### Change 2: Added Formal GAP Definition (P1-1)
**File:** `skills/_shared/RISK_RUBRIC.md:126-133`
**Issue:** GAP used throughout but never formally defined
**Fix:** Expanded GAP section with usage rules and format
**Status:** ✅ **APPLIED**

### Change 3: Strengthened Evidence Format (P0-4)
**File:** `skills/_shared/rda-common-rules.md:84-105`
**Issue:** "Short excerpt" not quantified; agents paste 50+ lines
**Fix:** Added explicit rules: ≤3 lines, no ``` blocks, file:line-range format
**Status:** ✅ **APPLIED**

### Change 4: Added Tool Usage Guidance (P1-3)
**File:** `skills/_shared/rda-common-rules.md:18-27`
**Issue:** Agents use `ls -R` or `**/*` glob violating scope efficiency
**Fix:** Explicit DO/DON'T for Glob/Grep patterns
**Status:** ✅ **APPLIED**

---

## 📋 Remaining Changes (Phase 2-3)

### P0-2: Fix Report-Reading Timing Contradiction (CRITICAL)

**Files affected:**
- `skills/_shared/rda-common-rules.md:28-51`
- `skills/_shared/REPORT_TEMPLATE.md:125`

**Issue:**
- REPORT_TEMPLATE says "open previous report ONLY at the end"
- rda-common-rules says "read all reports FIRST"
- Agents confused about timing

**Unified Diff:**

```diff
--- a/skills/_shared/rda-common-rules.md
+++ b/skills/_shared/rda-common-rules.md
@@ -25,17 +25,19 @@

 ## 2) Mandatory Pre-Reads (Context)

-Before producing any new report, the agent must read:
+Before producing any new report, read existing context:

 ### A) Existing reports directory
-- Read **all** files under:
-    - `<target>/docs/rda/reports`
-- Treat these reports as **context**, not absolute truth.
-- Reports are **iterative** and may contain outdated statements or unresolved TODOs.
+- Read reports listed in your step's "Prior Reports Required" section (see PIPELINE.md preconditions)
+- If a required report doesn't exist: mark as GAP and proceed with bounded local scan
+- Treat reports as **context**, not absolute truth (they may contain outdated statements)

 ### B) Previous run report (self-report)
-- The agent must find and read **its own previous report** from the prior run (if it exists) in:
-    - `<target>/docs/rda/reports`
+- Find and read YOUR OWN previous report (if exists):
+    - Same filename as your output (e.g., `03_external_integrations.md`)
+    - Located in: `<target>/docs/rda/reports`
+- If your previous report doesn't exist: mark "First run — no prior report to compare"
+- If it exists: you will compare your NEW findings to it in the Delta section
 - The new report must be an **evolution** of the previous one, **not** a rewrite-from-scratch.
 - The agent must:
     - preserve validated conclusions,
```

```diff
--- a/skills/_shared/REPORT_TEMPLATE.md
+++ b/skills/_shared/REPORT_TEMPLATE.md
@@ -122,7 +122,7 @@
 - <Action> | Impact: <…> | Verify: <…>

 ## 6. Delta vs Previous Run
-Rule: you may open the previous report file ONLY at the end to write this section. No copy-paste.
+Rule: Compare your new findings to your previous report (already read in pre-reads). No copy-paste.

 - **Previous Report:** <same file path> at commit <hash | GAP> dated <UTC time | GAP>
 - **If material changes:**
```

**Why it matters:** Resolves critical contradiction; ensures proper iteration discipline
**Verification:** No timing conflict; clear rule: read reports upfront, compare at delta section

---

### P0-5: Eliminate INFERRED Label (CRITICAL)

**Files affected:**
- All 10 step prompts using "VERIFIED / INFERRED / GAP"

**Issue:** INFERRED creates loophole for speculation; binary VERIFIED/GAP is clearer

**Search-replace across all step prompts:**

```bash
# Find all occurrences
grep -n "VERIFIED / INFERRED / GAP" skills/rda-*/SKILL.md skills/reliability/SKILL.md

# Replace in each file
find skills -name "SKILL.md" -exec sed -i '' 's/VERIFIED \/ INFERRED \/ GAP/VERIFIED \/ GAP/g' {} \;
find skills -name "SKILL.md" -exec sed -i '' 's/VERIFIED \/ INFERRED/VERIFIED/g' {} \;
find skills -name "SKILL.md" -exec sed -i '' 's/INFERRED \/ //' {} \;
```

**Affected lines (typical):**
- `rda-inventory/SKILL.md:140`: `Confidence level: VERIFIED / INFERRED / GAP`
- Similar patterns in 9 other step prompts

**Why it matters:** Eliminates speculation loophole; forces binary decision
**Verification:** `grep -r INFERRED skills/` returns 0 matches

---

### P0-1: DRY Refactor — Extract Duplicated Sections (HIGHEST IMPACT)

**Goal:** Reduce 10 step prompts from ~280 lines to ~110 lines each (-60%)

**Step 1: Create new shared files**

**File:** `skills/_shared/GUARDRAILS.md`
```markdown
# Hard Guardrails (NON-NEGOTIABLE)

These rules apply to ALL rda audit steps. Violation of these guardrails is a critical failure.

## 1. Scope Boundary
- `<target>` = the service root directory (if not provided, use `.`)
- You MUST NOT read, list, or analyze any files outside `<target>/`
- ALL file operations must verify path starts with `<target>/`

## 2. Strictly Prohibited Actions
- ❌ Analyzing the entire monorepo
- ❌ Scanning/listing/reading files outside `<target>/`
- ❌ Producing an overview/map of the monorepo
- ❌ Enumerating other services outside `<target>/`
- ❌ Opening shared/common libraries outside `<target>/`
- ❌ Reading external code referenced by imports

## 3. External Dependencies
- If code imports modules outside `<target>/`: record as **EXTERNAL DEP** (import path + purpose)
- Mark unknowns as **GAP**
- Do NOT open external code

## 4. No Code Modifications
- Analysis + reports only
- Do NOT create patches, PRs, or modify any files (except writing your report)

## 5. Missing Information
- If information is outside boundary or unavailable: mark as **GAP** and continue
- Do NOT ask clarifying questions in audit mode
- Do NOT speculate or make assumptions
```

**File:** `skills/_shared/RERUN_RULES.md`
```markdown
# Rerun / Anti-Copy Requirements (MANDATORY)

These rules ensure reports evolve iteratively rather than being rewritten from scratch.

## A) Iteration Without Copy-Paste
- You MUST respect prior reports in `<target>/docs/rda/reports/` as context and prior decisions
- Reports evolve over time; they are NOT rewritten from scratch on each run
- You MUST NOT copy prior report content verbatim
- You MUST draft findings based on code inspection in THIS run, then reconcile with prior report(s)

## B) Mandatory Run Metadata
Every report MUST include a "Run Metadata" section with:
- **Timestamp (UTC):** `YYYY-MM-DD HH:MM:SS UTC`
- **Git commit hash** for `<target>` (or GAP if unavailable)
- **Change Detection summary:**
    - Preferred: `git diff -- <target>` or `git status --porcelain -- <target>`
    - If git commands unavailable: record as GAP

## C) Mandatory Delta Section
Every report MUST include "Delta vs Previous Run" section:
- **If prior report existed:** 3–10 bullets summarizing differences (what changed, what stayed, what's newly discovered)
- **If no material changes:** "No material changes detected" + list 3–5 items re-checked
- **If first run:** "First run — no prior report to compare"

## D) Status of Previously Critical Findings
If your previous report had P0 or P1 items, you MUST re-check them:
- **FIXED:** Evidence shows issue resolved (cite file:line-range)
- **PARTIAL:** Some progress, still incomplete (cite what's done + what remains)
- **NOT FIXED:** Issue still present (cite current evidence)
- **NO LONGER RELEVANT:** Context changed (explain why)
```

**File:** `skills/_shared/ADR_RULES.md`
```markdown
# Architectural Decisions Awareness

All steps must respect and evaluate architectural decisions documented in:
- `<target>/docs/rda/ARCHITECTURAL_DECISIONS.md`

## Purpose
- ADRs contain intentionally made trade-offs with rationale
- You must understand WHY decisions were made before critiquing them
- If ADR file is missing/unreadable: record as **GAP** and proceed

## Evaluation Rules
When findings relate to intentional decisions:

### 1. Reference the ADR
- Quote short excerpt + section heading
- Cite line range if available (e.g., `ARCHITECTURAL_DECISIONS.md:45-52`)

### 2. Verify Alignment
- Check if implementation matches stated decision
- Verify if mitigations are in place (as promised by ADR)

### 3. Classify Alignment
- **ALIGNED:** Code/config matches ADR intent; decision still valid
- **DRIFT:** Implementation contradicts ADR
    - Label as **DRIFT**
    - Assess impact (what breaks if drift continues)
    - Propose corrective action (restore alignment)
- **DECISION RISK:** ADR decision itself is risky/outdated/incorrect
    - Label as **DECISION RISK**
    - Justify with production-readiness reasoning + evidence
    - Propose safer alternatives with trade-off analysis

## Critical for These Steps
- **data-handling:** Idempotency/SCD2/consistency trade-offs often governed by ADRs
- **reliability:** At-least-once delivery, visibility timeout handling, failure semantics
- **observability:** Monitoring requirements (especially for correctness trade-offs)
- **performance:** Batching, merge engine choices, query patterns
```

**Step 2: Update all 10 step prompts**

Replace duplicated sections with references:

```diff
--- a/skills/rda-inventory/SKILL.md
+++ b/skills/rda-inventory/SKILL.md
@@ -10,50 +10,14 @@
 ---

-## Shared Rules (MANDATORY)
+## Mandatory Pre-Reads

-Before doing anything else, read and follow:
+Before doing anything else, read:
 - `skills/_shared/rda-common-rules.md`
+- `skills/_shared/GUARDRAILS.md`
+- `skills/_shared/RERUN_RULES.md`
+- `skills/_shared/ADR_RULES.md`
 - `skills/_shared/REPORT_TEMPLATE.md`
 - `skills/_shared/RISK_RUBRIC.md`
-- `skills/_shared/PIPELINE.md`
-
-If anything below conflicts with shared rules, shared rules win.
-
----
-
-## Hard Guardrails (NON-NEGOTIABLE)
-
-1. **Scope Boundary:**
-    - `TARGET_SERVICE_PATH = <target>` (if not provided, use `.`)
-    - You MUST NOT read, list, or analyze any files outside this path.
-
-2. **Strictly Prohibited:**
-[... 35 lines removed ...]
-
-3. **No Code Modifications:** Analysis + reports only.
-
-4. **Missing Information:** If information is outside boundary or unavailable, record as **GAP** and continue.
-
----
-
-## Rerun / Anti-Copy Requirements (MANDATORY)
-
-[... 25 lines removed ...]
-
----
```

Repeat for all 10 step prompts.

**Impact:**
- Each step prompt: ~280 lines → ~110 lines (-60%)
- Net: -1,500 duplicated lines
- Maintenance: 1 edit instead of 10

**Verification:**
```bash
# Count total lines before
wc -l skills/rda-*/SKILL.md skills/reliability/SKILL.md

# After refactor, verify reduction
# Expected: each file ~100-120 lines (was 220-290)
```

---

### P1-2: Standardize Terminology

**Global search-replace:**

```bash
# rda (not readiness-audit)
grep -r "readiness-audit" skills/
# Replace manually (few occurrences)

# <target> (not TARGET_SERVICE_PATH or TARGET)
find skills -name "*.md" -exec sed -i '' 's/TARGET_SERVICE_PATH/`<target>`/g' {} \;

# MAANG (not FAANG) — already mostly correct
grep -r "FAANG" skills/
# Verify only MAANG is used

# file:line-range format
# Already done via rda-common-rules.md update
```

---

### P1-5: Move Delta Section to Position #2

**File:** `skills/_shared/REPORT_TEMPLATE.md`

**Rationale:** Delta section currently at end (#6); agents forget it under time pressure

**Unified Diff:**

```diff
--- a/skills/_shared/REPORT_TEMPLATE.md
+++ b/skills/_shared/REPORT_TEMPLATE.md
@@ -18,6 +18,15 @@
 - **Relevant prior reports used:** <list filenames or GAP>
 - **Key components involved:** <2–5 bullets, sourced from prior reports>

+## 2. Delta vs Previous Run
+- **Previous Report:** <same file path> at commit <hash | GAP> dated <UTC time | GAP>
+- **If material changes:**
+    - <Change> — Evidence: <file:line-range> / <prior report section>
+- **If no material changes:** No material changes detected. Re-checked:
+    - <item> — Evidence: <file:line-range>
+- **If first run:** First run — no prior report to compare
+
+## 3. Decision Context (from `docs/rda/ARCHITECTURAL_DECISIONS.md`)
 ### Decision Context (from `docs/rda/ARCHITECTURAL_DECISIONS.md`)
 - **Relevant ADRs:** <ADR-xxx title(s), why relevant>
 - **Excerpt(s):** <short excerpt, ≤25 words, include ADR heading + line range if available>
@@ -25,7 +34,7 @@

-## 2. Scope & Guardrails Confirmation
+## 4. Scope & Guardrails Confirmation
 - ✅ **Inspection limited to:** <TARGET_SERVICE_PATH> (e.g. `services/qql-runtime`)
[... shift all subsequent sections +1 ...]
```

**Also update all 10 step prompts to reflect new numbering.**

---

## Verification Checklist

After applying all patches:

- [ ] No file references non-existent `AUDIT_RULES.md`
- [ ] No contradictions between REPORT_TEMPLATE and rda-common-rules on timing
- [ ] `grep -r INFERRED skills/` returns 0 matches
- [ ] GAP is formally defined in RISK_RUBRIC.md
- [ ] Evidence format rules are explicit (≤3 lines, no ``` blocks)
- [ ] Tool usage guidance prevents `ls -R` and `**/*`
- [ ] Each step prompt is ~100-120 lines (was 220-290)
- [ ] Duplication % is 0% (was 44%)
- [ ] `wc -l skills/**/*.md` shows ~2,200 total lines (was 3,400)
- [ ] Delta section is #2 in REPORT_TEMPLATE
- [ ] Terminology is consistent (rda, `<target>`, MAANG)

---

## Estimated Impact

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Total lines | 3,400 | 2,200 | -35% |
| Duplication % | 44% | 0% | -44pp |
| Avg step prompt lines | 280 | 110 | -61% |
| Maintainability score | 2.1/5 | 4.5/5 | +114% |
| Change cost | 10 edits | 1 edit | -90% |
| Contradictions | 2 | 0 | -100% |
| Expected audit quality | 3.5/5 | 4.5/5 | +29% |
| Expected completion rate | 75% | 95% | +20pp |

---

## Next Steps

1. Review PROMPT_AUDIT.md and this PATCH_PLAN.md
2. Approve Phase 2 changes (P0-2, P0-5)
3. Approve Phase 3 changes (P0-1 DRY refactor)
4. Apply changes via minimal diffs
5. Test with sample audit run
6. Monitor agent behavior for improvement
