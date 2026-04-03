# preflight

A skill for code agents (Claude Code, Codex, Cursor, and others) that runs a security audit before you ship or open-source a project.

## What it checks

**Automated (runs without prompting)**

- Hardcoded secrets — API keys, tokens, passwords in source code
- Git history — credential residue in past commits
- `.gitignore` coverage — missing patterns for env files, certs, logs, build artifacts
- Required files — `LICENSE`, `README.md`, `.env.example`, `CONTRIBUTING.md`
- GitHub Actions — credentials not wrapped in `${{ secrets.xxx }}`
- Internal addresses — private IPs, database connection strings, internal domains hardcoded in application code
- Code comments — passwords, tokens, or IPs embedded in comments
- Dependency licenses — incompatible copyleft licenses against your project's license
- Dependency vulnerabilities — via `pip-audit` / `npm audit`, filtered by severity and relevance

**Manual checklist (output at the end)**

Items AI cannot verify: images/screenshots, business logic in comments, collaborator lists, proprietary code, data handling disclosures.

## How it works

- Results stream per check — no waiting for the full scan to finish
- CRITICAL findings pause the scan and ask whether to fix now or continue
- Dependency tool output is filtered — only actionable issues are surfaced
- Ends with a structured report: Critical / Warning / Pass / Manual

## Usage

Place `SKILL.md` (Chinese) or `SKILL.en.md` (English) in your Claude Code skills directory:

```
~/.claude/skills/security-audit/SKILL.md
```

Then trigger it by asking Claude:

```
Run a security audit on this project before I open-source it.
```

## Supported stacks

Python · Node.js · Go · Rust · Java/Kotlin
