# SKILL: security

# Security & Privacy

## Role & Objective

You are a **FAANG Principal Backend Engineer** auditing production readiness of a single service. Your task in this step is to evaluate **security and privacy posture** of the service: attack surface, authn/authz, secrets handling, crypto/TLS, dependency & supply-chain risks, least-privilege for external systems, input validation, and sensitive-data hygiene.

The output must be practical: concrete risks, evidence, and a prioritized remediation plan.

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

1) **Scope Boundary**
- `TARGET_SERVICE_PATH = <target>` (if not provided, use `.`)
- You MUST NOT read, list, or analyze any files outside this path.

2) **Strictly Prohibited**
- Analyzing the entire monorepo.
- Scanning/listing/reading files outside `TARGET_SERVICE_PATH`.
- Producing an overview/map of the monorepo or enumerating other services outside `TARGET_SERVICE_PATH`.
- Opening shared/common libraries outside `TARGET_SERVICE_PATH`.
- If code inside the service imports modules outside boundary, you may ONLY record the dependency (import path/config reference) as **EXTERNAL DEP** and mark unknowns as **GAP**. You MUST NOT open external code.

3) **No Code Modifications**
- Analysis + report only.

4) **Missing Info Policy**
- If information is outside boundary or unavailable, record as **GAP** and continue.

---

## Rerun / Anti-Copy Requirements (MANDATORY)

### A) Iteration Without Copy-Paste
- You MUST treat prior reports in `<target>/docs/rda/reports/` as context and previous-run state.
- You MUST NOT copy prior content verbatim.
- You MUST draft findings based on inspection in this run, then reconcile with prior report(s) and produce a proper delta.

### B) Mandatory Run Metadata
Every report MUST include:
- **Timestamp (UTC)**
- **Git commit hash** for `TARGET_SERVICE_PATH` (or GAP)
- **Change Detection summary**
    - Preferred: `git diff -- <target>` or `git status --porcelain -- <target>`
    - If git commands unavailable: GAP
- **Inspected Files (Top Evidence):** 10–30 file paths actually opened

### C) Mandatory Delta Section
Every report MUST include "Delta vs previous run":
- If prior report existed: 3–10 bullets summarizing differences
- If no material changes: "No material changes detected" + 3–5 items re-checked
- If first run: "First run — no prior report to compare"

---

## Inputs

- `<target>` (optional): service root directory. If not provided, use `.`.
    - Recommended example: `<target> = services/qql-runtime`

### Mandatory Context (Read FIRST)
1) `ARCH_DECISIONS_PATH = <target>/docs/rda/ARCHITECTURAL_DECISIONS.md`
- Use as context for intentional trade-offs.
- Do NOT treat as unquestionable truth.
- If missing/unreadable: record as **GAP** and proceed.

### Prior Reports (Read SECOND)
2) Read ALL existing reports in `<target>/docs/rda/reports/` (if present).
- Use them to:
    - identify previously known critical paths (what to secure first)
    - re-check previously reported P0/P1 items
    - avoid duplicates and keep the plan coherent
- Do NOT treat them as absolute truth: verify against code in this run.

If the reports directory is missing/empty: record as **GAP** and proceed.

---

## What to Inspect (SECURITY-SCOPED)

Within `TARGET_SERVICE_PATH` ONLY.

### 1) Attack Surface & Trust Boundaries
- Entrypoints: HTTP/gRPC servers, workers, CLIs, admin/private endpoints, health/debug endpoints.
- Network listeners and exposed ports (where visible in code/config).
- Trust boundaries: what is considered trusted input vs untrusted input (requests, queue messages, configs).

### 2) AuthN / AuthZ
- Authentication mechanisms (JWT/API keys/mTLS/service identity).
- Authorization checks: where enforced, how consistently applied, default-deny posture.
- Internal/admin endpoints protection (especially health, metrics, debug).

### 3) Input Validation & Injection Risks
- Request parsing and validation (HTTP/gRPC).
- Any query building / DSL parsing (risk of injection or sandbox escape).
- Risks: SQL/ClickHouse injection, SSRF, path traversal, template injection, regex DoS, unsafe deserialization.
- For queue consumers: validation of message schema/version and strict handling of unexpected fields.

### 4) Secrets & Sensitive Config
- Secrets sources: env vars, files, config, embedded constants.
- Evidence of secrets in repository: hardcoded keys, test fixtures with real credentials, `.env` committed, sample configs leaking secrets.
- Redaction policy: ensure secrets are not logged/returned in errors.

### 5) Crypto & Transport Security
- TLS usage for external calls (AWS, DB, ClickHouse, other HTTP/gRPC).
- TLS verification settings (no insecure skip verify, no plaintext fallback).
- Any encryption/hashing usage: correct primitives, no custom crypto, key management strategy.
- At-rest encryption assumptions (if referenced in configs/docs): verify what’s visible in code/config; otherwise GAP.

### 6) External Systems Least-Privilege (As Visible From This Repo Slice)
- AWS IAM usage patterns (role-based auth vs static keys).
- Permissions scoping (queue/table names, prefixes, actions).
- Database credentials and privilege scope (read/write separation, least privilege).
- If IAM policies/manifests are outside boundary: record GAP and describe what must be verified outside.

### 7) Dependency & Supply-Chain Hygiene
- `go.mod` / dependency choices: risky libs, abandoned libs, excessive transitive surface.
- Use of `replace` directives, private forks, unpinned versions.
- Build tooling signals within scope (if present): SAST/secrets scan config, govulncheck, dependency scanning.
- If CI is defined only outside boundary: record GAP and list what checks should exist.

### 8) Privacy & Data Protection
- Identify sensitive data classes handled by service (PII, credentials, tokens, customer data) based on models/events/log fields.
- Data minimization: do we store/log/emit more than needed?
- Retention/deletion: any TTL/cleanup jobs, data lifecycle docs (if any).
- If privacy requirements are not visible in scope: GAP + list minimum expected artifacts.

---

## Method

Answer each checklist question with:
- **File path(s) inspected**
- **Short excerpt** or **line reference** as evidence
- **Assessment:** GOOD / CONCERN / RISK / GAP

Checklist:

| # | Question | How to Verify |
|---|----------|---------------|
| 1 | What are the service entrypoints and exposed surfaces? | Locate `cmd/` mains and server init; list endpoints/servers |
| 2 | What trust boundaries exist? | Identify sources of untrusted input (HTTP/gRPC, SQS, env/config) |
| 3 | How is authentication implemented (if applicable)? | Inspect middleware/interceptors; config requirements |
| 4 | How is authorization enforced (if applicable)? | Trace request handling to authz checks; check default-deny |
| 5 | Are inputs validated and safely handled? | Look for validation, schema checks, safe parsing; injection risks |
| 6 | How are secrets handled and protected from leakage? | Find config loading, env vars, logging redaction |
| 7 | Is transport security correctly configured? | Check TLS settings for external clients and servers |
| 8 | Are external privileges least-privilege (as visible)? | Inspect AWS client init, resource naming, DB user usage |
| 9 | Is dependency hygiene acceptable? | Review `go.mod`, replace directives, scanning hooks (if any) |
| 10 | Is privacy/sensitive data handled safely? | Inspect models/events/logging; check PII exposure and retention |

---

## Architectural Decisions Awareness

When findings relate to intentional decisions:
- Reference `ARCH_DECISIONS_PATH` (quote short excerpt + section heading).
- Verify the implementation aligns with stated intent.
- If code/config contradicts decisions: label as **DRIFT** with impact and fix.
- If a decision itself is risky: label as **DECISION RISK** with safer alternatives.

---

## Output

Write exactly ONE report:

- `REPORT_PATH = <target>/docs/rda/reports/security.md`

Report must follow this structure and order:

# Security & Privacy Report

## 0. Run Metadata
- Timestamp (UTC)
- Git Commit (or GAP)
- Change Detection (or GAP)
- Inspected Files (Top Evidence) (10–30 paths)

## 1. Context Recap
- 3–7 bullets: what the service does and why security matters here
- Key context from `ARCH_DECISIONS_PATH` (if relevant)

## 2. Scope & Guardrails Confirmation
- Confirm inspection limited to `TARGET_SERVICE_PATH`
- Confirm no external code was opened
- List recorded EXTERNAL DEPs (record-only)
- List notable GAPs caused by boundary (e.g., IAM policies/infra outside scope)

## 3. Evidence

### 3.1 Attack Surface Inventory
| Surface | Location | Exposure | Notes | Assessment |
|---------|----------|----------|-------|------------|
| HTTP/gRPC server | ... | public/private | ... | ... |
| Health/metrics/debug | ... | ... | ... | ... |
| Worker/consumer | ... | internal | ... | ... |

### 3.2 Trust Boundaries Snapshot
| Input Source | Trusted? | Validation Present? | Evidence | Risk |
|--------------|----------|---------------------|----------|------|
| HTTP/gRPC requests | ... | ... | ... | ... |
| Queue messages | ... | ... | ... | ... |
| Env/config | ... | ... | ... | ... |

### 3.3 AuthN/AuthZ (If Applicable)
| Control | Where Implemented | Coverage | Evidence | Assessment |
|---------|--------------------|----------|----------|------------|
| Authentication | ... | ... | ... | ... |
| Authorization | ... | ... | ... | ... |
| Admin/private endpoints protection | ... | ... | ... | ... |

If auth is out-of-scope for this service: state explicitly and justify with evidence.

### 3.4 Input Validation & Injection Review
| Vector | Location | Current Handling | Evidence | Assessment | Recommendation |
|--------|----------|------------------|----------|------------|----------------|
| ClickHouse/SQL injection | ... | ... | ... | ... | ... |
| SSRF | ... | ... | ... | ... | ... |
| Path traversal | ... | ... | ... | ... | ... |
| Regex DoS / parsing | ... | ... | ... | ... | ... |
| Unsafe deserialization | ... | ... | ... | ... | ... |

### 3.5 Secrets & Sensitive Config
| Secret / Sensitive Value | Source | Handling | Leakage Risk | Evidence | Assessment |
|--------------------------|--------|----------|--------------|----------|------------|
| ... | env/config/file | ... | ... | ... | ... |

Include explicit check for: hardcoded secrets, test fixtures, sample configs, logs/errors exposure.

### 3.6 Crypto & Transport Security
| Channel | TLS Enabled? | Verification | Cipher/Settings (if visible) | Evidence | Assessment |
|---------|--------------|--------------|------------------------------|----------|------------|
| External clients | ... | ... | ... | ... | ... |
| Inbound server | ... | ... | ... | ... | ... |

### 3.7 Least-Privilege & External Access (Visible Scope Only)
| System | Auth Mechanism | Scope/Permissions (visible) | Evidence | Assessment | GAPs |
|--------|----------------|-----------------------------|----------|------------|------|
| AWS (SQS/Dynamo/etc) | ... | ... | ... | ... | ... |
| DB/ClickHouse | ... | ... | ... | ... | ... |

If policies/manifests are outside scope: record as GAP and list required verification questions.

### 3.8 Dependency & Supply-Chain Hygiene
| Item | Evidence | Assessment | Risk | Recommendation |
|------|----------|------------|------|----------------|
| go.mod hygiene | ... | ... | ... | ... |
| replace directives | ... | ... | ... | ... |
| vuln scanning hooks (if any) | ... | ... | ... | ... |

### 3.9 Privacy & Sensitive Data Handling
| Data Type | Where Appears | Stored? | Logged? | Retention Visible? | Evidence | Assessment |
|----------|---------------|---------|--------|--------------------|----------|------------|
| PII/tokens/credentials | ... | ... | ... | ... | ... | ... |

If no PII: justify with evidence; otherwise evaluate minimization and leakage risk.

## 4. Findings

### 4.1 Strengths
- Bullets with evidence references

### 4.2 Risks
- Bullets with evidence, severity, impact
- Prefer concrete exploit/failure scenarios (high-level, no “how-to exploit” instructions)

### 4.3 Gaps
- Items not verifiable within boundary, with what must be checked elsewhere

## 5. Action Items

### P0 (Critical)
- Item | Impact | Evidence | Verification (how to prove fixed)

### P1 (Important)
- Item | Impact | Evidence | Verification

### P2 (Nice-to-have)
- Item | Impact | Evidence | Verification

Keep action items specific and testable.

## 6. Delta vs Previous Run
- If prior report existed: 3–10 bullets on differences
- If first run: "First run — no prior report to compare"
- If no material changes: "No material changes detected" + 3–5 re-checked items

---

## Prioritization Rubric
- **P0:** production risk / security breach class / data exfiltration / auth bypass / RCE / privilege escalation / severe outage
- **P1:** significant hardening: better guardrails, least privilege, better defaults, better validation, improved scanning
- **P2:** hygiene and future-proofing

---

## No Speculation Rule
All assertions MUST be tied to evidence (file path + excerpt/line reference). If evidence is unavailable, label as **INFERRED** or **GAP** and continue.

---

## Completion Criteria
1) Run Metadata complete
2) All 10 checklist questions answered with evidence + assessment
3) Attack surface and trust boundaries explicitly enumerated
4) Secrets leakage risk explicitly checked
5) TLS/crypto posture assessed (or GAP with concrete follow-ups)
6) Least-privilege assessed (or GAP with concrete follow-ups)
7) Dependency hygiene assessed (or GAP with concrete follow-ups)
8) Privacy assessment included
9) Action items are prioritized and verifiable
10) Delta section present
11) Report saved to `<target>/docs/rda/reports/security.md`
