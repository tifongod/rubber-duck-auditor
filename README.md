# rubber-duck-auditor

Rubber Duck Auditor (RDA) — production readiness audit playbooks and prompt skills for evaluating a single service end-to-end (architecture, config, integrations, data, reliability, observability, testing) and producing an actionable improvement plan.

## What is this?

This repository provides a comprehensive suite of RDA skills under `skills/` for Claude Code. Each skill focuses on a specific aspect of production readiness (security, performance, observability, etc.) and can be invoked independently or as part of a full audit pipeline. Skills generate structured reports with evidence-based findings.

## Skills Overview

RDA includes **10 audit steps**, an **executive summary**, and a **pipeline orchestrator**. Each step produces a standalone Markdown report with evidence-based findings, risk ratings, and action items.

### Audit Steps

| Step | Skill | What It Audits |
|------|-------|----------------|
| 00 | `inventory` | Entrypoints, package structure, dependencies, infra requirements, test layout |
| 01 | `architecture` | Layering, DI pattern, interfaces, cross-cutting concerns (logging, metrics, errors) |
| 02 | `config` | Config loading, env vars, secrets management, validation, defaults |
| 03 | `integrations` | External clients: timeouts, retries, idempotency, TLS, error mapping |
| 04 | `data` | Data model, migrations, idempotency/dedup, SCD Type 2, consistency |
| 05 | `reliability` | Failure handling, backpressure, graceful shutdown, delivery semantics |
| 06 | `observability` | Structured logging, metrics, health checks, tracing, SLI/SLO readiness |
| 07 | `quality` | Test strategy, CI gates, delivery/rollback readiness, maintainability |
| 08 | `security` | Attack surface, authn/authz, injection risks, secrets, crypto, PII |
| 09 | `performance` | Latency/throughput paths, concurrency, batching, memory/CPU hotspots, capacity |

### Meta-Skills

| Skill | What It Does |
|-------|-------------|
| `exec-summary` | Synthesizes all 10 reports into a verdict (Ready / Conditionally ready / Not ready), a 100-point scorecard, and a prioritized roadmap. **Reads reports only — no code.** |
| `audit-pipeline` | Orchestrates all steps in wave-based dependency order with up to 10 subagents. Runs the full audit end-to-end. |

### Execution Order (Waves)

Steps run in dependency waves — each wave completes before the next starts:

```
Wave 1:  00 Inventory
Wave 2:  01 Architecture  |  02 Config          (parallel)
Wave 3:  03 Integrations
Wave 4:  04 Data          |  06 Observability    (parallel)
Wave 5:  05 Reliability
Wave 6:  07 Quality  |  08 Security  |  09 Performance  (parallel)
Wave 7:  Executive Summary
```

### Key Design Principles

- **Evidence-first:** Every finding cites source code (`file.go:12–14 — does X`). Binary confidence: VERIFIED or GAP — never assumed.
- **Scope isolation:** Hard boundary on `<target>` directory. No external file access, path traversal defense, shell escaping enforced.
- **Risk rubric:** Consistent severity (S0–S3) and priority (P0–P2) across all steps via shared `RISK_RUBRIC.md`.
- **ADR-aware:** Steps read `ARCHITECTURAL_DECISIONS.md` and label findings as ALIGNED / DRIFT / DECISION RISK.
- **Iterative:** Reports evolve across runs — delta-first updates, no rewrites.

### Shared Resources (`skills/_shared/`)

| File | Purpose |
|------|---------|
| `common-rules.md` | Mandatory agent rules (paths, evidence format, tool usage, iteration) |
| `GUARDRAILS.md` | Security & scope constraints (path traversal, prompt injection, shell escaping) |
| `PIPELINE.md` | Canonical step order and dependency graph |
| `REPORT_TEMPLATE.md` | Standard report structure (metadata, scope, findings, action items, delta) |
| `RISK_RUBRIC.md` | Severity/priority definitions, risk record format |

### Quick Start

Run a single step:
```
Run an inventory audit on this service
```

Run the full pipeline:
```
Run the full RDA audit pipeline on this service
```

Generate only the executive summary (requires existing reports):
```
Generate the RDA executive summary
```

Reports are saved to `docs/rda/reports/` and `docs/rda/summary/`.

## Installation (Claude Code)

### Step 1: Add the marketplace

```
/plugin marketplace add tifongod/rubber-duck-auditor
```

Alternatively, using the full Git URL:

```
/plugin marketplace add git@github.com:tifongod/rubber-duck-auditor.git
```

### Step 2: Install the plugin

```
/plugin install rda@rubber-duck-auditor
```

### Step 3: Verify installation

Start a new Claude Code session and trigger an RDA skill. Examples:

- "Run a security audit on this service"
- "Audit the observability setup for this repo"
- "Run the full RDA audit pipeline"

## Updating the Plugin

When a new version is released, update the plugin to get the latest features and improvements:

### Option 1: Reinstall (Recommended)

```
/plugin uninstall rda
/plugin install rda@rubber-duck-auditor
```

### Option 2: Update marketplace and reinstall

If the marketplace entry is outdated, refresh it first:

```
/plugin marketplace remove rubber-duck-auditor
/plugin marketplace add tifongod/rubber-duck-auditor
/plugin uninstall rda
/plugin install rda@rubber-duck-auditor
```

### Check current version

After updating, verify the version in any generated report's metadata section, or check:

```
/plugin list
```

## Notes

- **Plugin caching:** Claude Code copies the plugin directory to a cache. The plugin must not depend on files outside its source directory.
- **Skills location:** All skills are located in the `skills/` directory.

## For Maintainers

### Bumping versions

1. Update the version in `.claude-plugin/plugin.json`
2. Commit and push changes
3. Users will need to reinstall or update the plugin to get the new version

### Adding more plugins

To add additional plugins to this marketplace, add new entries to the `plugins` array in `.claude-plugin/marketplace.json` with their own `source` paths.
