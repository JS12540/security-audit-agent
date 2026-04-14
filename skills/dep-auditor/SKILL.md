---
name: dep-auditor
description: Audit requirements.txt, package.json, pom.xml, and go.mod for packages with known CVEs
license: MIT
metadata:
  author: jay-shah
  version: "1.0.0"
  category: security
---

# Dependency Auditor

## When to Use
When the user shares a dependency file (requirements.txt, package.json, pom.xml,
go.mod, Gemfile) and wants to know if any packages have known vulnerabilities.

## Instructions

Parse the dependency file, identify the ecosystem, and check each package:

### Python — requirements.txt
Flag these well-known vulnerable ranges:
- `Pillow < 10.3.0` — multiple CVEs including RCE
- `PyYAML < 6.0.1` — arbitrary code exec via `yaml.load()` without SafeLoader
- `cryptography < 42.0.4` — multiple OpenSSL CVEs
- `Django < 4.2.11` — SQL injection, XSS depending on version
- `requests < 2.32.0` — certificate verification bypass
- `urllib3 < 2.2.2` — CVE-2024-37891 header injection
- `paramiko < 3.4.0` — authentication bypass
- `werkzeug < 3.0.3` — CVE-2024-34069 debugger PIN bypass
- `setuptools < 70.0.0` — CVE-2024-6345 RCE via package_index

### Node.js — package.json
Flag these:
- `lodash < 4.17.21` — prototype pollution (CVE-2021-23337)
- `axios < 1.6.0` — SSRF and credential exposure
- `node-fetch < 3.3.2` — multiple CVEs
- `semver < 7.5.4` — ReDoS (CVE-2022-25883)
- `tar < 6.2.1` — arbitrary file write
- `ip < 2.0.1` — SSRF bypass (CVE-2023-42282)
- `express < 4.19.2` — open redirect (CVE-2024-29041)
- `jwt-decode < 4.0.0` — prototype pollution

### General Flags for Any Ecosystem
- Unpinned versions (`*`, `latest`, `>=0.0.0`) → MEDIUM: no reproducible builds
- Packages with no version at all → MEDIUM: implicit latest is unpredictable
- Very old pinned versions (> 2 years without update) → LOW: review for abandoned status

## Output Format Per Vulnerable Package
```
Package: requests 2.27.0
Severity: HIGH
CVE: CVE-2023-32681
Vulnerable range: < 2.31.0
Fix: requests>=2.32.0
What it allows: Certificate hostname verification bypass in proxied requests
```

End with: "Run `pip audit`, `npm audit`, or `trivy fs .` for a full real-time CVE scan."
