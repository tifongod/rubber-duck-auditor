# SKILL: config

# Configuration & Environment

## Role & Objective

You are a **FAANG Principal Backend Engineer** auditing production readiness of a single service. Your task in this step is to evaluate **runtime configuration, environment handling, secrets management, validation, and safe defaults**.

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
    - If code inside the service imports modules outside boundary, you may ONLY record the dependency (import path/config reference) as **EXTERNAL DEP** and mark unknowns as **GAP**. You MUST NOT open external code.

3. **No Code Modifications:** Analysis + reports only.

4. **Missing Information:** If information is outside boundary or unavailable, record as **GAP** and continue.

---

## Rerun / Anti-Copy Requirements (MANDATORY)

### A) Iteration Without Copy-Paste
- You MUST respect prior reports in `<target>/docs/rda/reports/**` as context and prior decisions (reports evolve over time).
- You MUST NOT copy prior report content verbatim.
- You MUST draft findings based on code inspection in this run, then reconcile with prior report(s) and produce a proper delta.

### B) Mandatory Run Metadata
Every report MUST include a "Run Metadata" section with:
- **Timestamp (UTC)**
- **Git commit hash** for `TARGET_SERVICE_PATH` (or GAP if unavailable)
- **Change Detection summary:**
    - Preferred: `git diff -- <target>` or `git status --porcelain -- <target>`
    - If git commands unavailable: record as GAP
- **Inspected Files (Top Evidence):** List 10–30 file paths that were actually opened (if the service is small, list all key files opened)

### C) Mandatory Delta Section
Every report MUST include "Delta vs previous run":
- If prior report existed: 3–10 bullets summarizing differences
- If no material changes: explicitly state "No material changes detected" and list 3–5 items re-checked
- If first run: "First run — no prior report to compare"

---

## Inputs

- `<target>` (optional): service root directory. If not provided, use `.`.
    - Recommended example for this service in a monorepo: `<target> = services/qql-runtime`

### Mandatory Context (Read FIRST)
1. **ARCH_DECISIONS_PATH:** `<target>/docs/rda/ARCHITECTURAL_DECISIONS.md`
    - Contains intentionally made architectural decisions with rationale.
    - Do NOT assume decisions are perfect or non-discussable.
    - You MUST consider these decisions during your audit.
    - If you believe a decision is risky/incorrect/outdated: explicitly call it out with evidence + reasoning and propose safer alternatives.
    - If file is missing/unreadable: record as **GAP** and proceed.

### Prior Reports Required (Context)
2. **Inventory report (recommended context):** `<target>/docs/rda/reports/01-service-inventory.md`
    - Use configuration surface and file inventory from this report.
    - If missing: record as **GAP** and scan likely config entrypoints and root `*.yaml`/`*.env*`/`*.json`/`*.toml` files.

---

## What to Inspect

Use file locations from inventory report where possible. Within `TARGET_SERVICE_PATH` ONLY:

### 1. Configuration Loading
- Identify the main config struct(s) and loading logic.
- Environment variable parsing mechanism.
- Config file formats (YAML/JSON/TOML) if used.
- Loading order and override precedence (env vs file vs flags).

### 2. Environment Handling
- `env.example` (or equivalent) — documented environment variables.
- How different environments are distinguished (dev/staging/prod).
- Feature flags or environment-specific behavior switches.

### 3. Secrets Management
- How secrets are loaded (env vars, files, external secret stores).
- Are secrets ever logged or exposed in errors?
- Credential rotation support (process/env reload, dynamic providers, or restart-only).

### 4. Validation & Defaults
- Are config values validated at startup?
- What happens with missing required config?
- Are defaults safe for production (timeouts, log level, debug flags, insecure TLS, localhost endpoints)?

### 5. Configuration at Runtime
- Is config immutable after startup or can it change?
- Hot-reload capabilities / watchers (if any).
- Config-driven behavior switches that impact safety.

---

## Method

Complete this checklist with evidence:

| # | Question | How to Verify |
|---|----------|---------------|
| 1 | Where is config loaded? | Find config struct(s) and parsing code |
| 2 | What env vars are required? | List env lookups with required/optional status |
| 3 | What are the default values? | Check struct tags, default assignments, fallbacks |
| 4 | Are defaults safe for production? | Identify defaults that could be dangerous in prod |
| 5 | Is config validated at startup? | Look for validation logic, tags, or explicit checks |
| 6 | What happens on invalid config? | Trace error handling for config failures |
| 7 | How are secrets handled? | Find credential loading; check exposure patterns |
| 8 | Are secrets logged anywhere? | Search log statements near credential usage and error paths |
| 9 | Is there env-specific config? | Check for dev/staging/prod differentiation |
| 10 | Can config change at runtime? | Look for hot-reload / watchers / dynamic sources |

For each answer, provide:
- **File path(s)** inspected
- **Short excerpt** or line reference as evidence
- **Assessment:** SAFE / CONCERN / RISK

---

## Architectural Decisions Awareness

When findings relate to intentional decisions:
- Reference `ARCH_DECISIONS_PATH` (quote short excerpt + section heading)
- Verify alignment between code/config and stated decision rationale
- If you see drift (code/config contradicts decisions): label as **DRIFT**, assess impact, propose corrective actions
- If you think the decision itself is poor: label as **DECISION RISK**, justify with evidence + production-readiness reasoning, propose safer alternatives

---

## Output

Write exactly ONE report:

**REPORT_PATH:** `<target>/docs/rda/reports/03-configuration-environment.md`

### Report Structure (in this exact order)

# Configuration & Environment Report

## 0. Run Metadata
- **Timestamp (UTC):** <YYYY-MM-DD HH:MM:SS UTC>
- **Git Commit:** <hash or GAP>
- **Change Detection:** <summary or GAP>
- **Inspected Files (Top Evidence):** <list 10-30 files>

## 1. Context Recap
- <2-5 bullets on service config needs from inventory report>
- <Include relevant decision context from ARCH_DECISIONS_PATH>

## 2. Scope & Guardrails Confirmation
- Confirm: inspection limited to `TARGET_SERVICE_PATH`
- Confirm: no external code was opened
- List any EXTERNAL DEPs recorded

## 3. Evidence

### 3.1 Configuration Structure
| Config Section | Struct/File | Required | Has Default | Default Value |
|----------------|-------------|----------|-------------|---------------|
| ... | ... | ... | ... | ... |

### 3.2 Environment Variables
| Variable | Required | Default | Safe for Prod? | Notes |
|----------|----------|---------|----------------|-------|
| ... | ... | ... | ... | ... |

### 3.3 Secrets Handling
| Secret Type | Loading Method | Exposure Risk | Evidence |
|-------------|----------------|---------------|----------|
| ... | ... | ... | ... |

### 3.4 Validation Analysis
| Validation Type | Implemented | Evidence |
|-----------------|-------------|----------|
| Struct tags | ... | ... |
| Startup validation | ... | ... |
| Runtime validation | ... | ... |

### 3.5 Environment Differentiation
| Environment | Detection Method | Config Differences |
|-------------|------------------|-------------------|
| ... | ... | ... |

## 4. Findings

### 4.1 Strengths
- <Bullet points with evidence references>

### 4.2 Risks
- <Bullet points with evidence, severity, impact>

### 4.3 Gaps
- <Items that could not be determined>

## 5. Action Items

### P0 (Critical)
- <Item> | Impact: <X> | Verify: <how>

### P1 (Important)
- <Item> | Impact: <X> | Verify: <how>

### P2 (Nice-to-have)
- <Item> | Impact: <X> | Verify: <how>

## 6. Delta vs Previous Run
- <If prior report existed: 3-10 bullets on differences>
- <If first run: "First run — no prior report to compare">
- <If no material changes: "No material changes detected" + list 3-5 re-checked items>

---

## Prioritization Rubric
- **P0:** Production risk / correctness / security / data loss / outage class
- **P1:** Significant reliability / operability / maintainability improvements
- **P2:** Nice-to-have / cleanup / future-proofing

---

## No Speculation Rule
All assertions MUST be tied to evidence (file path + excerpt/line reference). If evidence is unavailable, label as **INFERRED** or **GAP** and continue.

---

## Completion Criteria

Before submitting:
1. Run Metadata section is complete with timestamp and file list
2. All 10 checklist questions answered with evidence + assessment
3. All env vars documented with required/optional status (or explicitly marked GAP)
4. Secrets handling analyzed for exposure risks
5. Default values assessed for production safety
6. No files outside `TARGET_SERVICE_PATH` were opened
7. Delta section present (even if first run)
8. Report written to correct path
