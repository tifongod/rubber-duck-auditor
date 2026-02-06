# RISK_RUBRIC.md

This rubric is the single source of truth for how rda reports classify risk severity and prioritization (P0/P1/P2).
All audit steps must use this rubric consistently.

---

## 1) Definitions

### Finding
A concrete, evidence-backed observation (can be positive or negative).

### Risk
A negative finding that can lead to an incident, data loss, security/privacy breach, reliability degradation, runaway cost, or serious operational burden.

### Action Item
A concrete fix or verification step derived from a risk (or an important gap).

---

## 2) Required Fields for Every Risk

Every risk entry in any rda report MUST include:

- **Title:** short and specific
- **Category:** one of: `correctness`, `security`, `privacy`, `reliability`, `data`, `operability`, `performance`, `cost`, `maintainability`
- **Severity:** `S0/S1/S2/S3` (see below)
- **Priority:** `P0/P1/P2` (see below)
- **Impact:** what breaks, who is affected, what the worst-case outcome is
- **Likelihood:** how likely in production (based on code/config + typical failure modes)
- **Blast Radius:** scope of impact (single request, single tenant, whole service, whole system)
- **Detection:** how quickly it would be detected (metrics/logs/alerts/runbooks)
- **Evidence:** file path(s) + excerpt/line reference(s)
- **Recommendation:** the fix (or mitigation) in practical terms
- **Verification:** how to prove it’s fixed (tests, metrics, chaos check, load test, etc.)

If any field cannot be supported within scope boundary, mark it as **GAP** (do not speculate).

---

## 3) Severity Scale (S0–S3)

### S0 — Blocker
Unacceptable for production. Likely to cause severe incident or breach.
Examples:
- data loss or silent corruption
- authentication/authorization bypass
- secrets exposure in logs
- unbounded resource growth that can OOM/outage under normal conditions
- missing idempotency/dedup where at-least-once delivery exists and duplicates cause corruption

### S1 — Critical
High production risk. An incident is plausible and impact is serious.
Examples:
- retries without backoff/jitter causing cascading failures
- missing timeouts on external calls on critical path
- incorrect shutdown/ack semantics causing stuck pipeline or backlog explosion
- lack of monitoring for known critical failure modes (especially if ADR requires it)

### S2 — Major
Material quality/operability issues; not an immediate blocker, but will hurt reliability and velocity.
Examples:
- weak config validation leading to misconfig incidents
- poor error mapping causing noisy on-call and slow triage
- missing runbooks for common incidents
- performance inefficiencies likely to cause cost growth or degrade SLO under expected scale

### S3 — Minor
Low risk or mainly hygiene; worth fixing when convenient.
Examples:
- inconsistent naming/style
- minor test gaps with low-risk code
- documentation polish that doesn’t block operations

---

## 4) Likelihood, Blast Radius, Detection (simple scale)

Use these as short qualifiers (do not over-model).

### Likelihood (L)
- **L1 (rare):** requires unusual conditions or misuse
- **L2 (possible):** plausible in real production scenarios
- **L3 (likely):** expected to happen sooner or later under normal operation

### Blast Radius (B)
- **B1 (localized):** single request/job; easy to recover
- **B2 (service-level):** affects significant portion of traffic/jobs, or one major dependency
- **B3 (system-level):** widespread (multi-tenant/system outage, security/privacy impact, or permanent data impact)

### Detection (D)
- **D1 (good):** clear metrics/alerts, fast detection and diagnosis
- **D2 (partial):** logs exist but weak alerts/SLIs; diagnosis takes time
- **D3 (poor):** likely silent or discovered late (e.g., data drift/corruption)

---

## 5) Priority Mapping (P0/P1/P2)

Priority is what you ask the team to do first.

### P0 — Must Fix Before Production / ASAP
Assign **P0** if ANY of the following is true:
- Severity is **S0**
- Severity is **S1** AND (Likelihood is **L2/L3** OR Blast Radius is **B2/B3** OR Detection is **D3**)
- A known ADR-required mitigation is missing AND the ADR ties it to preventing an incident class

### P1 — Fix Soon
Assign **P1** if:
- Severity is **S1** but mitigated by low likelihood AND good detection, OR
- Severity is **S2** with L2/L3 or B2/B3 (material risk), OR
- The issue blocks scaling/operability but isn’t an immediate outage/breach

### P2 — Nice-to-have / Quality Improvements
Assign **P2** if:
- Severity is **S2** but low likelihood and localized, OR
- Severity is **S3**, OR
- It’s mainly cleanup, refactoring, polish, or optional optimizations

---

## 6) Special Labels (use in addition to category)

Use these tags when applicable:

- **DRIFT:** implementation/config contradicts `docs/rda/ARCHITECTURAL_DECISIONS.md`
- **DECISION RISK:** ADR decision itself is risky; explain why and propose safer alternative
- **GAP:** cannot be verified within scope boundary or missing artifacts (tests, docs, config, metrics)

Tags should never replace evidence; they only clarify the nature of the risk.

---

## 7) Action Item Quality Bar

Every Action Item must be:
- **Specific:** “Add timeout to ClickHouse client” not “Improve reliability”
- **Verifiable:** includes a check (test, metric, alert, runbook drill)
- **Bounded:** clear target (component + scope)

Preferred format (one line):
- `<Action>` | **Priority:** P0/P1/P2 | **Impact:** ... | **Verify:** ...

---

## 8) Examples (calibration)

### Example A (P0)
- Missing timeout on external call in hot path.
- Severity: S1, Likelihood: L3, Blast: B2, Detection: D2 → **Priority P0**

### Example B (P1)
- High-cardinality metric labels in hot path.
- Severity: S2, Likelihood: L2, Blast: B2, Detection: D1 → **Priority P1**

### Example C (P2)
- Minor style inconsistencies in non-critical package.
- Severity: S3 → **Priority P2**
