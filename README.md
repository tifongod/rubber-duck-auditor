# rubber-duck-auditor

Rubber Duck Auditor (RDA) — production readiness audit playbooks and prompt skills for evaluating a single service end-to-end (architecture, config, integrations, data, reliability, observability, testing) and producing an actionable improvement plan.

## What is this?

This repository provides a comprehensive suite of RDA skills under `skills/` for Claude Code. Each skill focuses on a specific aspect of production readiness (security, performance, observability, etc.) and can be invoked independently or as part of a full audit pipeline. Skills generate structured reports with evidence-based findings.

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

## Notes

- **Plugin caching:** Claude Code copies the plugin directory to a cache. The plugin must not depend on files outside its source directory.
- **Skills location:** All skills are located in the `skills/` directory and follow a consistent naming pattern (`rda-*`).

## For Maintainers

### Bumping versions

1. Update the version in `.claude-plugin/plugin.json`
2. Commit and push changes
3. Users will need to reinstall or update the plugin to get the new version

### Adding more plugins

To add additional plugins to this marketplace, add new entries to the `plugins` array in `.claude-plugin/marketplace.json` with their own `source` paths.
