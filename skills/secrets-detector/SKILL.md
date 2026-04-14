---
name: secrets-detector
description: Detect hardcoded API keys, passwords, tokens, private keys, and connection strings in source code
license: MIT
metadata:
  author: jay-shah
  version: "1.0.0"
  category: security
---

# Secrets Detector

## When to Use
When reviewing code for hardcoded credentials, API keys, passwords, or any
value that should be in an environment variable instead.

## Instructions

Scan for these patterns and report every match:

### Known Secret Formats
- **AWS**: `AKIA[0-9A-Z]{16}` access key IDs; long strings near `aws_secret`
- **GitHub**: tokens starting with `ghp_`, `ghs_`, `github_pat_`
- **Stripe**: `sk_live_`, `pk_live_`, `rk_live_`
- **Slack**: `xoxb-`, `xoxp-`, `xoxa-`
- **Google**: `AIza[0-9A-Za-z\-_]{35}`
- **Twilio**: `SK[0-9a-fA-F]{32}`
- **JWT**: three base64url segments joined by `.` assigned to a variable
- **Private keys**: `-----BEGIN RSA PRIVATE KEY-----`, `-----BEGIN EC PRIVATE KEY-----`
- **Database DSNs**: `postgresql://user:password@host`, `mongodb://user:pass@host`

### Variable Name Patterns
Flag any non-empty string literal assigned to variables named:
`password`, `passwd`, `pwd`, `secret`, `api_key`, `apikey`, `access_token`,
`auth_token`, `private_key`, `client_secret`, `bearer_token`, `db_pass`

### Config File Patterns
- `.env` files with real values (not placeholders like `your_key_here`)
- `config.yaml` / `settings.py` with credential fields set to real values

## Reporting Rules
- **Never output the actual secret value** — always show `[REDACTED]`
- Show: file, line number, variable name, secret type
- Severity: CRITICAL for production credentials, HIGH for dev/test credentials
- Always recommend: "Move to environment variable or secrets manager"

## Output Format Per Finding
```
Severity: CRITICAL
Type: AWS_ACCESS_KEY
Location: app/config.py:14
Variable: aws_key = "[REDACTED]"
Fix: aws_key = os.getenv("AWS_ACCESS_KEY_ID")
```
