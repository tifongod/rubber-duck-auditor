# RDA Troubleshooting Guide

This document covers common failure modes when running Rubber Duck Auditor and how to resolve them.

---

## 1. Git Command Timeouts

**Symptom:** Audit step hangs or fails with timeout error when running git commands (e.g., `git log`, `git diff`).

**Root Cause:** Large repositories with extensive history may exceed the default Bash tool timeout (120 seconds / 2 minutes).

**Solution:**
- For very large repositories, consider using shallow clones or limiting git log depth
- If a specific git operation is timing out, the agent should mark it as GAP and continue rather than blocking the entire audit
- Check `skills/_shared/common-rules.md:40` for timeout guidance

**Prevention:**
- The Bash tool default timeout is 120,000ms (2 minutes) per `common-rules.md:35`
- For operations known to be slow, agents should add explicit timeout parameters or use narrower git queries

---

## 2. Missing Reports After Pipeline Run

**Symptom:** After running `/audit-pipeline`, one or more expected report files are missing from `docs/rda/reports/`.

**Root Cause:** One of several issues:
1. Subagent failed to complete the step and did not generate the report
2. Write tool failed silently (filesystem full, permission denied)
3. Agent skipped the step due to fast-path logic but didn't create updated report
4. Wave completion verification passed incorrectly

**Diagnosis:**
1. Check which reports are missing: `ls docs/rda/reports/`
2. Check pipeline Quality Gates output - did it verify report existence?
3. Check git status - were any reports staged but not committed?
4. Re-run the specific missing step manually to see error messages

**Solution:**
- **For single missing report:** Re-run that specific audit step skill (e.g., `/reliability`)
- **For multiple missing reports:** Re-run the entire pipeline with `/audit-pipeline`
- **If Write tool is failing:** Check filesystem permissions and available disk space

**Prevention:**
- Pipeline Quality Gates (see `skills/audit-pipeline/SKILL.md:301-313`) validate report existence and structure after each wave
- One retry is attempted automatically per `skills/audit-pipeline/SKILL.md:288-292`

---

## 3. Fast-Path P0 Re-Inspection Failures

**Symptom:** A report from a previous run had P0 (critical) findings, but the new run used fast-path optimization and did not re-inspect the P0 items. The new report still shows the same P0s without verification.

**Root Cause:** All audit step skills require P0 re-inspection even when using fast-path (no code changes detected). If the agent incorrectly applies fast-path logic without checking for prior P0s, critical issues may persist unverified.

**Diagnosis:**
1. Open the prior report and check section 4.2 for P0 risks
2. Open the new report and check if Delta section mentions "re-inspected P0 items"
3. Check if new report contains updated evidence for the P0 items

**Solution:**
- Manually re-run the affected audit step with explicit instructions: "Re-inspect all P0 items from prior report even if no code changes detected"
- Verify the new report includes fresh evidence that P0s are fixed, partially fixed, or not fixed

**Prevention:**
- Per `skills/_shared/GUARDRAILS.md:54-56`, agents MUST inspect code in current run even when reconciling with prior reports
- Fast-path optimization applies only when prior report has NO P0 items
- Pipeline Quality Gates should validate P0 re-inspection evidence (not yet implemented - see observability report P1 finding)

---

## 4. Absolute Paths in Reports (Non-Portable Reports)

**Symptom:** Reports contain absolute paths like `/Users/username/...` making them machine-specific and non-portable.

**Root Cause:** Reports generated before commit 65b6c58 (2026-02-06) did not enforce the `<run_root>` path hygiene rules introduced in `skills/_shared/common-rules.md:10-17`.

**Diagnosis:**
- Grep for absolute paths: `grep -r "/Users/" docs/rda/reports/` or `grep -r "/home/" docs/rda/reports/`

**Solution:**
- Re-run the affected audit steps to generate compliant reports with relative paths
- All reports should use paths relative to `<run_root>` (the repository root)

**Prevention:**
- New reports automatically comply with path hygiene rules per `common-rules.md:12-16`
- Reports from commit 65b6c58 onward are portable

---

## 5. ADR File Not Found Errors

**Symptom:** Multiple reports flag "GAP: ADR file missing at `docs/rda/ARCHITECTURAL_DECISIONS.md`"

**Root Cause:** Expected ADR file does not exist or is at wrong path.

**Diagnosis:**
- Check if file exists: `ls -la docs/rda/ARCHITECTURAL_DECISIONS.md`
- If file exists at `docs/ARCHITECTURAL_DECISIONS.md` (wrong path), it needs to be moved

**Solution:**
- Ensure ADR file exists at correct path: `docs/rda/ARCHITECTURAL_DECISIONS.md`
- File should contain architectural decisions in ADR format (see existing file for template)
- If no ADRs exist yet, create file with placeholder: "No formal ADRs yet. Decisions captured in commit messages."

**Prevention:**
- Keep ADR file updated at `docs/rda/ARCHITECTURAL_DECISIONS.md` per `common-rules.md:64-66`
- All audit steps expect this file to exist and will flag GAP if missing

---

## 6. Exec-Summary Fails to Find Reports

**Symptom:** The `/exec-summary` skill fails with "cannot find reports" or produces incomplete summary.

**Root Cause:** Exec-summary expects all 10 step reports (00-09) to exist at `docs/rda/reports/*.md`.

**Diagnosis:**
- Check report count: `ls docs/rda/reports/*.md | wc -l` (should be 10)
- Check which reports are missing: `ls docs/rda/reports/`

**Solution:**
- Run full audit pipeline first: `/audit-pipeline`
- Once all 10 step reports exist, run `/exec-summary`

**Prevention:**
- Always run `/audit-pipeline` before `/exec-summary`
- Exec-summary is reports-only and does not inspect source code per `skills/exec-summary/SKILL.md:5`

---

## 7. Report Contains Questions Instead of Findings

**Symptom:** A report contains questions like "What is the authentication mechanism?" instead of definitive findings.

**Root Cause:** Agent violated the "no questions in audit mode" rule from `skills/_shared/GUARDRAILS.md:89-93`.

**Diagnosis:**
- Search report for question marks: `grep "?" docs/rda/reports/<step>.md`

**Solution:**
- Re-run the audit step with explicit reminder: "Mark unknowns as GAP, do not ask questions"
- Edit the report to replace questions with GAP findings: "GAP: Authentication mechanism unknown — impacts security assessment. Missing: No auth config or middleware found."

**Prevention:**
- Per GUARDRAILS.md, agents MUST mark unknowns as GAP and continue rather than asking clarifying questions
- This is audit mode, not interactive mode

---

## 8. Subagent Budget Exceeded

**Symptom:** Pipeline fails with "subagent budget exceeded" error.

**Root Cause:** Audit pipeline has a cap of 10 subagents total across all waves per `skills/audit-pipeline/SKILL.md:84-89`.

**Diagnosis:**
- Check pipeline output for subagent spawn count
- Typical audit uses exactly 10 subagents (one per step 00-09)

**Solution:**
- This error indicates pipeline attempted to spawn more than 10 subagents
- Check if any step is spawning nested subagents (not expected in normal operation)
- Contact plugin maintainer if this occurs in normal usage

**Prevention:**
- Pipeline design assumes 1 subagent per audit step (10 total)
- Budget should not be exceeded in normal operation

---

## Getting Help

If none of these solutions resolve your issue:

1. **Check plugin version:** Ensure you're running the latest version of RDA
2. **Review recent changes:** Check if codebase changes introduced new failure modes
3. **File an issue:** Report at https://github.com/tifongod/rubber-duck-auditor/issues with:
   - Plugin version
   - Affected audit step(s)
   - Error messages or symptoms
   - Relevant report excerpts (redact sensitive data!)

---

<sub>Rubber Duck Auditor v0.1.7 | [GitHub](https://github.com/tifongod/rubber-duck-auditor)</sub>
