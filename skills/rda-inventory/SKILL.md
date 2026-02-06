# SKILL: inventory

# Service Inventory & Boundaries

## Role & Objective

You are a **MAANG Principal Backend Engineer** auditing production readiness of a single service. Your task in this step is to produce a comprehensive **service inventory**: boundaries, entrypoints, runtime model, code structure, and key file locations.

This inventory will serve as the foundation for all subsequent audit steps (but this skill must be runnable independently).

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
- Read all prior reports under:
    - `<target>/docs/rda/reports/**`
- Treat them as context, not absolute truth.
- Read your previous version of this report (if exists) to ensure continuity and to verify prior critical items.

---

## What to Inspect

Within `TARGET_SERVICE_PATH` ONLY, systematically examine:

### 1. Entrypoints & Runtime Model
- `cmd/` directory — identify all `main.go` files and what each binary does
- Determine runtime model: API server, worker, batch job, CLI tool, or hybrid

### 2. Code Structure
- `internal/` directory layout — enumerate top-level packages
- `configs/` — configuration file templates
- `db/` — database migration structure
- Root files: `go.mod`, `go.sum`, `README.md`, `env.example` (or equivalents)

### 3. Key Package Identification
For each package under `internal/`, identify:
- Purpose (app, config, handler, repository, usecase, queue, models, middleware, helpers)
- Key files and approximate line counts / size signals
- Any obvious patterns (DI, ports/adapters, clean architecture)

### 4. External Dependencies (Record Only)
- Parse `go.mod` for direct dependencies
- Identify imports in key files that reference paths outside `TARGET_SERVICE_PATH`
- Record each as **EXTERNAL DEP** with import path — DO NOT open external code

### 5. Configuration Surface
- Identify config struct locations
- List environment variables referenced
- Note any secrets/credentials handling patterns

---

## Method

Complete this checklist with evidence:

| # | Question | How to Verify |
|---|----------|---------------|
| 1 | What binaries does the service produce? | List all entrypoints (e.g., `main.go` under `cmd/` or equivalents) with their purpose |
| 2 | What is the primary runtime model? | Analyze main entrypoint: HTTP server, queue consumer, batch job? |
| 3 | What is the package structure? | List `internal/` subdirectories with 1-line purpose each |
| 4 | What are the key domain models? | Examine models/entities packages for core entities |
| 5 | What external systems does it integrate with? | Check imports/config for queue/db/cache references |
| 6 | What infrastructure does it require? | Parse configs/docs for required services (DB, queue, cache) |
| 7 | What configuration is required? | List all env vars from config parsing |
| 8 | Are there database migrations? | Check `db/` and/or migrations tooling |
| 9 | What is the test structure? | Identify `*_test.go` files and patterns |
| 10 | What documentation exists? | List markdown files with their scope |

For each answer, provide:
- **File path(s)** inspected
- **Short excerpt** or line reference as evidence
- **Confidence level:** VERIFIED / INFERRED / GAP

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

**REPORT_PATH:** `<target>/docs/rda/reports/01-service-inventory.md`

### Report Structure (in this exact order)

# Service Inventory Report

## 0. Run Metadata
- **Timestamp (UTC):** <YYYY-MM-DD HH:MM:SS UTC>
- **Git Commit:** <hash or GAP>
- **Change Detection:** <summary or GAP>
- **Inspected Files (Top Evidence):** <list 10-30 files>

## 1. Context Recap
- <2-5 bullets summarizing service purpose and scope>
- <Include relevant context from ARCH_DECISIONS_PATH if applicable>

## 2. Scope & Guardrails Confirmation
- Confirm: inspection limited to `TARGET_SERVICE_PATH`
- Confirm: no external code was opened
- List any EXTERNAL DEPs recorded

## 3. Evidence

### 3.1 Entrypoints
| Binary | Path | Purpose | Runtime Model |
|--------|------|---------|---------------|
| ... | ... | ... | ... |

### 3.2 Package Structure
| Package | Path | Purpose | Key Files |
|---------|------|---------|-----------|
| ... | ... | ... | ... |

### 3.3 External Dependencies
| Dependency | Type | Import Path | Notes |
|------------|------|-------------|-------|
| ... | ... | ... | ... |

### 3.4 Configuration Surface
| Config Area | Source File | Env Vars / Keys |
|-------------|-------------|-----------------|
| ... | ... | ... |

### 3.5 Infrastructure Requirements
| System | Purpose | Evidence |
|--------|---------|----------|
| ... | ... | ... |

### 3.6 File Inventory (Key Files)
- <List critical files with approximate size/complexity signals and purpose>

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
2. All 10 checklist questions answered with evidence + confidence
3. No files outside `TARGET_SERVICE_PATH` were opened
4. External dependencies recorded but not inspected
5. Delta section present (even if first run)
6. Report written to correct path
