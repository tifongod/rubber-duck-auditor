# SKILL: architecture

# Architecture & Design

## Role & Objective

You are a **MAANG Principal Backend Engineer** auditing production readiness of a single service. Your task in this step is to evaluate the **architectural patterns, layering, dependency direction, composition root, and cross-cutting concerns** of the service.

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
    - Use file inventory, package structure, and entrypoints from this report.
    - If missing: record as **GAP** and perform minimal targeted scan of entrypoints and top-level packages only.

---

## What to Inspect

Use file locations from the inventory report where possible. Within `TARGET_SERVICE_PATH` ONLY:

### 1. Composition Root
- Locate where dependencies are wired together (typically `cmd/*/main.go`, server bootstrap, or `internal/app/`).
- Analyze DI pattern: manual, wire, fx, or custom.
- Check if dependencies are injected or if there are hidden globals/singletons.

### 2. Layering & Package Structure
- Map the layer hierarchy: entrypoint → app wiring → usecases/services → repositories/clients → models/domain.
- Verify dependency direction: outer layers depend on inner, not vice versa.
- Identify layering violations (e.g., repository importing usecase, domain importing infra).

### 3. Interface Usage
- Check if dependencies are abstracted via interfaces at boundaries (repos, external clients).
- Locate interface definitions: same package as implementation or consumer package?
- Evaluate whether interfaces are minimal, stable, and test-supportive (or over-abstraction).

### 4. Cross-Cutting Concerns
- **Logging:** How is the logger passed? context-based? global? per-request fields?
- **Metrics:** Where are metrics registered and used? dependency injection? global registry?
- **Error handling:** Consistent patterns? wrapping/classification? typed errors?
- **Context propagation:** Is `context.Context` used across call chains? cancellation respected?

### 5. Separation of Concerns
- Are domain models separate from infrastructure concerns?
- Is business logic in usecases/services, not handlers/controllers or repositories?
- Are external integrations abstracted behind interfaces or clear adapters?

---

## Method

Complete this checklist with evidence:

| # | Question | How to Verify |
|---|----------|---------------|
| 1 | Where is the composition root? | Find main wiring logic; trace dependency construction |
| 2 | What DI pattern is used? | Analyze how dependencies are created and passed |
| 3 | What are the architectural layers? | Map package dependencies with import analysis |
| 4 | Are there layering violations? | Check for imports from outer to inner layers |
| 5 | Are interfaces used for external deps? | Find interface definitions for repos/clients |
| 6 | Where are interfaces defined? | Same package as impl or consumer package? |
| 7 | How is logging handled? | Trace logger creation and propagation |
| 8 | How are metrics implemented? | Find metrics registration and usage points |
| 9 | Is error handling consistent? | Sample error paths in key flows |
| 10 | Is context propagated correctly? | Verify `context.Context` passed through call chains |

For each answer, provide:
- **File path(s)** inspected
- **Short excerpt** or line reference as evidence
- **Assessment:** GOOD / CONCERN / VIOLATION

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

**REPORT_PATH:** `<target>/docs/rda/reports/02-architecture-design.md`

### Report Structure (in this exact order)

# Architecture & Design Report

## 0. Run Metadata
- **Timestamp (UTC):** <YYYY-MM-DD HH:MM:SS UTC>
- **Git Commit:** <hash or GAP>
- **Change Detection:** <summary or GAP>
- **Inspected Files (Top Evidence):** <list 10-30 files>

## 1. Context Recap
- <2-5 bullets on service architecture, referencing inventory report where relevant>
- <Include relevant decision context from ARCH_DECISIONS_PATH>

## 2. Scope & Guardrails Confirmation
- Confirm: inspection limited to `TARGET_SERVICE_PATH`
- Confirm: no external code was opened
- List any EXTERNAL DEPs recorded (if relevant to architecture)

## 3. Evidence

### 3.1 Composition Root Analysis
| Aspect | Location | Pattern | Assessment |
|--------|----------|---------|------------|
| Main wiring | ... | ... | ... |
| DI approach | ... | ... | ... |
| Hidden globals? | ... | ... | ... |

<Include short excerpts/line refs only>

### 3.2 Layer Diagram (Actual, Evidence-Based)
- Show an ASCII diagram of the real dependency flow (based on inspected imports).
- If uncertain, mark uncertain edges as **INFERRED** and explain why.

Example format:
cmd/
└── main.(*)  → internal/app  → internal/usecases  → internal/repositories/clients  → internal/models

### 3.3 Interface Analysis
| Interface | Location | Implementors | Usage Pattern | Assessment |
|-----------|----------|--------------|---------------|------------|
| ... | ... | ... | ... | ... |

### 3.4 Cross-Cutting Concerns
| Concern | Implementation | Evidence | Assessment |
|---------|----------------|----------|------------|
| Logging | ... | ... | ... |
| Metrics | ... | ... | ... |
| Errors | ... | ... | ... |
| Context | ... | ... | ... |

## 4. Findings

### 4.1 Strengths
- <Bullets with evidence references>

### 4.2 Risks
- <Bullets with evidence, severity, impact>

### 4.3 Gaps
- <Items that could not be determined within scope>

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
3. Composition root clearly identified with DI pattern analysis
4. Layer diagram matches actual code structure (or marks uncertainties explicitly)
5. No files outside `TARGET_SERVICE_PATH` were opened
6. Delta section present (even if first run)
7. Report written to correct path
