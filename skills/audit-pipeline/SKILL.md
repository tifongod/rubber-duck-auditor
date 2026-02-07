# SKILL: audit-pipeline

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
- `skills/_shared/common-rules.md`
- `skills/_shared/GUARDRAILS.md`
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
- Run steps sequentially in **wave order** (see Execution Model below) in the same agent, but after each step:
    - perform `/clear`,
    - re-assert `<target>`,
    - re-load shared rules,
    - then run the next step.
- CLEARCTX mode cannot parallelize within waves, so it will be slower than SUBAGENTS mode.
- You must still wait for reports to be written between waves (verify with Read tool before proceeding).

If `/clear` is unavailable or ineffective, treat it as **GAP** and fall back to `SUBAGENTS`.

---

## Subagent Budget (MANDATORY CAP)

- You may spawn **up to 11 subagents total** per full pipeline run.
- With wave-based execution (see below), you will spawn **at most 3 concurrent subagents per wave**.
- Total subagents needed: 10 steps + 1 summary = 11.
- Within a step, you may spawn additional "micro-subagents" ONLY if you stay within the global cap of 11.

---

## Execution Model: Wave-Based Sequencing (CRITICAL)

Steps MUST be executed in **dependency waves** to ensure later steps can consume earlier reports.

**IMPORTANT:** Do NOT run all steps in parallel. Steps have explicit dependencies (defined in `PIPELINE.md`). Running them in parallel causes:
- Missing context (later steps can't read non-existent reports)
- Increased GAPs (redundant discovery work)
- Lower quality findings (inconsistent conclusions)

### Wave Execution Rules

1. **Execute one wave at a time** — all steps in a wave can run in parallel (using subagents).
2. **Wait for wave completion** — before starting the next wave, verify all expected reports exist.
3. **Report verification** — after each wave, use Bash `ls` or Read tool to confirm reports were created.
4. **Handle failures** — if a step fails to produce a report, record as **GAP** and continue (per Pipeline Quality Gates).

### Dependency Waves (Canonical Order)

**Wave 1: Foundation**
- Step 00 (Inventory) — no dependencies
- Output: `00_service_inventory.md`
- Must complete before Wave 2

**Wave 2: Core Systems** (parallel)
- Step 01 (Architecture & Design) — depends on: 00
- Step 02 (Configuration & Environment) — depends on: 00
- Outputs: `01_architecture_design.md`, `02_configuration_environment.md`
- Must complete before Wave 3

**Wave 3: External Boundaries**
- Step 03 (External Integrations) — depends on: 00, 02
- Output: `03_external_integrations.md`
- Must complete before Wave 4

**Wave 4: Data & Observability** (parallel)
- Step 04 (Data Handling) — depends on: 00, 03
- Step 06 (Observability) — depends on: 00, 01
- Outputs: `04_data_handling.md`, `06_observability.md`
- Must complete before Wave 5

**Wave 5: Reliability**
- Step 05 (Reliability & Failure Handling) — depends on: 00, 03, 04
- Output: `05_reliability_failure_handling.md`
- Must complete before Wave 6

**Wave 6: Quality & Security** (parallel)
- Step 07 (Testing, Delivery & Maintainability) — depends on: 00–06
- Step 08 (Security & Privacy) — depends on: 00, 02, 03, 06
- Step 09 (Performance & Capacity) — depends on: 00, 03, 04, 05, 06
- Outputs: `07_testing_delivery_maintainability.md`, `08_security_privacy.md`, `09_performance_capacity.md`
- Must complete before Wave 7

**Wave 7: Summary**
- Executive Summary — depends on: all reports 00–09
- Output: `<target>/docs/rda/summary/00_summary.md`

### Wave Execution Example (SUBAGENTS mode)

```
1. Spawn subagent for Step 00 → wait for completion → verify 00_service_inventory.md exists

2. Spawn 2 subagents in parallel:
   - Step 01 (Architecture)
   - Step 02 (Configuration)
   → wait for both → verify 01_*.md and 02_*.md exist

3. Spawn subagent for Step 03 → wait → verify 03_*.md exists

4. Spawn 2 subagents in parallel:
   - Step 04 (Data)
   - Step 06 (Observability)
   → wait for both → verify 04_*.md and 06_*.md exist

5. Spawn subagent for Step 05 → wait → verify 05_*.md exists

6. Spawn 3 subagents in parallel:
   - Step 07 (Testing)
   - Step 08 (Security)
   - Step 09 (Performance)
   → wait for all three → verify 07_*.md, 08_*.md, 09_*.md exist

7. Spawn subagent for Executive Summary → wait → verify summary/00_summary.md exists
```

**Total subagents:** 11 (1+2+1+2+1+3+1)

---

## Step Order & Expected Outputs

The following steps MUST be executed in wave order (see Dependency Waves above). Each step must write exactly one report to `<target>/docs/rda/reports/`:

**Wave 1:**
1. Step 00 — Inventory
    - Skill: `inventory`
    - Output: `<target>/docs/rda/reports/00_service_inventory.md`

**Wave 2 (parallel):**
2. Step 01 — Architecture & Design
    - Skill: `architecture`
    - Output: `<target>/docs/rda/reports/01_architecture_design.md`

3. Step 02 — Configuration & Environment
    - Skill: `config`
    - Output: `<target>/docs/rda/reports/02_configuration_environment.md`

**Wave 3:**
4. Step 03 — External Integrations
    - Skill: `integrations`
    - Output: `<target>/docs/rda/reports/03_external_integrations.md`

**Wave 4 (parallel):**
5. Step 04 — Data Handling
    - Skill: `data`
    - Output: `<target>/docs/rda/reports/04_data_handling.md`

6. Step 06 — Observability
    - Skill: `observability`
    - Output: `<target>/docs/rda/reports/06_observability.md`

**Wave 5:**
7. Step 05 — Reliability & Failure Handling
    - Skill: `reliability`
    - Output: `<target>/docs/rda/reports/05_reliability_failure_handling.md`

**Wave 6 (parallel):**
8. Step 07 — Testing, Delivery & Maintainability
    - Skill: `quality`
    - Output: `<target>/docs/rda/reports/07_testing_delivery_maintainability.md`

9. Step 08 — Security & Privacy
    - Skill: `security`
    - Output: `<target>/docs/rda/reports/08_security_privacy.md`

10. Step 09 — Performance & Capacity
    - Skill: `performance`
    - Output: `<target>/docs/rda/reports/09_performance_capacity.md`

**Wave 7:**
11. Executive Summary
    - Skill: `exec-summary`
    - Output: `<target>/docs/rda/summary/00_summary.md`
    - Reads reports only (no code inspection)

If any step skill is missing, the pipeline must:
- record the missing step as **GAP** (in the summary),
- continue with remaining steps.

---

## Mandatory Per-Step Execution Contract

For each step run (regardless of mode):

1. **Load shared rules**
    - Must explicitly follow `common-rules.md`, `REPORT_TEMPLATE.md`, `RISK_RUBRIC.md`, `PIPELINE.md`.

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

## Wave Completion Verification (MANDATORY)

After each wave completes, you MUST verify all expected reports exist before proceeding to the next wave:

1. **Check report existence:**
   - Use Bash `ls <target>/docs/rda/reports/` to list existing reports
   - OR use Glob tool: `<target>/docs/rda/reports/*.md`
   - OR use Read tool to check specific expected paths

2. **Expected reports per wave:**
   - Wave 1: `00_service_inventory.md`
   - Wave 2: `01_architecture_design.md`, `02_configuration_environment.md`
   - Wave 3: `03_external_integrations.md`
   - Wave 4: `04_data_handling.md`, `06_observability.md`
   - Wave 5: `05_reliability_failure_handling.md`
   - Wave 6: `07_testing_delivery_maintainability.md`, `08_security_privacy.md`, `09_performance_capacity.md`
   - Wave 7: `summary/00_summary.md`

3. **Handling missing reports:**
   - If a report is missing after subagent completion, check the subagent output for errors
   - Retry the step once (respecting isolation)
   - If still missing, record as **GAP** and continue to next wave
   - Document the missing report in the final summary

4. **Do NOT proceed to next wave until:**
   - All expected reports from current wave exist OR are explicitly marked as GAP
   - This ensures later steps have the context they need

---

## Pipeline Quality Gates (MANDATORY)

After each wave completes and reports are verified to exist, validate (lightweight, in-scope) that each produced report contains:
- Run Metadata (timestamp, git commit or GAP)
- Scope & guardrails confirmation
- Findings split into strengths/risks/gaps
- Action items with P0/P1/P2
- Delta vs previous run section

If a report is missing required sections:
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
- All 7 waves have been executed in sequence
- Reports 00–09 exist OR missing ones are explicitly recorded as GAP in the summary
- Summary exists at `<target>/docs/rda/summary/00_summary.md`
- Wave verification was performed between each wave (reports checked before proceeding)
- No scope boundary violations occurred
- No external code was opened
- Reports follow shared template constraints (including "no inspected files" verbosity)
- Total subagent count did not exceed 11
