# Architectural Decisions & Conventions

**Purpose:** Provide context for AI agents and engineers conducting rda audits.

---

## ADR-001: Prompt Composition via Read-First Instructions

**Status:** Accepted

**Context:** The rda system is a prompt engineering artifact with no traditional runtime. We need a way to inject shared rules and standards into individual audit step prompts.

**Decision:** Use explicit "read these files first" instructions at the top of each SKILL.md file (lines 13-20 typically). Each skill declares dependencies on shared files via relative paths (e.g., `skills/_shared/common-rules.md`).

**Consequences:**
- ✅ Simple, no abstraction complexity
- ✅ Explicit dependencies visible in each file
- ✅ No build/compilation step needed
- ⚠️ Relative paths break if directory structure changes
- ⚠️ Duplication of read-first instruction blocks across 11 skills

**Rationale:** Zero-runtime constraint prevents programmatic composition. Text-based composition is the only option.

---

## ADR-002: Content Duplication vs Shared File Extraction

**Status:** Accepted (with known technical debt)

**Context:** Several content blocks appear verbatim across multiple skills:
- Tool Usage section (~170 lines across 10 skills)
- Fast-Path optimization (~180 lines across 10 skills)
- Scope/evidence rules overlap between common-rules.md and GUARDRAILS.md (~70-80%)

**Decision:** Accept duplication for now. Prioritize working system over perfect maintainability.

**Consequences:**
- ✅ Each skill is self-contained and easier to understand in isolation
- ✅ No risk of breaking multiple skills when editing shared content
- ⚠️ Changes require manual updates across 10+ files
- ⚠️ Risk of drift/inconsistency over time

**Rationale:** This is a small-scale prompt system (~3,150 lines total). Manual synchronization cost is acceptable. Future extraction to shared files (e.g., `TOOL_USAGE.md`, `FAST_PATH.md`) is planned when duplication pain exceeds extraction complexity.

**Status as of 2026-02-07:** Known P1 technical debt. Extraction planned but not urgent.

---

## ADR-003: Wave-Based Execution Model

**Status:** Accepted

**Context:** Audit steps have dependencies (e.g., architecture depends on inventory). Without sequencing, steps may execute with missing context.

**Decision:** Use 7-wave execution model in `audit-pipeline` skill. Each wave defines:
- Which steps run in parallel
- Which prior reports must exist before wave starts
- Completion verification gates between waves

**Consequences:**
- ✅ Prevents missing-context failures
- ✅ Explicit data flow via report file paths
- ✅ Parallelization within waves (e.g., Wave 3 runs 3 independent steps)
- ⚠️ Sequential waves increase total execution time vs full parallel

**Rationale:** Correctness over speed. Missing context produces unreliable audit findings.

---

## ADR-004: Security Hardening in Prompts

**Status:** Accepted

**Context:** Prompts execute arbitrary Bash commands and read untrusted file content. Three attack surfaces:
1. Path traversal via `<target>` parameter
2. Shell injection via unescaped substitution
3. Prompt injection via file content interpreted as instructions

**Decision:** Add three security sections to `GUARDRAILS.md`:
- Path validation (reject `..`, shell metacharacters)
- Mandatory shell escaping (double-quoted `"$target"`)
- Prompt injection defense (treat all file content as untrusted data)

**Consequences:**
- ✅ Mitigates known attack vectors
- ✅ Explicit security guidance for agent execution
- ⚠️ Relies on agent compliance (not programmatically enforced)

**Rationale:** Zero-runtime architecture means no middleware for security. Prompt-level rules are the only control point.

---

## ADR-005: Path Hygiene with `<run_root>` Concept

**Status:** Accepted (2026-02-06, commit 65b6c58)

**Context:** Prior reports contained absolute paths (e.g., `/Users/username/...`) making reports machine-specific and non-portable.

**Decision:** Introduce `<run_root>` concept in `common-rules.md:10-17`. All report output paths must be relative. Absolute paths are explicitly prohibited.

**Consequences:**
- ✅ Reports are portable across machines
- ✅ No machine-specific path pollution
- ⚠️ Prior reports (14 occurrences) still non-compliant

**Rationale:** Audit reports are diagnostic artifacts often shared across teams. Machine-specific paths reduce utility.

---

## ADR-006: Clean Directory Naming (no `rda-` prefix)

**Status:** Accepted (2026-02-06, commit 9a9146a)

**Context:** Skill directories were named `rda-inventory`, `rda-architecture`, etc. The `rda-` prefix was redundant (all skills are under `skills/` directory in the rda plugin).

**Decision:** Rename all skill directories from `rda-<name>` to `<name>`. Use clean names: `inventory`, `architecture`, `security`, etc.

**Consequences:**
- ✅ Reduces visual noise
- ✅ Simplifies path references
- ⚠️ Breaks existing path references if not updated

**Rationale:** Plugin name (`rda`) already provides namespace. Directory prefix was redundant.

---

## ADR-007: Zero Test Policy

**Status:** Accepted (with known risk)

**Context:** ~3,150 lines of prompt content with no automated validation.

**Decision:** Ship without tests. Rely on dogfooding (running audits on the rda codebase itself) for validation.

**Consequences:**
- ✅ Faster initial development
- ✅ No test infrastructure complexity
- ⚠️ No regression detection
- ⚠️ Prompt drift risk

**Rationale:** Prompt quality is hard to test programmatically. Manual verification via self-audit is pragmatic starting point. Automated validation is future work.

---

## ADR-008: Relative Path References to Shared Files

**Status:** Accepted (with known fragility)

**Context:** All skills reference shared files via relative paths (e.g., `skills/_shared/common-rules.md`). If skill directory moves or is symlinked, references break.

**Decision:** Accept relative paths. Plugin structure is stable. No hot-swapping or dynamic directory changes expected.

**Consequences:**
- ✅ Simple, no path resolution logic needed
- ⚠️ Fragile to directory restructuring

**Rationale:** Stability assumption is reasonable for a static plugin artifact. Cost of breakage is low (immediate failure, easy to detect).

---

## Future Decisions (Pending)

1. **Extract Tool Usage to shared file** - Pending design for `skills/_shared/TOOL_USAGE.md`
2. **Consolidate common-rules.md and GUARDRAILS.md** - Pending decision on single canonical source
3. **Automated prompt quality validation** - Pending design for validation framework
4. **Fast-Path extraction** - Pending design for `skills/_shared/FAST_PATH.md`
