# REPORT_TEMPLATE.md — rda step report (generic)

> Purpose: reusable template for any rda step report (00–09+).  
> Keep it short, evidence-driven, and within scope.

---

# Step <STEP_ID>: <STEP_TITLE> Report

## 0. Run Metadata
- **Timestamp (UTC):** <YYYY-MM-DD HH:MM:SS UTC>
- **Audit Author:** Rubber Duck Auditor v0.1.1 (Claude Code plugin)
- **Git Commit:** <hash | GAP>
- **Change Detection:** <`git diff -- <TARGET_SERVICE_PATH>` summary | `git status --porcelain -- <TARGET_SERVICE_PATH>` summary | GAP>

## 1. Context Recap
- **Service:** <service-name> — <1 sentence what it does / runtime mode>
- **Scope for this step:** <1 sentence (what you audited in this step)>
- **Relevant prior reports used:** <list filenames or GAP>
- **Key components involved:** <2–5 bullets, sourced from prior reports>

### Decision Context (from `docs/rda/ARCHITECTURAL_DECISIONS.md`)
- **Relevant ADRs:** <ADR-xxx title(s), why relevant>
- **Excerpt(s):** <short excerpt, ≤25 words, include ADR heading + line range if available>
- If decisions file missing/unreadable: **GAP** — <impact>

## 2. Scope & Guardrails Confirmation
- ✅ **Inspection limited to:** <TARGET_SERVICE_PATH> (e.g. `services/some-service`)
- ✅ **No external code opened:** CONFIRMED
- ✅ **No files outside TARGET_SERVICE_PATH accessed:** CONFIRMED

### External Dependencies Recorded
Rule: if code imports/references outside boundary, record as **EXTERNAL DEP** and do NOT open it.

| Import / Reference | Type | Used In | Notes |
|---|---|---|---|
| <module/path or endpoint ref> | EXTERNAL DEP / Direct | <file(s)> | <purpose> |
| ... | ... | ... | ... |

## 3. Evidence
Hard rule: **No speculation.** Every assertion must include evidence (file path + short excerpt or file path + line range).

### 3.1 Inventory (Step-Specific)
Provide step-specific inventory table(s). Examples:
- Integrations: clients + config locations
- Data handling: models + migrations + repositories
- Reliability: poller/batcher/consumer + shutdown hooks
- Observability: logger/metrics/tracing/health endpoints
- Testing: test files + coverage map
- Security: authn/authz, secrets, TLS, PII handling
- Performance: limits, batching, memory, capacity assumptions

| Item | Location | Purpose | Notes |
|---|---|---|---|
| <component> | <path> | <purpose> | <notes> |
| ... | ... | ... | ... |

### 3.2 Deep-Dive Analysis (Step-Specific)
Use 1–3 focused tables. Keep them scannable.

| Aspect | Configuration / Behavior | Evidence | Assessment |
|---|---|---|---|
| <aspect> | <what it is> | <file:line-range + tiny excerpt> | CORRECT / CONCERN / RISK / GAP |
| ... | ... | ... | ... |

### 3.3 Cross-Cutting Concerns (if applicable)
| Concern | Implementation | Evidence | Assessment |
|---|---|---|---|
| Error mapping | <pattern> | <file:line-range> | CORRECT / CONCERN / RISK / GAP |
| Timeouts | <values> | <file:line-range> | CORRECT / CONCERN / RISK / GAP |
| Retries/backoff | <strategy> | <file:line-range> | CORRECT / CONCERN / RISK / GAP |
| Idempotency/dedup | <strategy> | <file:line-range> | CORRECT / CONCERN / RISK / GAP |
| TLS/credentials | <how> | <file:line-range> | CORRECT / CONCERN / RISK / GAP |
| ... | ... | ... | ... |

## 4. Findings

### 4.1 Strengths
Only strengths that materially reduce production risk / MTTR. Each bullet must cite evidence.

- <Strength title> — <why it matters>. Evidence: <file:line-range>; <file:line-range>.
- <Strength title> — <why it matters>. Evidence: <file:line-range>.

### 4.2 Risks
Include severity and impact. Each risk must be tied to evidence or marked as GAP.

#### P0 (Critical)
- <Risk title> — <impact/outage class/data loss/security>. Evidence: <file:line-range>.

#### P1 (Important)
- <Risk title> — <impact>. Evidence: <file:line-range>.

#### P2 (Nice-to-have)
- <Risk title> — <impact>. Evidence: <file:line-range>.

### 4.3 Decision Alignment Assessment
Use when findings relate to an ADR. Classify:
- **ALIGNED**: code/config matches ADR
- **DRIFT**: contradicts ADR
- **DECISION RISK**: ADR itself is risky/outdated

- **Decision:** <ADR-xxx title>
    - **Expected (ADR):** <summary>
    - **Actual:** <what code does>
    - **Assessment:** ALIGNED / DRIFT / DECISION RISK
    - **Evidence:** <file:line-range>; <ADR heading + line-range or GAP>

### 4.4 Gaps
Unknowns that block confidence. Must explain why it matters and what is missing.

- **GAP:** <what is unknown> — <why it matters>. Missing: <file/report/dir not found>.

## 5. Action Items
Each item must be verifiable (“Verify:” is mandatory). Keep it short.

### P0 (Critical)
- <Action> | Impact: <what it prevents/unlocks> | Verify: <specific check/log/metric/test>

### P1 (Important)
- <Action> | Impact: <…> | Verify: <…>

### P2 (Nice-to-have)
- <Action> | Impact: <…> | Verify: <…>

## 6. Delta vs Previous Run
Rule: you may open the previous report file ONLY at the end to write this section. No copy-paste.

- **Previous Report:** <same file path> at commit <hash | GAP> dated <UTC time | GAP>
- **If material changes:**
    - <Change> — Evidence: <file:line-range> / <prior report section>
    - <Change> — Evidence: <file:line-range> / <prior report section>
- **If no material changes:** No material changes detected. Re-checked:
    - <item> — Evidence: <file:line-range>
    - <item> — Evidence: <file:line-range>

---

## Notes (author conventions)
- Use explicit labels when applicable: **DRIFT**, **DECISION RISK**, **GAP**, **ASSUMPTION**.
- **NEVER include "Inspected Files" or "Files Read" sections** in the report.
  - These are verbose and low-signal.
  - All evidence should be inline citations: "Evidence: file.go:12-14 — does X".
- Prefer "Evidence: <file:line-range>" over long prose.
- Keep it scannable: a reader should extract the key risks in <5 minutes.
- Never open files outside <TARGET_SERVICE_PATH>; record them as **EXTERNAL DEP** only.

---

<sub>Generated by [Rubber Duck Auditor v0.1.1](https://github.com/tifongod/rubber-duck-auditor) — a Claude Code plugin for MAANG-grade production readiness audits | Install: `/plugin marketplace add tifongod/rubber-duck-auditor && /plugin install rda@rubber-duck-auditor`</sub>
