# Changelog

All notable changes to the Rubber Duck Auditor (RDA) plugin will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.8] - 2026-02-06

### Added
- Path hygiene enforcement via `<run_root>` concept in `skills/_shared/common-rules.md`
- Architectural decisions document at `docs/rda/ARCHITECTURAL_DECISIONS.md` with 8 ADRs
- `.claude/settings.json` for project-level plugin enablement
- `.claude/settings.example.json` template for contributor setup

### Changed
- Renamed all skill directories from `rda-<name>` to `<name>` (removed redundant prefix)
- Renamed `skills/_shared/rda-common-rules.md` to `common-rules.md`
- Simplified Run Metadata section (removed Change Detection field)
- Reports now use relative paths only (no absolute paths)

### Fixed
- Absolute path violations in prior reports (now use `.` as repository root)
- Path references after skill directory renaming
- Version alignment across all configuration files

## [0.1.2] - 2026-02-05

### Added
- Project-level settings file (`.claude/settings.json`)
- Enhanced security documentation in `SECURITY.md`

### Changed
- Expanded shared rules and guardrails
- Report template version updated

## [0.1.1] - 2026-02-04

### Added
- Initial release of RDA plugin with 12 audit skills
- 10 production readiness audit step skills (00-09)
- Audit pipeline orchestrator skill
- Executive summary generator skill
- Shared framework files (common-rules, guardrails, templates, pipeline, rubrics)
- Security documentation and trust model

### Security
- Zero runtime dependencies
- Zero secrets handling
- Local-only operation (no network calls)
- Prompt injection defense mechanisms
- Scope boundary enforcement
