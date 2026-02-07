# SKILL: inventory

# Service Inventory & Boundaries

## Role & Objective

You are a **MAANG Principal Backend Engineer** auditing production readiness of a single service. Your task in this step is to produce a comprehensive **service inventory**: boundaries, entrypoints, runtime model, code structure, and key file locations.

This inventory will serve as the foundation for all subsequent audit steps (but this skill must be runnable independently).

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

## ⚠️ Concurrent Execution Warning

**DO NOT run this skill concurrently with itself.** Running the same step multiple times in parallel will cause last-write-wins behavior where the second run silently overwrites the first run's report with no conflict detection. This results in silent data loss.

If you need to re-run this step, wait for the current execution to complete before starting a new one.

(This warning does not apply to pipeline-orchestrated runs, which use wave-based sequencing to prevent concurrent writes.)

---

## Inputs

- `<target>` (optional): service root directory. If not provided, use `.`.
    - Recommended example for this service in a monorepo: `<target> = services/some-service`

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

### Fast-Path for Unchanged Target

If **both** of the following are true:
1. `git diff -- "${target}"` returns empty (no uncommitted changes)
2. `git status --porcelain -- "${target}"` returns empty (no untracked files)

Then you MAY take the fast-path:
- Read the previous report for this step (if exists)
- Confirm it has a "No material changes" delta section OR recent critical items
- Update **only** the Run Metadata:
  - New timestamp (UTC)
  - Same commit hash (if unchanged)
- Add note in Delta section: "Fast-path: no code changes detected since last run (commit XXXXX). Prior findings re-validated without deep re-inspection."
- **EXCEPTION:** If prior report has **P0 items marked "not fixed"**, you MUST re-inspect those specific areas (no fast-path for P0s)

If the prior report does NOT exist, or has P0s, or you are uncertain: skip fast-path and perform full inspection.

**Rationale:** Unchanged code → unchanged findings. Fast-path saves ~70% of agent time on reruns for stable codebases.

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

## Tool Usage (MANDATORY)

**DO use these tools:**
- ✅ **Glob** for file patterns: `**/*.go`, `internal/*/`, `cmd/*/main.go`, `**/*.{yaml,yml}`
- ✅ **Grep** for content search:
  - `pattern="timeout"`, `--type go`, `--glob "*.yaml"`
  - Use `output_mode="files_with_matches"` for listing, `output_mode="content"` for excerpts
- ✅ **Read** for known file paths (always prefer Read over cat/head/tail)

**DO NOT use:**
- ❌ Bash commands: `ls -R`, `find .`, `grep`, `cat`, `head`, `tail`, `sed`, `awk`
- ❌ Glob without extension filter: `**/*` (too broad, lists everything)
- ❌ Reading files outside `<target>/` (verify path starts with target first)

**Rationale:** Dedicated tools provide better permission handling, are faster, and are auditable by the user.

See `skills/_shared/common-rules.md` lines 22-29 for full details.

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
- **Confidence level:** VERIFIED / GAP

---

## Output

Write exactly ONE report:

**REPORT_PATH:** `<target>/docs/rda/reports/00_service_inventory.md`

### Report Structure (in this exact order)

# Service Inventory Report

## 0. Run Metadata
- **Timestamp (UTC):** <YYYY-MM-DD HH:MM:SS UTC>
- **Git Commit:** <hash or GAP>

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

## Completion Criteria

Before submitting:
1. Run Metadata section is complete with timestamp and file list
2. All 10 checklist questions answered with evidence + confidence
3. No files outside `TARGET_SERVICE_PATH` were opened
4. External dependencies recorded but not inspected
5. Delta section present (even if first run)
6. Report written to correct path
