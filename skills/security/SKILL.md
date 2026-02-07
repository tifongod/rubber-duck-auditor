# SKILL: security

# Security & Privacy

## Role & Objective

You are a **MAANG Principal Backend Engineer** auditing production readiness of a single service. Your task in this step is to evaluate the **security and privacy posture** of the service in a technology-agnostic way: attack surface, authn/authz, secrets handling, crypto/TLS, dependency & supply-chain risks, least-privilege for external systems, input validation, and sensitive-data hygiene.

**Do not assume** any specific platform (cloud/vendor), datastore, queue, or deployment model. If a technology matters, you must first **detect it within scope** and then evaluate it.

The output must be practical: concrete risks, evidence, and a prioritized remediation plan.

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
  - Recommended example: `<target> = services/some-service`

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
- Add note in Delta section: "Fast-path: no code changes detected since last run (commit XXXXX). Prior findings re-validated without deep re-inspection."
- **EXCEPTION:** If prior report has **P0 items marked "not fixed"**, you MUST re-inspect those specific areas (no fast-path for P0s)

If the prior report does NOT exist, or has P0s, or you are uncertain: skip fast-path and perform full inspection.

**Rationale:** Unchanged code → unchanged findings. Fast-path saves ~70% of agent time on reruns for stable codebases.

---

## Pattern Discovery (MANDATORY)

Before deep inspection, quickly identify which security surfaces apply to this service and keep evidence (paths + short excerpts):
- Runtime model: API server / worker / cron / CLI / mixed
- Entry points and exposed surfaces: HTTP/gRPC, message consumption, admin/debug/metrics endpoints
- Authn/authz presence: middleware/interceptors, identity tokens/keys, service-to-service auth
- External dependencies: datastores, caches, queues/brokers, object storage, external HTTP/gRPC
- Secrets/config sources: env/config files/secret managers (as visible), and any redaction/logging patterns
- Sensitive data classes: tokens/credentials, PII, customer data, internal identifiers

If something is ABSENT (e.g., no auth layer, no inbound server, no queue usage), explicitly state "Not detected" with discovery evidence later.

---

## What to Inspect (SECURITY-SCOPED)

Within `TARGET_SERVICE_PATH` ONLY.

### 1) Attack Surface & Trust Boundaries
- Entrypoints: HTTP/gRPC servers, workers, CLIs, admin/private endpoints, health/debug/metrics endpoints.
- Network listeners and exposed ports (where visible in code/config).
- Trust boundaries: what is considered trusted input vs untrusted input (requests, queue messages, configs).

### 2) AuthN / AuthZ (If Applicable)
- Authentication mechanisms (JWT/API keys/mTLS/service identity).
- Authorization checks: where enforced, how consistently applied, default-deny posture.
- Internal/admin endpoints protection (especially health, metrics, debug).

### 3) Input Validation & Injection Risks
- Request parsing and validation (HTTP/gRPC).
- Any query building / DSL parsing (risk of injection or sandbox escape).
- Risks (evaluate only if applicable/detected): datastore query injection (SQL/NoSQL/DSL), SSRF, path traversal, command execution, template injection, regex DoS, unsafe deserialization.
- For consumers: validation of message schema/version and strict handling of unexpected fields.

### 4) Secrets & Sensitive Config
- Secrets sources: env vars, files, config, embedded constants.
- Evidence of secrets in repository: hardcoded keys, test fixtures with real credentials, `.env` committed, sample configs leaking secrets.
- Redaction policy: ensure secrets are not logged/returned in errors.

### 5) Crypto & Transport Security
- TLS usage for external calls (cloud SDKs, datastores, external HTTP/gRPC).
- TLS verification settings (no insecure skip verify, no plaintext fallback).
- Any encryption/hashing usage: correct primitives, no custom crypto, key management strategy.
- At-rest encryption assumptions (if referenced in configs/docs): verify what’s visible in code/config; otherwise GAP.

### 6) External Systems Least-Privilege (As Visible From This Repo Slice)
- Cloud IAM usage patterns (role-based auth vs static keys), scoped permissions as visible in code/config.
- Permissions scoping (resource names, prefixes, actions) where visible.
- Database credentials and privilege scope (read/write separation, least privilege).
- If IAM policies/manifests are outside boundary: record GAP and describe what must be verified outside.

### 7) Dependency & Supply-Chain Hygiene
- `go.mod` / dependency choices: risky libs, abandoned libs, excessive transitive surface.
- Use of `replace` directives, private forks, unpinned versions.
- Build/tooling signals within scope (if present): SAST/secrets scan config, govulncheck, dependency scanning.
- If CI is defined only outside boundary: record GAP and list what checks should exist.

### 8) Privacy & Data Protection
- Identify sensitive data classes handled by service (PII, credentials, tokens, customer data) based on models/events/log fields.
- Data minimization: do we store/log/emit more than needed?
- Retention/deletion: any TTL/cleanup jobs, data lifecycle docs (if any).
- If privacy requirements are not visible in scope: GAP + list minimum expected artifacts.

---

## Tool Usage (MANDATORY)

**DO use these tools:**
- ✅ **Glob** for file patterns: `**/*.go`, `internal/*/`, `cmd/*/main.go`, `**/*.{yaml,yml,json,sql,md}`
- ✅ **Grep** for content search:
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

Answer each checklist question with:
- **File path(s) inspected**
- **Short excerpt** or **line reference** as evidence
- **Assessment:** GOOD / CONCERN / RISK / GAP / N/A

When you write *Risks* and *Action Items*, you MUST follow `skills/_shared/RISK_RUBRIC.md`:
- Risks must include all required fields (Category, Severity S0–S3, Priority P0–P2, Impact, Likelihood L1–L3, Blast Radius B1–B3, Detection D1–D3, Evidence, Recommendation, Verification).
- If any required field cannot be supported within scope: mark it as **GAP** (do not speculate).
- Action items must be specific, bounded, and verifiable (include “Verify”).

Checklist:

| # | Question | How to Verify |
|---|----------|---------------|
| 1 | What are the service entrypoints and exposed surfaces? | Locate `cmd/` mains and server init; list endpoints/servers |
| 2 | What trust boundaries exist? | Identify sources of untrusted input (HTTP/gRPC, consumer messages, env/config) |
| 3 | How is authentication implemented (if applicable)? | Inspect middleware/interceptors; config requirements |
| 4 | How is authorization enforced (if applicable)? | Trace request handling to authz checks; check default-deny |
| 5 | Are inputs validated and safely handled? | Look for validation/schema checks/safe parsing; injection risks |
| 6 | How are secrets handled and protected from leakage? | Find config loading, env vars, logging/redaction, error surfaces |
| 7 | Is transport security correctly configured? | Check TLS settings for external clients and servers |
| 8 | Are external privileges least-privilege (as visible)? | Inspect client init, resource naming, DB user usage; note GAPs |
| 9 | Is dependency hygiene acceptable? | Review `go.mod`, replace directives, scanning hooks (if any) |
| 10 | Is privacy/sensitive data handled safely? | Inspect models/events/logging; check PII/token exposure and retention |

---

## Output

Write exactly ONE report:

- `REPORT_PATH = <target>/docs/rda/reports/08_security_privacy.md`

Report must follow this structure and order:

# Security & Privacy Report

## 0. Run Metadata
- Timestamp (UTC)
- Git Commit (or GAP)

## 1. Context Recap
- 3–7 bullets: what the service does and why security matters here
- Key context from `ARCH_DECISIONS_PATH` (if relevant)

## 2. Scope & Guardrails Confirmation
- Confirm inspection limited to `TARGET_SERVICE_PATH`
- Confirm no external code was opened
- List recorded EXTERNAL DEPs (record-only)
- List notable GAPs caused by boundary (e.g., IAM policies/infra outside scope)

## 3. Pattern Discovery Summary
| Surface | Status (PRESENT/ABSENT/GAP) | Evidence (paths + brief note) |
|---------|------------------------------|-------------------------------|
| Inbound servers (HTTP/gRPC) | ... | ... |
| Consumers/workers | ... | ... |
| AuthN/AuthZ | ... | ... |
| Datastores/caches | ... | ... |
| External HTTP/gRPC calls | ... | ... |
| Secrets/config patterns | ... | ... |
| Sensitive data classes | ... | ... |

## 4. Evidence

### 4.1 Attack Surface Inventory
| Surface | Location | Exposure | Notes | Assessment |
|---------|----------|----------|-------|------------|
| ... | ... | public/private/internal | ... | ... |

### 4.2 Trust Boundaries Snapshot
| Input Source | Trusted? | Validation Present? | Evidence | Risk |
|--------------|----------|---------------------|----------|------|
| ... | ... | ... | ... | ... |

### 4.3 AuthN/AuthZ (If Applicable)
| Control | Where Implemented | Coverage | Evidence | Assessment |
|---------|--------------------|----------|----------|------------|
| Authentication | ... | ... | ... | ... |
| Authorization | ... | ... | ... | ... |
| Admin/private endpoints protection | ... | ... | ... | ... |

If auth is out-of-scope for this service: state explicitly and justify with evidence.

### 4.4 Input Validation & Injection Review
| Vector | Location | Current Handling | Evidence | Assessment | Recommendation |
|--------|----------|------------------|----------|------------|----------------|
| Datastore query injection (store-specific) | ... | ... | ... | ... | ... |
| SSRF | ... | ... | ... | ... | ... |
| Path traversal | ... | ... | ... | ... | ... |
| Regex DoS / parsing | ... | ... | ... | ... | ... |
| Unsafe deserialization | ... | ... | ... | ... | ... |

### 4.5 Secrets & Sensitive Config
| Secret / Sensitive Value | Source | Handling | Leakage Risk | Evidence | Assessment |
|--------------------------|--------|----------|--------------|----------|------------|
| ... | env/config/file | ... | ... | ... | ... |

Include explicit check for: hardcoded secrets, test fixtures, sample configs, logs/errors exposure.

### 4.6 Crypto & Transport Security
| Channel | TLS Enabled? | Verification | Settings (if visible) | Evidence | Assessment |
|---------|--------------|--------------|------------------------|----------|------------|
| External clients | ... | ... | ... | ... | ... |
| Inbound server | ... | ... | ... | ... | ... |

### 4.7 Least-Privilege & External Access (Visible Scope Only)
| System | Auth Mechanism | Scope/Permissions (visible) | Evidence | Assessment | GAPs |
|--------|----------------|-----------------------------|----------|------------|------|
| Cloud SDKs / queues / storage | ... | ... | ... | ... | ... |
| Datastores | ... | ... | ... | ... | ... |

If policies/manifests are outside scope: record as GAP and list required verification questions.

### 4.8 Dependency & Supply-Chain Hygiene
| Item | Evidence | Assessment | Risk | Recommendation |
|------|----------|------------|------|----------------|
| go.mod hygiene | ... | ... | ... | ... |
| replace directives | ... | ... | ... | ... |
| vuln scanning hooks (if any) | ... | ... | ... | ... |

### 4.9 Privacy & Sensitive Data Handling
| Data Type | Where Appears | Stored? | Logged? | Retention Visible? | Evidence | Assessment |
|----------|---------------|---------|--------|--------------------|----------|------------|
| PII/tokens/credentials | ... | ... | ... | ... | ... | ... |

If no PII: justify with evidence; otherwise evaluate minimization and leakage risk.

## 5. Findings

### 5.1 Strengths
- Bullets with evidence references

### 5.2 Risks
All risks MUST be written using `skills/_shared/RISK_RUBRIC.md` required fields.  
Avoid “how-to exploit” instructions; describe risks at a high level with evidence and concrete mitigations.

### 5.3 Gaps
- Items not verifiable within boundary, with what must be checked elsewhere

## 6. Action Items
Use the one-line format from `skills/_shared/RISK_RUBRIC.md`:
`<Action>` | **Priority:** P0/P1/P2 | **Impact:** ... | **Verify:** ...

Keep action items specific and testable.

## 7. Delta vs Previous Run
- If prior report existed: 3–10 bullets on differences
- If first run: "First run — no prior report to compare"
- If no material changes: "No material changes detected" + 3–5 re-checked items

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
9) Risks and action items comply with `skills/_shared/RISK_RUBRIC.md`
10) Delta section present
11) Report saved to `<target>/docs/rda/reports/08_security_privacy.md`
