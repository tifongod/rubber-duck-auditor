# SKILL: rda-audit-pipeline

# Rubber Duck Audit — Full Pipeline (Isolated Steps)

## Role & Objective

You are a **MAANG Principal Backend Engineer** running a **full production-readiness audit pipeline** for a single service directory.

Your objective is to **execute all RDA steps end-to-end** (00 → 09 + executive summary), while ensuring:
- **hard scope boundary** (`<target>` only),
- **high audit quality** (evidence-based, risk-ranked, actionable),
- **iteration correctness** (delta-first, no copy-paste),
- **context isolation between steps** (to prevent prompt drift and cross-step leakage).

---

## Shared Rules (MANDATORY)

Before doing anything else, read and follow:
- `skills/_shared/rda-common-rules.md`
- `skills/_shared/REPORT_TEMPLATE.md`
- `skills/_shared/RISK_RUBRIC.md`
- `skills/_shared/PIPELINE.md`

If anything below conflicts with shared rules, shared rules win.

---

## Hard Guardrails (NON-NEGOTIABLE)

1. **Scope Boundary:**
    - `TARGET_SERVICE_PATH = <target>` (if not provided, use `.`)
    - You MUST NOT read, list, or analyze any files outside this path.

2. **Strictly Prohibited:**
    - Analyzing the entire monorepo.
    - Scanning/listing/reading files outside `TARGET_SERVICE_PATH`.
    - Producing an overview/map of the monorepo or enumerating other services outside `TARGET_SERVICE_PATH`.
    - Opening shared/common libraries outside `TARGET_SERVICE_PATH`.
    - If code inside the service imports modules outside boundary, you may ONLY record the dependency as **EXTERNAL DEP** and mark unknowns as **GAP**. You MUST NOT open external code.

3. **No Code Modifications:** Analysis + reports only.

4. **Missing Information:** If information is outside boundary or unavailable, record as **GAP** and continue.

---

## Inputs

- `<target>` (optional): service root directory. If not provided, use `.`.
    - Recommended example in monorepo: `<target> = services/some-service`

- `<mode>` (optional): step isolation mode.
    - `SUBAGENTS` (default): each step runs in a fresh isolated subagent with no memory.
    - `CLEARCTX`: run steps sequentially in one agent, but force a context reset between steps (`/clear`).

If `<mode>` is not provided, assume `SUBAGENTS`.

---

## Pipeline Isolation Model (CRITICAL)

### A) SUBAGENTS mode (default)
- Each audit step must run in a **fresh subagent** with:
    - the same `<target>` boundary,
    - the same shared rules,
    - no reliance on prior step chat context,
    - report output written to the deterministic path defined by that step.

### B) CLEARCTX mode
- Run steps sequentially in the same agent, but after each step:
    - perform `/clear`,
    - re-assert `<target>`,
    - re-load shared rules,
    - then run the next step.

If `/clear` is unavailable or ineffective, treat it as **GAP** and fall back to `SUBAGENTS`.

---

## Subagent Budget (MANDATORY CAP)

- You may spawn **up to 10 subagents total** per full pipeline run.
- Recommended allocation:
    - 1 subagent per step (00–09) and 1 for summary is too many (11), so:
        - Prefer: run 00–09 in 9 subagents, and run the executive summary in the *same* subagent as Step 09 (still isolated from earlier steps), OR
        - Run 00–08 in 9 subagents, run 09 + summary in the 10th.
- Within a step, you may spawn additional “micro-subagents” ONLY if you stay within the global cap of 10.

---

## Step Order & Expected Outputs

Run the following steps in order. Each step must write exactly one report to `<target>/docs/rda/reports/`:

1. Step 00 — Inventory
    - Output: `<target>/docs/rda/reports/00_service_inventory.md`

2. Step 01 — Architecture & Design
    - Output: `<target>/docs/rda/reports/01_architecture_design.md`

3. Step 02 — Configuration & Environment
    - Output: `<target>/docs/rda/reports/02_configuration_environment.md`

4. Step 03 — External Integrations
    - Output: `<target>/docs/rda/reports/03_external_integrations.md`

5. Step 04 — Data Handling
    - Output: `<target>/docs/rda/reports/04_data_handling.md`

6. Step 05 — Reliability & Failure Handling
    - Output: `<target>/docs/rda/reports/05_reliability_failure_handling.md`

7. Step 06 — Observability
    - Output: `<target>/docs/rda/reports/06_observability.md`

8. Step 07 — Testing, Delivery & Maintainability
    - Output: `<target>/docs/rda/reports/07_testing_delivery_maintainability.md`

9. Step 08 — Security & Privacy
    - Output: `<target>/docs/rda/reports/08_security_and_privacy.md`

10. Step 09 — Performance & Capacity
- Output: `<target>/docs/rda/reports/09_performance_and_capacity.md`

Then produce the executive summary (reads reports only):

11. Executive Summary
- Output: `<target>/docs/rda/summary/00_summary.md`

If any step skill is missing, the pipeline must:
- record the missing step as **GAP** (in the summary),
- continue with remaining steps.

---

## Mandatory Per-Step Execution Contract

For each step run (regardless of mode):

1. **Load shared rules**
    - Must explicitly follow `rda-common-rules.md`, `REPORT_TEMPLATE.md`, `RISK_RUBRIC.md`, `PIPELINE.md`.

2. **Respect boundary**
    - Only read files within `<target>`.

3. **Use the step’s report template**
    - Reports must follow `skills/_shared/REPORT_TEMPLATE.md`.
    - Do not add custom sections unless the step template requires them.

4. **Iteration correctness**
    - Do not copy-paste older reports.
    - Produce “Delta vs previous run” as required.
    - Re-check previously critical items and report status.

5. **Evidence discipline**
    - Every non-trivial claim must cite evidence (file path + short reference).
    - Otherwise mark **GAP**.

6. **No “Inspected Files” verbosity**
    - Do not include an “Inspected Files” list inside reports (template must reflect this rule).

---

## Pipeline Quality Gates (MANDATORY)

After each step completes, validate (lightweight, in-scope) that the produced report contains:
- Run Metadata (timestamp, git commit or GAP, change detection or GAP)
- Scope & guardrails confirmation
- Findings split into strengths/risks/gaps
- Action items with P0/P1/P2
- Delta vs previous run section

If the report is missing required sections:
- re-run the same step once (still respecting isolation),
- if it still fails, proceed and mark as **GAP** in the summary.

---

## Summary Generation Rules (STRICT)

The executive summary step must:
- read **only** markdown reports in `<target>/docs/rda/reports/`
- avoid any code/config inspection
- produce:
    - short verdict (Ready / Conditionally ready / Not ready)
    - top risks (P0/P1) and gaps
    - 100-point scorecard (per summary spec)
    - fastest remediation roadmap

If any step report is missing, the summary must explicitly list it under **Missing inputs** and reduce confidence accordingly.

---

## Completion Criteria

A full pipeline run is complete only if:
- Reports 00–09 exist OR missing ones are explicitly recorded as GAP in the summary
- Summary exists at `<target>/docs/rda/summary/00_summary.md`
- No scope boundary violations occurred
- No external code was opened
- Reports follow shared template constraints (including “no inspected files” verbosity)
