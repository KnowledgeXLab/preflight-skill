---
name: security-audit
description: Pre-release / open-source security audit. Scans for hardcoded credentials, dependency vulnerabilities, .gitignore gaps, and missing required files. Produces phased interactive results with a final archivable report.
---

# Security Audit Skill

## When to activate

Activate when the user mentions any of: "安全检查", "security audit", "pre-release", "准备开源", "密钥泄漏", "open source check", "上线前检查", "security scan", "有没有泄漏".

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
检测到项目类型：[Python / Node.js / ...]
开始安全扫描，共两个自动阶段 + 一份人工自查清单...
```

---

### Phase 1+2: Automated Scan

Run all checks below sequentially. Output results per check as they complete — do not wait until all checks finish.

**Critical interrupt rule**: When a CRITICAL issue is found, pause and ask:
> 发现 CRITICAL 问题，是否先修复再继续，还是扫描完成后统一处理？[先修复 / 继续扫描]

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
CRITICAL  硬编码密钥
  config.yaml:17    DashScope API Key (sk-4a26...)
  embedding.py:22   SiliconFlow API Key (sk-hgmm...)
```

If clean:
```
PASS      硬编码密钥扫描：未发现
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
PASS      Git 历史扫描：未发现密钥残留
```

---

#### 3. .gitignore Coverage

Read `.gitignore` and check for the following patterns. Report each missing one as WARNING:

| Pattern | Purpose |
|---|---|
| `.env`, `.env.*`, `.env.local` | 环境变量文件 |
| `*.pem`, `*.key`, `*.p12`, `*.cer` | 证书与私钥 |
| `*.log` | 运行时日志 |
| `*.db`, `*.sqlite` | 本地数据库 |
| `node_modules/` | Node 依赖（Node.js 项目） |
| `__pycache__/`, `*.pyc` | Python 缓存（Python 项目） |
| `dist/`, `build/` | 构建产物 |
| `.idea/`, `.vscode/settings.json` | IDE 个人配置 |

If all present:
```
PASS      .gitignore 配置：完整
```

---

#### 4. Required Files

Check existence of:

- `.env.example` — WARNING if missing（应提供变量名模板）
- `LICENSE` or `LICENSE.md` — WARNING if missing
- `README.md` — WARNING if missing
- `CONTRIBUTING.md` — WARNING if missing（对外开源项目）

Output each as one line:
```
PASS      LICENSE 存在（AGPL-3.0）
WARNING   .env.example 缺失，建议提供变量名模板
```

---

#### 5. GitHub Actions Secrets Check

Glob `.github/workflows/*.yml` and Grep for:
- Credentials NOT wrapped in `${{ secrets.xxx }}` syntax
- Pattern to flag: literal key-like strings in env/with blocks

```
PASS      GitHub Actions：凭证引用规范
```
or:
```
CRITICAL  GitHub Actions：发现硬编码凭证
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
WARNING   发现内网地址或连接串
  src/db/client.py:8    postgresql://admin:pass@192.168.1.10:5432/prod
  → 移至环境变量，避免暴露内部网络结构
```

If clean:
```
PASS      内网地址 / 连接串扫描：未发现
```

---

#### 7. Sensitive Content in Code Comments

Grep comments (lines starting with `#`, `//`, `/*`, `*`) for patterns suggesting sensitive technical information:

- Password-like patterns: `password`, `passwd`, `pwd` followed by actual values
- IP addresses or connection strings embedded in comments
- Tokens or keys mentioned inline

Note: Only flag technically sensitive content (credentials, addresses). Do NOT attempt to judge whether business logic in comments is "sensitive" — that requires human review and will be noted in the manual checklist.

```
WARNING   代码注释中发现技术敏感信息
  src/api/auth.py:42   # default password is Abc123!
  → 移除注释中的凭证信息
```

If clean:
```
PASS      代码注释扫描：未发现技术敏感信息
```

---

#### 8. Dependency License Compatibility

Run the appropriate tool based on detected project type:

- **Python**: `pip-licenses --format=markdown` (install via `pip install pip-licenses` if not available)
- **Node.js**: `npx license-checker --summary`
- **Other**: skip with note

Compare detected licenses against the project's own license using this compatibility reference:

| Project License | Incompatible dependency licenses |
|---|---|
| MIT / Apache 2.0 | GPL-2.0-only, GPL-3.0-only, AGPL-3.0 |
| GPL-2.0 | Apache-2.0 (one-way), AGPL-3.0 |
| AGPL-3.0 | generally compatible with GPL/LGPL/MIT, flag proprietary licenses |
| Proprietary | GPL, AGPL, LGPL (copyleft licenses) |

Flag any dependency whose license is potentially incompatible as WARNING. If `pip-licenses` is unavailable, note it and skip.

```
WARNING   License 兼容性风险
  somelib 1.2.0   GPL-3.0-only
  → 项目为 MIT 协议，GPL 依赖可能引入传染性约束，建议法务确认或替换依赖

PASS      其余 42 个依赖 License 兼容（MIT / Apache-2.0 / BSD）
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
| Critical / High + fix available | CRITICAL，说明受影响路径，给出升级版本 |
| High + no fix | WARNING，说明无可用修复 |
| Medium | WARNING，一句话说明影响范围 |
| Low / Info | 忽略，不展示 |
| Dev-only dependency | 降一级处理（High → Warning，Medium → 忽略） |

For ambiguous cases where impact depends on usage (e.g., proxy auth, specific API endpoints), ask one targeted yes/no question before classifying.

Example output:
```
WARNING   pip-audit 发现 2 个需关注的漏洞

  cryptography 41.0.0   CVE-2024-xxxx  高危
    影响：RSA 解密路径，建议升级到 42.0.0

  setuptools 65.0.0     CVE-2022-xxxx  低危（构建期依赖，运行时不影响，已过滤）
```

If no actionable issues:
```
PASS      依赖漏洞扫描：无需关注的漏洞
```

---

### Phase 3: Manual Checklist

After automated scan completes, output this section as a static recommendation list — do not ask questions interactively:

```
MANUAL    以下项目需团队人工自查（AI 无法自动验证）

  [ ] 图片、截图等静态资源中无内网地址、内部邮箱、未公开数据
  [ ] 代码注释中无业务敏感逻辑（技术性敏感信息已自动扫描）
  [ ] GitHub Collaborators 列表已清理，移除不再参与项目的成员
  [ ] 代码中无来自前公司或其他雇主的专有代码
  [ ] 若涉及用户数据，README 中已说明数据处理方式
  [ ] 已在全新环境按 README 步骤完整运行一遍，确认可正常启动
```

---

### Final Report

Output a consolidated archivable report at the end:

```
## 安全审查报告 · [项目名] · [YYYY-MM-DD]

结论：[一句话总结，例如"发现 2 个严重问题，建议修复后再发布"或"未发现严重问题，可以开源/上线"]

---

### 严重问题（必须修复）

[若无则写"无"]

1. [问题标题]
   [文件路径或具体位置]
   → [具体操作建议]

### 警告（建议修复）

[若无则写"无"]

- [问题描述] → [建议操作]

### 通过项

[所有 PASS 项合并为一行逗号分隔，不展开细节]

### 人工自查清单

[ ] 图片、截图等静态资源中无内网地址、内部邮箱、未公开数据
[ ] 代码注释中无业务敏感逻辑（技术性敏感信息已自动扫描）
[ ] GitHub Collaborators 列表已清理，移除不再参与项目的成员
[ ] 代码中无来自前公司或其他雇主的专有代码
[ ] 若涉及用户数据，README 中已说明数据处理方式
[ ] 已在全新环境按 README 步骤完整运行一遍，确认可正常启动
```

---

## Output Rules

1. **即时输出**：每项检查完成后立即输出结果，不等待全部完成
2. **只打扰一次**：CRITICAL 问题合并询问，不要每发现一个就打断一次
3. **过滤噪音**：依赖漏洞等工具的原始输出不直接展示，AI 先过滤再呈现
4. **PASS 项简洁**：通过的项一行即可，最终报告合并展示
5. **给出 action**：每个 WARNING/CRITICAL 项都要说"建议做什么"，不只是报告问题
6. **Phase 3 不交互**：人工自查清单直接输出，不逐条询问用户
