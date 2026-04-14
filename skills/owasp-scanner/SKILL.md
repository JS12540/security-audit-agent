---
name: owasp-scanner
description: Scan code for all OWASP Top 10 (2021) vulnerability categories with severity, location, and fix
license: MIT
metadata:
  author: jay-shah
  version: "1.0.0"
  category: security
---

# OWASP Scanner

## When to Use
When the user pastes source code, shares a single file, or asks "is this code safe?"
For full directory or GitHub URL scans, use the `codebase-scanner` skill first —
it calls this skill per-file during the scan.

## Instructions

Scan systematically through all 10 OWASP 2021 categories. For every finding:

```
Severity: CRITICAL | HIGH | MEDIUM | LOW
Category: A0X:2021 — Name
Location: filename.py:line_number (or describe the code block)
Finding: One sentence on what the vulnerability is
Evidence: The exact vulnerable code snippet
Fix: The corrected code or specific action to take
```

### A01 — Broken Access Control
- User-supplied IDs used directly in DB queries without ownership check (IDOR)
- Missing authorization on routes/endpoints
- Path traversal patterns: `../`, `%2e%2e`, `....//`
- Privileged operations accessible without role check

### A02 — Cryptographic Failures
- Passwords hashed with MD5 or SHA1 (not bcrypt/argon2/scrypt)
- Hardcoded encryption keys or IVs in source
- Weak RSA key sizes (< 2048 bit)
- HTTP used for sensitive data transmission

### A03 — Injection
- String concatenation into SQL: `"SELECT * FROM users WHERE id = " + user_id`
- `os.system()`, `subprocess(shell=True)`, `exec()`, `eval()` with user input
- f-strings or format strings building queries/commands from user data
- Template injection: user content rendered in Jinja2/Mako/Twig without escaping

### A04 — Insecure Design
- No rate limiting on login or password-reset endpoints
- Security-critical logic only enforced client-side
- No account lockout after failed attempts

### A05 — Security Misconfiguration
- `DEBUG=True` or `NODE_ENV=development` in production configs
- Overly permissive CORS: `Access-Control-Allow-Origin: *` on authenticated endpoints
- Default credentials present in config files
- Stack traces or internal paths exposed to users

### A06 — Vulnerable and Outdated Components
- Flag dependency versions (see dep-auditor skill for deep scan)
- Deprecated libraries with no maintenance activity

### A07 — Identification and Authentication Failures
- Session tokens in URLs (`?session_id=...`)
- Missing `HttpOnly` / `Secure` flags on session cookies
- Passwords stored in plain text or reversibly encoded
- No MFA for privileged operations

### A08 — Software and Data Integrity Failures
- `pickle.loads()`, `yaml.load()` (without SafeLoader), `marshal.loads()` on untrusted data
- Deserialization of user-supplied content without type validation

### A09 — Security Logging and Monitoring Failures
- No logging of authentication events
- Sensitive data (passwords, tokens) appearing in log statements
- No alerting on repeated failed access attempts

### A10 — Server-Side Request Forgery (SSRF)
- User-supplied URLs passed to `requests.get()`, `urllib.urlopen()`, `fetch()` without allowlist
- Internal metadata endpoints reachable: `169.254.169.254`

## Output Format
End with a summary table:
| Severity | Count |
|----------|-------|
| CRITICAL | N |
| HIGH | N |
| MEDIUM | N |
| LOW | N |
| **Total** | **N** |
