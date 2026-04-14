---
name: codebase-scanner
description: Scan a full local directory or a GitHub repository URL for security vulnerabilities — file discovery, triage, per-file OWASP analysis, and a consolidated report
license: MIT
metadata:
  author: jay-shah
  version: "1.0.0"
  category: security
---

# Codebase Scanner

## When to Use
When the user provides:
- A **local directory path** — e.g., `scan ./my-app` or `audit /home/user/project`
- A **GitHub URL** — e.g., `scan https://github.com/owner/repo`
- A **branch-specific GitHub URL** — e.g., `https://github.com/owner/repo/tree/main`

Do NOT use this skill when the user pastes a single file or snippet — use `owasp-scanner` for that.

---

## Instructions

### Phase 0 — Determine Input Type

**If the input is a GitHub URL:**

```bash
# Clone with depth 1 (no full history needed)
git clone --depth 1 <repo-url> /tmp/security-scan-<repo-name>
cd /tmp/security-scan-<repo-name>
```

For a specific branch:
```bash
git clone --depth 1 --branch <branch-name> <repo-url> /tmp/security-scan-<repo-name>
```

If `git` is not available:
```bash
# Fallback: use GitHub archive download
curl -L https://github.com/<owner>/<repo>/archive/refs/heads/main.zip -o /tmp/repo.zip
unzip /tmp/repo.zip -d /tmp/security-scan-<repo-name>
```

Set `SCAN_ROOT=/tmp/security-scan-<repo-name>` and proceed to Phase 1.

**If the input is a local path:**
Set `SCAN_ROOT=<provided-path>` and proceed to Phase 1.

---

### Phase 1 — File Discovery

List all source files in the codebase. Skip everything that is not application code.

**Directories to always skip:**
```
node_modules/
.git/
dist/
build/
.next/
.nuxt/
__pycache__/
.pytest_cache/
.mypy_cache/
.tox/
venv/
env/
.venv/
vendor/
target/       (Java/Rust build output)
.gradle/
coverage/
htmlcov/
*.egg-info/
```

**File extensions to scan (by priority):**

| Priority | Extensions | Why |
|----------|------------|-----|
| P1 — Always | `.py`, `.js`, `.ts`, `.go`, `.java`, `.rb`, `.php`, `.cs` | Application logic |
| P1 — Always | `.env`, `.env.*`, `*.config.js`, `*.settings.py` | Config & secrets |
| P1 — Always | `requirements.txt`, `package.json`, `go.mod`, `Gemfile`, `pom.xml`, `build.gradle` | Dependencies |
| P2 — High | `.jsx`, `.tsx`, `.mjs`, `.cjs`, `.coffee` | Frontend logic |
| P2 — High | `*.yaml`, `*.yml`, `*.toml`, `*.ini`, `*.cfg` | Configuration |
| P2 — High | `Dockerfile`, `docker-compose*.yml`, `.github/workflows/*.yml` | Infrastructure |
| P3 — Normal | `.html`, `.ejs`, `.hbs`, `.jinja`, `.j2` | Templates |
| P3 — Normal | `.sh`, `.bash`, `.zsh`, `Makefile` | Scripts |
| Skip | `*.min.js`, `*.min.css`, `*.map`, `*.lock`, `*.sum` | Generated/binary |
| Skip | `*.png`, `*.jpg`, `*.gif`, `*.ico`, `*.woff`, `*.ttf` | Assets |
| Skip | `*.md`, `*.rst`, `*.txt` (except requirements.txt) | Docs |
| Skip | `*.test.*`, `*.spec.*`, `*_test.go`, `test_*.py` | Tests (scan separately if asked) |

**Run discovery:**
```bash
find $SCAN_ROOT -type f \( \
  -name "*.py" -o -name "*.js" -o -name "*.ts" -o -name "*.go" \
  -o -name "*.java" -o -name "*.rb" -o -name "*.php" -o -name "*.cs" \
  -o -name "*.jsx" -o -name "*.tsx" -o -name "*.env" \
  -o -name "requirements.txt" -o -name "package.json" -o -name "go.mod" \
  -o -name "Dockerfile" -o -name "docker-compose*.yml" \
  -o -name "*.yaml" -o -name "*.yml" -o -name "*.sh" \
\) \
  ! -path "*/node_modules/*" \
  ! -path "*/.git/*" \
  ! -path "*/dist/*" \
  ! -path "*/build/*" \
  ! -path "*/__pycache__/*" \
  ! -path "*/venv/*" \
  ! -path "*/vendor/*" \
  ! -name "*.min.js" \
  ! -name "*.lock" \
  | sort
```

Report the file list to the user: total count, breakdown by type, and the scan root.

---

### Phase 2 — Triage (High-Risk File Identification)

Before reading every file, do a fast grep pass to identify the highest-risk files.
Scan these patterns across all discovered files:

```bash
# SQL injection indicators
grep -rl --include="*.py" --include="*.js" --include="*.ts" --include="*.java" \
  -e "execute(" -e "query(" -e "cursor.execute" -e "db.query" \
  $SCAN_ROOT | head -20

# Hardcoded secrets indicators
grep -rl -e "secret" -e "password" -e "api_key" -e "ACCESS_KEY" -e "token" \
  --include="*.py" --include="*.js" --include="*.ts" --include="*.env" \
  $SCAN_ROOT | head -20

# Dangerous function calls
grep -rl -e "eval(" -e "exec(" -e "shell=True" -e "os.system(" \
  --include="*.py" --include="*.js" --include="*.rb" \
  $SCAN_ROOT | head -20

# Deserialization
grep -rl -e "pickle.loads" -e "yaml.load(" -e "Marshal.load" -e "deserialize" \
  $SCAN_ROOT | head -20

# SSRF indicators
grep -rl -e "requests.get" -e "urllib.open" -e "fetch(" -e "http.get(" \
  $SCAN_ROOT | head -20
```

Create a priority scan list. Files matching patterns above get scanned first.

---

### Phase 3 — Per-File Scan

Read and analyze each file. Apply the full OWASP Top 10 checklist from the
`owasp-scanner` skill to each file. Also apply `secrets-detector` patterns.

**For each file, output:**
```
### File: src/routes/users.py

[CRITICAL] A03:2021 — Injection | Line 47
  query = "SELECT * FROM users WHERE id = " + user_id
  Fix: cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))

[HIGH] A07:2021 — Auth Failures | Line 112
  Session cookie set without Secure flag
  Fix: response.set_cookie('session', value, secure=True, httponly=True)

No other findings.
```

If a file has zero findings, write:
```
### File: src/utils/formatting.py — Clean ✓
```

**Scan order:**
1. P1 files with triage hits first
2. P1 files without triage hits
3. Dependency files (requirements.txt, package.json, go.mod)
4. P2 files
5. Infrastructure files (Dockerfile, compose, CI/CD workflows)
6. P3 files

**Dependency files** — use `dep-auditor` skill for these. Do not just skim them.

---

### Phase 4 — Consolidated Report

After scanning all files, produce the full report:

```markdown
# Security Audit Report
**Repository:** <url or path>
**Scanned:** <timestamp>
**Files scanned:** N total (N Python, N JavaScript, N config, ...)
**Files with findings:** N

---

## Critical Findings

### [CRITICAL] SQL Injection
**File:** src/db/users.py:47
**Category:** A03:2021 — Injection
**Evidence:**
    query = "SELECT * FROM users WHERE id = " + user_id
**Why it's exploitable:** An attacker can pass `user_id = "1 OR 1=1"` to dump all users
**Fix:**
    cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))

---

## High Findings
[...same format...]

---

## Medium Findings
[...same format...]

---

## Dependency Vulnerabilities
| Package | Installed | CVE | Severity | Fix |
|---------|-----------|-----|----------|-----|
| requests | 2.18.0 | CVE-2023-32681 | HIGH | >=2.32.0 |

---

## Infrastructure & CI/CD Issues
[Dockerfile misconfigs, exposed ports, overly permissive CI workflows]

---

## Summary

| Severity | Count | Files Affected |
|----------|-------|----------------|
| CRITICAL | N     | file1.py, file2.js |
| HIGH     | N     | file3.py |
| MEDIUM   | N     | ... |
| LOW      | N     | ... |
| **Total**| **N** | **N files** |

## Risk Assessment
- **Overall risk:** CRITICAL / HIGH / MEDIUM / LOW
- **Most urgent fix:** [single most dangerous finding]
- **Estimated remediation effort:** [small/medium/large — explain why]
```

---

### Phase 5 — Cleanup (MANDATORY — runs after every GitHub URL scan)

This phase is **not optional**. Run it immediately after the report is delivered,
every time, without waiting for the user to ask.

**Step 1 — Remove the cloned repository permanently**

```bash
# Linux / macOS
rm -rf /tmp/security-scan-<repo-name>

# Windows (PowerShell)
Remove-Item -Recurse -Force "C:\Users\<user>\AppData\Local\Temp\security-scan-<repo-name>"

# Windows (CMD)
rmdir /s /q "C:\Users\<user>\AppData\Local\Temp\security-scan-<repo-name>"
```

**Step 2 — Verify it is gone**

```bash
# Linux / macOS — should return nothing / "No such file or directory"
ls /tmp/security-scan-<repo-name>

# Windows PowerShell — should return nothing
Test-Path "C:\...\security-scan-<repo-name>"
# Expected: False
```

If the directory still exists after the first delete attempt, force-remove:
```bash
# Linux / macOS
find /tmp/security-scan-<repo-name> -type f -delete
find /tmp/security-scan-<repo-name> -type d -delete

# Windows PowerShell
Get-ChildItem -Path "C:\...\security-scan-<repo-name>" -Recurse | Remove-Item -Force -Recurse
```

**Step 3 — Confirm deletion to the user**

After successful cleanup, always end with this message:
```
Scan complete. The cloned repository has been permanently deleted from /tmp/security-scan-<repo-name>.
No files from the scanned repository remain on disk.
```

**What gets deleted:**
- The full cloned repository directory and all its contents
- All source files, config files, and any files that contained secrets
- The `.git/` folder and all commit history
- Any intermediate files written during the scan

**What is NOT deleted:**
- The audit report itself (it stays in the conversation — the user needs it)
- Secret values found during scanning — these are never written to disk; they are
  reported as `[REDACTED]` in the conversation only

---

## Large Codebase Handling

If the codebase has **more than 200 files**, do the following:
1. Complete Phase 1 (discovery) and Phase 2 (triage) as normal
2. Announce: "Found N files. Scanning high-risk files first. Will complete full scan in batches."
3. Scan P1 + triage-hit files first, deliver an interim report
4. Ask: "Found N critical/high issues in the priority files. Continue scanning remaining N files?"
5. Continue in batches of 50 files, updating the report after each batch

---

## Supported Input Formats

```
# Local directory
scan ./my-project
audit /home/user/backend
check the codebase at /var/www/app

# GitHub URL (public repo)
scan https://github.com/owner/repo
audit https://github.com/owner/repo/tree/develop
security review https://github.com/owner/repo

# GitHub URL with branch
scan https://github.com/owner/repo on the staging branch

# npm / PyPI package (fetch source and scan)
audit the npm package express@4.17.1
```
