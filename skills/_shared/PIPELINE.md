# PIPELINE.md

This file defines the canonical rda pipeline: what steps exist, what each produces, and the intended execution order.
Steps MAY be run independently, but this order is the default because later steps consume context from earlier reports.

Important:
- Step folder names MUST NOT encode ordering (no numeric prefixes).
- Ordering is defined ONLY here.
- Each step writes exactly one report into: `<target>/docs/rda/reports/`
- All steps must follow: `skills/_shared/common-rules.md` and `skills/_shared/RISK_RUBRIC.md`

---

## Pipeline Overview (Canonical Order)

1) **service-inventory**
    - Goal: boundaries, entrypoints, runtime model, package structure, key files.
    - Output: `00_service_inventory.md`
    - Consumed by: all later steps (as the inventory baseline).

2) **architecture-design**
    - Goal: layering, dependency direction, DI/composition root, cross-cutting placement.
    - Output: `01_architecture_design.md`
    - Consumed by: observability, testing/maintainability, performance (structure-driven analysis).

3) **configuration-environment**
    - Goal: config loading, env var surface, validation, secrets, safe defaults.
    - Output: `02_configuration_environment.md`
    - Consumed by: integrations, security/privacy, reliability, performance (timeouts/limits).

4) **external-integrations**
    - Goal: correctness of external deps usage (timeouts/retries/idempotency/error mapping/security).
    - Output: `03_external_integrations.md`
    - Consumed by: data-handling, reliability, security/privacy, performance.

5) **data-handling**
    - Goal: data model, migrations, idempotency/dedup, caching correctness, SCD2 semantics.
    - Output: `04_data_handling.md`
    - Consumed by: reliability, observability, testing, performance.

6) **reliability-failure-handling**
    - Goal: backpressure, concurrency bounds, graceful shutdown, queue semantics, resilience patterns.
    - Output: `05_reliability_failure_handling.md`
    - Consumed by: observability, performance, testing/maintainability.

7) **observability**
    - Goal: logging/metrics/tracing/health checks, SLI/SLO readiness, diagnosability.
    - Output: `06_observability.md`
    - Consumed by: testing/maintainability, performance, summary.

8) **testing-delivery-maintainability**
    - Goal: test strategy/quality, CI signals (if visible), release/rollback posture, runbooks, maintainability.
    - Output: `07_testing_delivery_maintainability.md`
    - Consumed by: summary.

9) **security-privacy**
    - Goal: authn/authz assumptions, secrets/PII handling, transport security, dependency posture, abuse cases.
    - Output: `08_security_privacy.md`
    - Consumed by: summary.

10) **performance-capacity**
- Goal: performance posture, bottlenecks, resource bounds, capacity assumptions, load-shedding, cost hotspots.
- Output: `09_performance_capacity.md`
- Consumed by: summary.

11) **summary**
- Goal: executive summary + top risks + prioritized improvement plan (consolidation).
- Output: `SUMMARY.md`
- Consumes: all reports 00–09.

---

## Report Naming (Stable Contract)

Reports are named with stable numeric prefixes ONLY to keep chronological ordering inside the reports folder.
Skill names and folders must remain order-agnostic.

Expected files in `<target>/docs/rda/reports/`:
- `00_service_inventory.md`
- `01_architecture_design.md`
- `02_configuration_environment.md`
- `03_external_integrations.md`
- `04_data_handling.md`
- `05_reliability_failure_handling.md`
- `06_observability.md`
- `07_testing_delivery_maintainability.md`
- `08_security_privacy.md`
- `09_performance_capacity.md`
- `SUMMARY.md`

---

## Minimal Preconditions Per Step (Inputs)

- **service-inventory:** only needs code under `<target>`
- **architecture-design:** prefers `00_*`
- **configuration-environment:** prefers `00_*`
- **external-integrations:** prefers `00_*`, `02_*`
- **data-handling:** prefers `00_*`, `03_*` (+ ADRs)
- **reliability:** prefers `00_*`, `03_*`, `04_*`
- **observability:** prefers `00_*`, `01_*` (+ ADR monitoring)
- **testing/maintainability:** prefers `00_*`–`06_*`
- **security/privacy:** prefers `00_*`, `02_*`, `03_*`, `06_*`
- **performance/capacity:** prefers `00_*`, `03_*`, `04_*`, `05_*`, `06_*`
- **summary:** consumes all

If a preferred input is missing, the step must record **GAP** and proceed with a bounded local scan inside `<target>`.

---

## Execution Notes

- Steps MUST respect the scope boundary: never open files outside `<target>`.
- Steps MUST be evidence-backed: file paths + excerpts/line references.
- Steps MUST write one report each; no extra artifacts unless explicitly required by the skill.
- Steps MUST include Run Metadata and Delta vs Previous Run (per shared rules).
