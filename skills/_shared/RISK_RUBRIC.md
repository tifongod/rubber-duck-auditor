# RISK_RUBRIC.md

This rubric is the single source of truth for how rda reports classify risk severity and prioritization (P0/P1/P2).
All audit steps must use this rubric consistently.

This version is **universal**: it is not tied to any specific architecture, language, datastore, queue, or deployment model.

---

## 1) Definitions

### Finding
A concrete, evidence-backed observation (positive or negative) derived from code/config/docs within the scope boundary.

### Risk
A negative finding that can lead to an incident, data loss/corruption, security/privacy breach, reliability degradation, runaway cost, severe operability burden, or significant loss of developer velocity.

### Action Item
A concrete fix or verification step derived from a risk (or an important gap). Action items must be specific, bounded, and verifiable.

---

## 2) Required Fields for Every Risk

Every risk entry in any rda report MUST include:

- **Title:** short and specific
- **Category:** one of: `correctness`, `security`, `privacy`, `reliability`, `data`, `operability`, `performance`, `cost`, `maintainability`
- **Severity:** `S0/S1/S2/S3` (see below)
- **Priority:** `P0/P1/P2` (see below)
- **Impact:** what breaks, who is affected, and worst-case outcome
- **Likelihood:** how likely in production based on code/config + typical failure modes (`L1/L2/L3`)
- **Blast Radius:** scope of impact (`B1/B2/B3`)
- **Detection:** how quickly it would be detected (`D1/D2/D3`) using metrics/logs/alerts/runbooks or lack thereof
- **Evidence:** file path(s) + excerpt/line reference(s)
- **Recommendation:** the fix (or mitigation) in practical terms
- **Verification:** how to prove it’s fixed (tests, metrics/alerts, runbook drill, chaos check, load test, etc.)
- **Tags:** optional labels (see §6)

If any field cannot be supported within the scope boundary, mark it as **GAP** (do not speculate).

---

## 3) Severity Scale (S0–S3)

Severity is about **how bad the outcome can be** if the risk triggers.

### S0 — Blocker
Unacceptable for production. Likely to cause severe incident, breach, or irreversible harm.
Examples (non-exhaustive, universal):
- data loss, silent corruption, irreversible divergence between systems
- authentication/authorization bypass, privilege escalation
- secrets/PII exposure (logs, responses, configs, artifacts)
- unbounded resource growth that can OOM/outage under normal or expected load
- unsafe concurrency that can corrupt state or cause runaway amplification
- missing idempotency/deduplication where duplicates can corrupt state under at-least-once delivery
- destructive migration/maintenance operation without safeguards that can irreversibly break production

### S1 — Critical
High production risk. An incident is plausible and impact is serious, but not necessarily catastrophic/irreversible.
Examples:
- missing timeouts/deadlines on external calls on critical paths
- retry storms (no backoff/jitter) causing cascading failures
- incorrect shutdown/ack semantics causing stuck pipelines, backlog explosions, or prolonged partial outages
- lack of monitoring/alerts for known critical failure modes (especially when ADRs require mitigations)
- unsafe default limits (too high) that can overload dependencies or cause cost spikes

### S2 — Major
Material quality/operability issues; not an immediate blocker, but will hurt reliability, cost, or delivery velocity.
Examples:
- weak config validation leading to misconfiguration incidents
- poor error taxonomy/mapping causing noisy on-call and slow triage
- missing runbooks for common incidents, weak operational docs for critical components
- performance inefficiencies likely to degrade SLOs or increase cost at expected scale
- insufficient isolation (e.g., noisy-neighbor risk) without immediate outage certainty

### S3 — Minor
Low risk or mainly hygiene; worth fixing when convenient.
Examples:
- inconsistent naming/style that reduces readability
- minor test gaps in low-risk code paths
- documentation polish that does not block operation or correctness

---

## 4) Likelihood, Blast Radius, Detection (simple scale)

Use these as short qualifiers. Do not over-model.

### Likelihood (L)
- **L1 (rare):** requires unusual conditions, misuse, or multiple coincident failures
- **L2 (possible):** plausible in real production scenarios
- **L3 (likely):** expected to happen sooner or later under normal operation, expected load, or routine incidents

### Blast Radius (B)
- **B1 (localized):** single request/job/tenant; easy to recover
- **B2 (service-level):** affects significant portion of traffic/jobs or one major dependency
- **B3 (system-level):** widespread (multi-tenant/system outage, security/privacy impact, or permanent data impact)

### Detection (D)
- **D1 (good):** clear metrics/alerts/runbooks → fast detection and diagnosis
- **D2 (partial):** logs exist but weak alerts/SLIs/runbooks → diagnosis takes time
- **D3 (poor):** likely silent or discovered late (e.g., data drift/corruption, latent security issue)

---

## 5) Priority Mapping (P0/P1/P2)

Priority is what you ask the team to do first.

### P0 — Must Fix Before Production / ASAP
Assign **P0** if ANY of the following is true:
- Severity is **S0**
- Severity is **S1** AND (**Likelihood** is **L2/L3** OR **Blast Radius** is **B2/B3** OR **Detection** is **D3**)
- A known ADR-required mitigation is missing AND the ADR ties it to preventing a specific incident class

### P1 — Fix Soon
Assign **P1** if:
- Severity is **S1** but mitigated by **low likelihood** AND **good detection**, OR
- Severity is **S2** with **L2/L3** or **B2/B3** (material risk), OR
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
- **GAP:** cannot be verified within scope boundary or missing artifacts. Use when:
  - Evidence would require reading files outside `<target>/`
  - Required artifact doesn't exist (tests, docs, config, metrics, migration, runbook)
  - Information exists only in runtime/logs/metrics, not in code/config/docs
  - External system behavior/policy is unknowable from service code alone  
    Format: `GAP: <what is unknown> — <why it matters> — <what evidence is missing>`
- **EXTERNAL DEP:** reference to code/system outside `<target>/`. Record import path or endpoint; do NOT open external code.

Tags never replace evidence; they only clarify the nature of the risk.

---

## 7) Action Item Quality Bar

Every Action Item must be:
- **Specific:** “Add timeout to <dependency client>” not “Improve reliability”
- **Verifiable:** includes a concrete check (test, metric, alert, runbook drill, chaos, load test)
- **Bounded:** clear target (component + scope)

Preferred format (one line):
- `<Action>` | **Priority:** P0/P1/P2 | **Impact:** ... | **Verify:** ...

---

## 8) Examples (calibration)

### Example A (P0)
- Missing timeout/deadline on an external call in a hot path.
- Severity: S1, Likelihood: L3, Blast: B2, Detection: D2 → **Priority P0**

### Example B (P1)
- High-cardinality metric labels in a hot path.
- Severity: S2, Likelihood: L2, Blast: B2, Detection: D1 → **Priority P1**

### Example C (P2)
- Minor style inconsistencies in non-critical package.
- Severity: S3 → **Priority P2**
