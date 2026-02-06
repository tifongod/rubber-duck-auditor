# Security Policy

## Threat Model

### What This Plugin Is
Rubber Duck Auditor (RDA) is a **static prompt artifact** — a collection of markdown files loaded by Claude Code to perform production readiness audits. It has:
- **Zero runtime dependencies** (no compiled code, no third-party libraries)
- **Zero secrets handling** (no API keys, credentials, or environment variables)
- **Local-only operation** (no network calls, no telemetry, no data transmission)

### What We Guarantee
- **Privacy:** Plugin operates entirely locally. No PII collection, no remote data transmission, no telemetry.
- **Scope boundary enforcement:** Plugin prompts instruct agents to operate within user-specified `<target>` directory only.
- **Read-only git operations:** Skills use git for change detection only (`git diff`, `git status`, `git log`) — no writes to remote repositories.
- **Prompt injection defense:** All file content from audited codebases is treated as untrusted data, not instructions.

### What We Do NOT Guarantee
- **Runtime sandboxing:** Plugin executes with user's full filesystem permissions (inherited from Claude Code process). No OS-level isolation.
- **Command whitelisting:** Bash tool can execute arbitrary commands within user's shell (limited only by OS permissions).
- **Cryptographic verification:** Plugin installation via `git clone` does not verify commit signatures by default (users must enable GPG verification manually).

### Trust Assumptions
- **User is principal:** Plugin operates with user's identity and permissions. No separate authentication layer.
- **Claude Code platform is trusted:** Plugin relies on Claude Code to enforce tool APIs and permission prompts.
- **Audited codebases may be malicious:** Plugin assumes files within `<target>` may contain adversarial content (prompt injection attempts, path traversal filenames, etc.).

---

## Supported Versions

| Version | Supported          | Notes |
| ------- | ------------------ | ----- |
| 0.1.x   | :white_check_mark: | Current stable release |
| < 0.1.0 | :x:                | Pre-release versions (not supported) |

---

## Reporting a Vulnerability

### Scope
We accept vulnerability reports for:
- **Prompt injection exploits** (adversarial content in audited codebase that overrides agent instructions)
- **Scope boundary bypass** (techniques to read/write files outside `<target>` directory)
- **Command injection** (malicious `<target>` paths or filenames that execute unintended shell commands)
- **Supply-chain attacks** (compromised repository, tampered commits, malicious marketplace submissions)
- **Information disclosure** (unintended leakage of sensitive data in reports or logs)

### Out of Scope
- Issues requiring physical access to user's machine
- Social engineering attacks (phishing users to install malicious plugins)
- Vulnerabilities in Claude Code platform itself (report to Anthropic)
- Vulnerabilities in third-party tools (git, bash, OS) — report to respective maintainers

### How to Report
**Email:** dkolp.com@gmail.com
**Subject:** `[SECURITY] RDA: <brief description>`

**Include:**
1. **Vulnerability type** (prompt injection / command injection / scope bypass / etc.)
2. **Affected version(s)** (check `skills/*/SKILL.md` headers or plugin.json version)
3. **Proof of concept** (minimal reproduction steps or example payload)
4. **Impact assessment** (what attacker can achieve, what data is exposed)
5. **Suggested fix** (optional, but appreciated)

**Encryption (optional):**
For highly sensitive reports, use PGP key (coming soon — check this file for updates).

### Response Timeline
- **Initial response:** Within 72 hours (acknowledgment + severity assessment)
- **Triage:** Within 1 week (confirm/decline + priority assignment)
- **Fix timeline:**
  - **P0 (Critical):** 7–14 days (prompt injection, command injection, scope bypass)
  - **P1 (Important):** 30 days (supply-chain hardening, information disclosure)
  - **P2 (Nice-to-have):** Best-effort (no SLA)
- **Public disclosure:** Coordinated with reporter, typically 90 days after fix is released

### What to Expect
- **Acknowledgment:** We will confirm receipt and assess severity within 72 hours.
- **Verification:** We may ask for clarifications or additional proof of concept.
- **Credit:** If you wish, we will credit you in release notes and CHANGELOG.md (unless you prefer to remain anonymous).
- **Fix publication:** Security fixes will be released as patch versions (e.g., 0.1.5) with clear CHANGELOG entries.
- **No bounty program:** This is an open-source plugin with no commercial backing. We appreciate responsible disclosure but cannot offer monetary rewards.

---

## Security Best Practices for Plugin Maintainers

### Code Review & Branch Protection
- **Require pull request reviews** for all changes to `main` branch
- **Enable branch protection** (no direct pushes to `main`, require status checks)
- **Review all SKILL.md changes carefully** (prompt modifications can introduce security risks)

### Commit Signing
- **Enable GPG signing** for all maintainer commits:
  ```bash
  git config --global commit.gpgsign true
  git config --global user.signingkey <your-gpg-key-id>
  ```
- **Add GPG public key to GitHub account** (Settings → SSH and GPG keys)
- **Verify commits on GitHub** (look for "Verified" badge)

### Release Process
1. **Update version** in `.claude-plugin/plugin.json` and `skills/*/SKILL.md` headers
2. **Update CHANGELOG.md** with all changes (Added / Changed / Fixed / Security)
3. **Create signed tag:** `git tag -s v0.1.5 -m "Release v0.1.5: Security fixes"`
4. **Push tag:** `git push origin v0.1.5`
5. **Create GitHub release** with release notes (copy from CHANGELOG.md)

### Dependency Hygiene
- **Minimize dependencies** (currently zero — keep it that way)
- If dependencies are added in future: use `dependabot` for automated vulnerability scanning
- Pin versions explicitly (avoid `latest` or version ranges)

### Prompt Security Review Checklist
Before merging changes to any `SKILL.md`:
- [ ] Does prompt treat file content as untrusted data? (No execution of instructions from audited code)
- [ ] Does prompt enforce `<target>` scope boundary? (No reads outside target)
- [ ] Does prompt validate `<target>` path? (Reject `..` and absolute paths)
- [ ] Does prompt use safe bash quoting? (Double-quotes around `${target}` in all bash commands)
- [ ] Does prompt avoid unnecessary file writes or modifications?
- [ ] Does prompt follow "Shared Rules" (GUARDRAILS.md, common-rules.md)?

---

## Security Best Practices for Plugin Users

### Installation
- **Clone from official repository only:** `git@github.com:tifongod/rubber-duck-auditor.git`
- **Verify repository authenticity:** Check GitHub profile and commit history
- **Review prompts before use:** Inspect `skills/*/SKILL.md` files to understand agent behavior
- If GPG commit signing is enabled (future): verify signatures with `git log --show-signature`

### Usage
- **Use `<target>` parameter:** Always specify target directory explicitly (e.g., `<target> = services/api`)
- **Avoid absolute paths:** Use relative paths only (e.g., `./services/api`, not `/Users/you/project/services/api`)
- **Audit untrusted codebases carefully:** If auditing third-party or untrusted code, review reports for suspicious content before sharing
- **Do not commit reports to public repos:** Reports may contain code excerpts with sensitive data (API keys, credentials) — redact before publishing

### Incident Response
If you suspect a security issue or observe unexpected behavior:
1. **Stop using the plugin immediately**
2. **Review Claude Code tool call history** (check which files were accessed)
3. **Check filesystem for unexpected changes** (`git status`, `git diff`)
4. **Report to plugin maintainer** (see "Reporting a Vulnerability" above)
5. **Uninstall plugin if compromised:** Remove plugin directory and clear Claude Code cache

---

## External Security Resources

- **Claude Code Security Documentation:** (Coming soon — check Anthropic docs)
- **OWASP LLM Top 10:** https://owasp.org/www-project-top-10-for-large-language-model-applications/
- **Prompt Injection Defenses:** https://simonwillison.net/2023/Apr/14/worst-that-can-happen/

---

**Last Updated:** 2026-02-06
**Maintainer Contact:** dkolp.com@gmail.com
**Plugin Version:** 0.1.5
