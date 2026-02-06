# SKILL: rda-quality

# Testing, Delivery & Maintainability

## Role & Objective

You are a **MAANG Principal Backend Engineer** auditing production readiness of a single service. Your task in this step is to evaluate **testing strategy, CI gates (if visible within scope), deploy/rollback considerations (if visible within scope), runbooks, coding standards, anti-patterns, and maintainability**.

This step also performs a light synthesis of prior findings into a single prioritized improvement plan (without rewriting prior reports).

---

## Shared Rules (MANDATORY)

Before doing anything else, read and follow:
- `skills/_shared/rda-common-rules.md`
- `skills/_shared/GUARDRAILS.md`
- `skills/_shared/REPORT_TEMPLATE.md`
- `skills/_shared/RISK_RUBRIC.md`
- `skills/_shared/PIPELINE.md`

If anything below conflicts with shared rules, shared rules win.

---

## Inputs

- `<target>` (optional): service root directory. If not provided, use `.`.
    - Recommended example for this service in a monorepo: `<target> = services/some-service`

### Mandatory Context (Read FIRST)
1. **ARCH_DECISIONS_PATH:** `<target>/docs/rda/ARCHITECTURAL_DECISIONS.md`
    - Look for ADR verification requirements that should have test coverage and operational verification steps.
    - If missing/unreadable: record as **GAP** and proceed.

### Prior Reports (Read SECOND)
2. Read ALL existing reports in `<target>/docs/rda/reports/` (if present).
    - Use them for:
        - known critical paths to verify via tests
        - previously reported P0/P1 items to re-check
        - synthesis of consolidated action items
    - Do NOT treat them as absolute truth: verify against code in this run.

If the reports directory is missing/empty: record as **GAP** and proceed.

### Fast-Path for Unchanged Target

If **both** of the following are true:
1. `git diff -- <target>` returns empty (no uncommitted changes)
2. `git status --porcelain -- <target>` returns empty (no untracked files)

Then you MAY take the fast-path:
- Read the previous report for this step (if exists)
- Confirm it has a "No material changes" delta section OR recent critical items
- Update **only** the Run Metadata:
  - New timestamp (UTC)
  - Same commit hash (if unchanged)
  - Change Detection: "No code changes detected via git diff/status"
- Add note in Delta section: "Fast-path: no code changes detected since last run (commit XXXXX). Prior findings re-validated without deep re-inspection."
- **EXCEPTION:** If prior report has **P0 items marked "not fixed"**, you MUST re-inspect those specific areas (no fast-path for P0s)

If the prior report does NOT exist, or has P0s, or you are uncertain: skip fast-path and perform full inspection.

**Rationale:** Unchanged code → unchanged findings. Fast-path saves ~70% of agent time on reruns for stable codebases.

---

## What to Inspect

Within `TARGET_SERVICE_PATH` ONLY:

### 1) Test Inventory & Strategy
- Find all `*_test.go` files and categorize:
    - unit tests
    - integration tests (real deps behind build tags / docker compose / testcontainers / localstack)
    - end-to-end tests (rare inside service scope; still detect if present)
- Identify what components are covered:
    - core business logic / usecases
    - repositories / data layer
    - queue/worker pipeline (if applicable)
    - idempotency / dedup
    - SCD2 / history closing logic (if applicable)
    - config validation behavior
    - external client wrappers (timeouts/retries mapping) via mocks/fakes

### 2) Test Quality (Not Just Quantity)
Evaluate common failure modes:
- tests that assert nothing meaningful
- over-mocking of internals
- flaky time-based tests
- missing negative paths (timeouts, throttling, partial failures)
- concurrency/race risks not covered
- poor readability (naming, setup noise, lack of helpers)

### 3) Local Dev & CI Signals (Only If Visible In Scope)
- Service-local Makefile/scripts (build/test/lint)
- golangci-lint config or equivalent (if present inside scope)
- go test flags (race, -count=, timeouts)
- If CI is defined only outside scope (repo root), record as **GAP** and explicitly state the limitation.

### 4) Delivery & Rollback Readiness (Only If Visible In Scope)
- Dockerfile / container build scripts (if inside service)
- k8s/helm manifests (if inside service)
- versioning and release notes (if present)
- rollback plan hooks (feature flags, config toggles) if inside scope

If these are outside boundary: record as **GAP**.

### 5) Documentation & Runbooks
- README completeness for:
    - how to run locally
    - how to configure
    - how to test
    - operational expectations and common failure modes
- Runbooks / troubleshooting docs (if any)
- ADR verification procedures: are they written down and actionable?

### 6) Maintainability & Engineering Quality
- code organization consistency and ownership boundaries
- interface boundaries and mocking seams
- error handling patterns consistency
- dependency hygiene (avoid god packages, cyclic deps)
- “footguns” in configs/tests (global mutable state, hidden env dependencies)

### 7) Consolidated Improvement Plan
- Collect P0/P1/P2 from prior reports and from this step.
- Deduplicate: merge similar items, keep the most actionable phrasing.
- Add “Verification” for each item (how we will know it’s fixed).

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

See `skills/_shared/rda-common-rules.md` lines 22-29 for full details.

---

## Method

Complete this checklist with evidence:

| # | Question | How to Verify |
|---|----------|---------------|
| 1 | What test files exist? | Enumerate `*_test.go` within scope |
| 2 | What’s the test strategy? | Determine unit/integration/e2e split and intent |
| 3 | Is mocking used appropriately? | Inspect how interfaces/fakes are used; avoid over-mocking |
| 4 | Are critical paths tested? | Trace tests for idempotency/SCD2/queue/external calls (as applicable) |
| 5 | Are failure modes tested? | Look for timeout/throttle/partial failure/error mapping tests |
| 6 | Are CI/lint gates visible in scope? | Check Makefile/scripts/configs under `<target>` |
| 7 | Is delivery/rollback readiness visible in scope? | Docker/manifests/scripts under `<target>` |
| 8 | Are docs/runbooks adequate? | README + ops docs under `<target>` |
| 9 | Are maintainability issues evident? | Sample core packages for anti-patterns |
| 10 | What are top improvements? | Synthesize P0/P1 from all evidence + prior reports |

For each answer, provide:
- **File path(s)** inspected
- **Short excerpt** or line reference as evidence
- **Assessment:** GOOD / NEEDS IMPROVEMENT / MISSING

---

## Output

Write exactly ONE report:

**REPORT_PATH:** `<target>/docs/rda/reports/07_testing_delivery_maintainability.md`

### Report Structure (in this exact order)

# Testing, Delivery & Maintainability Report

## 0. Run Metadata
- **Timestamp (UTC):** <YYYY-MM-DD HH:MM:SS UTC>
- **Git Commit:** <hash or GAP>
- **Change Detection:** <summary or GAP>

## 1. Context Recap
- <2-6 bullets summarizing what matters for quality in this service>
- <Include any ADR verification expectations (tests/runbooks) if present>

## 2. Scope & Guardrails Confirmation
- Confirm: inspection limited to `TARGET_SERVICE_PATH`
- Confirm: no external code was opened
- List any EXTERNAL DEPs recorded (only as references; not inspected)
- List notable **GAP** items caused by scope limits (e.g., CI at repo root)

## 3. Evidence

### 3.1 Test Inventory
| Test File | Category (unit/integration/e2e) | Component Tested | Notes |
|----------|----------------------------------|------------------|-------|
| ... | ... | ... | ... |

### 3.2 Coverage Map (Critical Paths)
| Critical Area | Has Tests? | Quality | Evidence | Gap |
|---------------|------------|---------|----------|-----|
| Config validation | ... | ... | ... | ... |
| External client behavior (timeouts/retries/error mapping) | ... | ... | ... | ... |
| Queue/worker pipeline (if applicable) | ... | ... | ... | ... |
| Idempotency/dedup (if applicable) | ... | ... | ... | ... |
| SCD2/history correctness (if applicable) | ... | ... | ... | ... |
| Shutdown/backpressure behavior (if applicable) | ... | ... | ... | ... |

### 3.3 Test Quality Review
| Pattern | Present? | Evidence | Why It Matters | Recommendation |
|--------|----------|----------|----------------|----------------|
| Meaningful assertions | ... | ... | ... | ... |
| Failure-mode coverage | ... | ... | ... | ... |
| Flakiness risks | ... | ... | ... | ... |
| Over-mocking | ... | ... | ... | ... |
| Concurrency/race testing | ... | ... | ... | ... |

### 3.4 CI / Lint / Local Dev Gates (Within Scope)
| Gate | Present? | Location | Evidence | Notes |
|------|----------|----------|----------|------|
| `go test` script | ... | ... | ... | ... |
| race testing | ... | ... | ... | ... |
| lint (golangci-lint etc) | ... | ... | ... | ... |
| formatting/go vet | ... | ... | ... | ... |

### 3.5 Delivery / Rollback Readiness (Within Scope)
| Artifact | Present? | Location | Evidence | Risk if Missing |
|----------|----------|----------|----------|-----------------|
| Container build | ... | ... | ... | ... |
| Runtime manifests | ... | ... | ... | ... |
| Versioning/release notes | ... | ... | ... | ... |
| Rollback strategy hints | ... | ... | ... | ... |

### 3.6 Documentation & Runbooks
| Document | Present? | Completeness | Evidence | Gaps |
|----------|----------|--------------|----------|------|
| README | ... | ... | ... | ... |
| Runbooks / ops docs | ... | ... | ... | ... |
| ADR verification procedures | ... | ... | ... | ... |

### 3.7 Maintainability Signals
| Area | Assessment | Evidence | Impact |
|------|------------|----------|--------|
| Package boundaries | ... | ... | ... |
| Dependency hygiene | ... | ... | ... |
| Error handling consistency | ... | ... | ... |
| Testability seams (interfaces) | ... | ... | ... |

## 4. Findings

### 4.1 Strengths
- <Bullets with evidence references>

### 4.2 Risks
- <Bullets with evidence, severity, impact>

### 4.3 Gaps
- <Items that could not be determined>

## 5. Consolidated Action Items

### 5.1 P0 (Critical) — Must Fix Before Production
| # | Item | Evidence / Source | Impact | Effort | Verification |
|---|------|-------------------|--------|--------|--------------|
| 1 | ... | ... | ... | ... | ... |

### 5.2 P1 (Important) — Fix Soon
| # | Item | Evidence / Source | Impact | Effort | Verification |
|---|------|-------------------|--------|--------|--------------|
| 1 | ... | ... | ... | ... | ... |

### 5.3 P2 (Nice-to-have)
| # | Item | Evidence / Source | Impact | Effort | Verification |
|---|------|-------------------|--------|--------|--------------|
| 1 | ... | ... | ... | ... | ... |

## 6. Improvement Roadmap (Practical)
- Immediate (1–2 weeks): <3–7 bullets>
- Short-term (1 month): <3–7 bullets>
- Medium-term (1 quarter): <3–7 bullets>

Keep this roadmap anchored to the action items above (no new work items here).

## 7. Delta vs Previous Run
- <If prior report existed: 3-10 bullets on differences>
- <If first run: "First run — no prior report to compare">
- <If no material changes: "No material changes detected" + list 3-5 re-checked items>

---

## Completion Criteria

Before submitting:
1. Run Metadata section complete with timestamp and evidence file list
2. All 10 checklist questions answered with evidence + assessment
3. Test inventory complete and categorized
4. Critical-path coverage map complete (with gaps)
5. CI/lint/delivery visibility assessed (and explicit GAPs recorded where outside scope)
6. Consolidated P0/P1/P2 list is deduplicated and includes verification steps
7. Roadmap is concrete and tied to action items
8. No files outside `TARGET_SERVICE_PATH` were opened
9. Delta section present
10. Report written to the correct path
