---
name: security-audit
description: Pre-release / open-source security audit. Scans for hardcoded credentials, dependency vulnerabilities, .gitignore gaps, and missing required files. Produces phased interactive results with a final archivable report.
---

# Security Audit Skill

## When to activate

Activate when the user mentions any of: "security audit", "pre-release check", "open source check", "security scan", "any secrets leaked", "ship safely", "preflight check".

---

## Execution Flow

### Step 0: Detect project type

Check for the following files to determine tech stack (multiple may apply):

- `pyproject.toml`, `requirements.txt`, `setup.py` → Python
- `package.json` → Node.js
- `go.mod` → Go
- `Cargo.toml` → Rust
- `pom.xml`, `build.gradle` → Java/Kotlin

Announce before starting:

```
Detected project type: [Python / Node.js / ...]
Starting security scan — 9 automated checks + a manual checklist...
```

---

### Phase 1+2: Automated Scan

Run all checks below sequentially. Output results per check as they complete — do not wait until all checks finish.

**Critical interrupt rule**: When a CRITICAL issue is found, pause and ask:
> Found a CRITICAL issue. Fix it now before continuing, or finish the full scan first? [Fix now / Continue scan]

Only interrupt once per phase — batch multiple CRITICAL findings together if they appear close in sequence.

---

#### 1. Hardcoded Secrets

Use Grep to search the entire codebase with these patterns:

```
(API_KEY|SECRET|PASSWORD|TOKEN|PRIVATE_KEY|api_key|secret_key|access_key|ak|sk)\s*[=:]\s*["'][^"']{8,}["']
```

Exclude from search: `*.example`, `*.md`, `node_modules/`, `__pycache__/`, `.git/`, `dist/`, `build/`

For each match, determine if the value looks like a real credential vs a placeholder:
- Placeholders (skip): `YOUR_KEY`, `<your-key>`, `xxx`, `...`, `CHANGE_ME`, `sk-xxx`
- Real values (flag): anything that looks like an actual key (random chars, starts with known prefixes like `sk-`, `ghp_`, `AKIA`)

Output format:
```
CRITICAL  Hardcoded secrets found
  config.yaml:17    DashScope API Key (sk-4a26...)
  embedding.py:22   SiliconFlow API Key (sk-hgmm...)
```

If clean:
```
PASS      Hardcoded secrets: none found
```

---

#### 2. Git History Scan

Run:
```bash
git log --all -p | grep -iE "(api_key|secret|password|token|private_key)\s*[=:]\s*['\"][^'\"]{8,}"
```

- If findings: CRITICAL — explain that deleting the file or adding a new commit is insufficient; the history must be wiped (git-filter-repo or orphan branch) and the key must be rotated immediately.
- If clean:
```
PASS      Git history: no credential residue found
```

---

#### 3. .gitignore Coverage

Read `.gitignore` and check for the following patterns. Report each missing one as WARNING:

| Pattern | Purpose |
|---|---|
| `.env`, `.env.*`, `.env.local` | Environment variable files |
| `*.pem`, `*.key`, `*.p12`, `*.cer` | Certificates and private keys |
| `*.log` | Runtime logs |
| `*.db`, `*.sqlite` | Local databases |
| `node_modules/` | Node dependencies (Node.js projects) |
| `__pycache__/`, `*.pyc` | Python cache (Python projects) |
| `dist/`, `build/` | Build artifacts |
| `.idea/`, `.vscode/settings.json` | IDE personal config |

If all present:
```
PASS      .gitignore coverage: complete
```

---

#### 4. Required Files

Check existence of:

- `.env.example` — WARNING if missing (provide a variable name template)
- `LICENSE` or `LICENSE.md` — WARNING if missing
- `README.md` — WARNING if missing
- `CONTRIBUTING.md` — WARNING if missing (required for public open-source projects)

Output each as one line:
```
PASS      LICENSE exists (MIT)
WARNING   .env.example missing — add a variable name template for contributors
```

---

#### 5. GitHub Actions Secrets Check

Glob `.github/workflows/*.yml` and Grep for:
- Credentials NOT wrapped in `${{ secrets.xxx }}` syntax
- Pattern to flag: literal key-like strings in env/with blocks

```
PASS      GitHub Actions: all credentials use ${{ secrets.xxx }}
```
or:
```
CRITICAL  GitHub Actions: hardcoded credential found
  .github/workflows/deploy.yml:14   API_KEY = "sk-..."
```

---

#### 6. Internal Addresses and Connection Strings

Grep the entire codebase (excluding `*.md`, `node_modules/`, `.git/`) for:

- Private IP ranges: `192\.168\.`, `10\.\d+\.`, `172\.(1[6-9]|2\d|3[01])\.`
- Localhost variants that may indicate hardcoded dev endpoints: `localhost`, `127\.0\.0\.1` (flag only if in non-test, non-config-example files)
- Database connection strings: `postgresql://`, `mysql://`, `mongodb://`, `redis://`, `jdbc:`
- Internal domain suffixes: `\.internal`, `\.local`, `\.corp`, `\.intra`

For each match, judge whether it belongs in source code or should be in config/env:
- In a config example file or `.env.example`: skip
- Hardcoded in application code: WARNING

```
WARNING   Internal address or connection string found
  src/db/client.py:8    postgresql://admin:pass@192.168.1.10:5432/prod
  → Move to environment variable to avoid exposing internal network topology
```

If clean:
```
PASS      Internal addresses / connection strings: none found
```

---

#### 7. Sensitive Content in Code Comments

Grep comments (lines starting with `#`, `//`, `/*`, `*`) for patterns suggesting sensitive technical information:

- Password-like patterns: `password`, `passwd`, `pwd` followed by actual values
- IP addresses or connection strings embedded in comments
- Tokens or keys mentioned inline

Note: Only flag technically sensitive content (credentials, addresses). Do NOT attempt to judge whether business logic in comments is "sensitive" — that requires human review and will be noted in the manual checklist.

```
WARNING   Sensitive technical content in code comments
  src/api/auth.py:42   # default password is Abc123!
  → Remove credentials from comments
```

If clean:
```
PASS      Code comments: no sensitive technical content found
```

---

#### 8. Dependency License Compatibility

Run the appropriate tool based on detected project type:

- **Python**: `pip-licenses --format=markdown` (install via `pip install pip-licenses` if not available)
- **Node.js**: `npx license-checker --summary`
- **Other**: skip with note

Compare detected licenses against the project's own license using this compatibility reference:

| Project License | Potentially incompatible dependency licenses |
|---|---|
| MIT / Apache 2.0 | GPL-2.0-only, GPL-3.0-only, AGPL-3.0 |
| GPL-2.0 | Apache-2.0 (one-way), AGPL-3.0 |
| AGPL-3.0 | generally compatible with GPL/LGPL/MIT; flag proprietary licenses |
| Proprietary | GPL, AGPL, LGPL (copyleft licenses) |

Flag any dependency whose license is potentially incompatible as WARNING. If the tool is unavailable, note it and skip.

```
WARNING   License compatibility risk
  somelib 1.2.0   GPL-3.0-only
  → Project is MIT; GPL dependency may introduce copyleft obligations — confirm with legal or replace

PASS      42 other dependencies have compatible licenses (MIT / Apache-2.0 / BSD)
```

---

#### 9. Dependency Vulnerabilities

Run the appropriate tool based on detected project type:

- **Python**: `pip-audit`
- **Node.js**: `npm audit`
- **Go**: `govulncheck ./...` (if installed, else skip with note)
- **Rust**: `cargo audit` (if installed, else skip with note)

**Filtering rules** — do not dump raw output. Classify and present:

| Severity | Action |
|---|---|
| Critical / High + fix available | CRITICAL — describe affected path, provide fix version |
| High + no fix available | WARNING — note no fix available |
| Medium | WARNING — one-line description of impact |
| Low / Info | Ignore, do not surface |
| Dev-only dependency | Downgrade by one level (High → Warning, Medium → ignore) |

For ambiguous cases where impact depends on usage (e.g., proxy auth, specific API endpoints), ask one targeted yes/no question before classifying.

Example output:
```
WARNING   pip-audit: 2 actionable vulnerabilities

  cryptography 41.0.0   CVE-2024-xxxx  HIGH
    Impact: RSA decryption path — upgrade to 42.0.0

  setuptools 65.0.0     CVE-2022-xxxx  LOW (build-time only, filtered)
    No runtime impact — excluded from report
```

If no actionable issues:
```
PASS      Dependency vulnerabilities: nothing actionable found
```

---

### Phase 3: Manual Checklist

After automated scan completes, output this section as a static recommendation list — do not ask questions interactively:

```
MANUAL    The following items require human review (AI cannot verify automatically)

  [ ] Images and screenshots contain no internal IPs, internal emails, or unpublished data
  [ ] Code comments contain no sensitive business logic (technical secrets already scanned)
  [ ] GitHub Collaborators list is clean — remove anyone no longer on the project
  [ ] Code contains no proprietary code from a previous employer
  [ ] If the project handles user data, README explains data handling practices
  [ ] The project runs end-to-end in a fresh environment following the README
```

---

### Final Report

Output a consolidated archivable report at the end:

```
## Security Audit Report · [Project Name] · [YYYY-MM-DD]

Summary: [One-line verdict, e.g. "2 critical issues found — fix before shipping" or "No critical issues — ready to ship"]

---

### Critical Issues (must fix)

[Write "None" if empty]

1. [Issue title]
   [File path or location]
   → [Specific action to take]

### Warnings (recommended fixes)

[Write "None" if empty]

- [Issue description] → [Recommended action]

### Passed

[All PASS items on one comma-separated line — no detail needed]

### Manual Checklist

[ ] Images and screenshots contain no internal IPs, internal emails, or unpublished data
[ ] Code comments contain no sensitive business logic (technical secrets already scanned)
[ ] GitHub Collaborators list is clean — remove anyone no longer on the project
[ ] Code contains no proprietary code from a previous employer
[ ] If the project handles user data, README explains data handling practices
[ ] The project runs end-to-end in a fresh environment following the README
```

---

## Output Rules

1. **Stream results**: Output each check result immediately — do not wait for all checks to finish
2. **Interrupt sparingly**: Batch CRITICAL findings together — do not pause after every single issue
3. **Filter noise**: Never dump raw tool output — AI filters and summarizes before presenting
4. **Keep PASS items brief**: One line per passing check; consolidate in final report
5. **Always give an action**: Every WARNING/CRITICAL item must say what to do, not just what's wrong
6. **Phase 3 is non-interactive**: Output the manual checklist as a static list — do not ask questions
