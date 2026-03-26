# PII & Secret Scan Patterns

Used by the `open-source-prep` skill in Phase 4. These are regex patterns for use with `rg` (ripgrep) or the Grep tool.

**Default exclusions** (pass to `rg` via `--glob '!<pattern>'`):
```
!.git/
!node_modules/
!vendor/
!dist/
!build/
!bin/
!*.pb.go
!*.min.js
!*.lock
!go.sum
```

---

## Pattern Catalog

### 1. Email Addresses (Non-Example Domains)

Finds real email addresses, excluding common placeholder domains.

```regex
[a-zA-Z0-9._%+-]+@(?!example\.com|example\.org|example\.net|yourdomain\.com|domain\.com|test\.com|localhost)[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}
```

**rg command:**
```bash
rg --glob '!.git' --glob '!node_modules' --glob '!vendor' \
  '[a-zA-Z0-9._%+-]+@(?!example\.(com|org|net)|yourdomain\.com|domain\.com|test\.com|localhost)[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' \
  <repo-path> -l
```

**What to look for:** Developer email addresses, internal mailing lists, support contacts that reveal org information.

---

### 2. Internal / Private Hostnames

Finds references to internal network hostnames.

```regex
\b\w[\w.-]*\.(internal|corp|local|intranet|lan|priv|pvt|int)\b
```

**rg command:**
```bash
rg --glob '!.git' \
  '\b\w[\w.-]*\.(internal|corp|local|intranet|lan|priv|pvt|int)\b' \
  <repo-path> -n
```

Also check for patterns like `*.company-name.com` in non-public-facing configs.

---

### 3. Private / RFC-1918 IP Addresses

Finds hardcoded private IP ranges in strings/configs.

```regex
["'`\s](10\.\d{1,3}\.\d{1,3}\.\d{1,3}|172\.(1[6-9]|2[0-9]|3[01])\.\d{1,3}\.\d{1,3}|192\.168\.\d{1,3}\.\d{1,3})["'`\s]
```

**rg command:**
```bash
rg --glob '!.git' \
  '"(10\.\d+\.\d+\.\d+|172\.(1[6-9]|2[0-9]|3[01])\.\d+\.\d+|192\.168\.\d+\.\d+)"' \
  <repo-path> -n
```

---

### 4. Known API Key Patterns

High-confidence patterns for specific services.

```regex
# OpenAI / Anthropic
sk-[a-zA-Z0-9]{20,}

# GitHub personal access tokens (classic)
ghp_[a-zA-Z0-9]{36}

# GitHub fine-grained tokens
github_pat_[a-zA-Z0-9_]{82}

# GitHub Actions tokens
ghs_[a-zA-Z0-9]{36}

# AWS Access Key IDs
AKIA[A-Z0-9]{16}

# AWS Secret Access Keys (used in assignment context)
(?i)aws.secret.access.key\s*[=:]\s*["']?[A-Za-z0-9/+]{40}["']?

# Slack tokens
xoxb-[0-9]+-[0-9]+-[a-zA-Z0-9]+
xoxp-[0-9]+-[0-9]+-[0-9]+-[a-zA-Z0-9]+

# Stripe keys
sk_live_[a-zA-Z0-9]{24,}
pk_live_[a-zA-Z0-9]{24,}

# SendGrid
SG\.[a-zA-Z0-9._-]{22}\.[a-zA-Z0-9._-]{43}

# Twilio
AC[a-z0-9]{32}

# Google API keys
AIza[0-9A-Za-z_-]{35}

# Generic hex secrets (32-64 char hex strings in assignment context)
(?i)(secret|api_key|auth_token|access_token|signing_key)\s*[=:]\s*["']?[0-9a-fA-F]{32,64}["']?
```

**rg command (run separately for each pattern):**
```bash
rg --glob '!.git' --glob '!*.md' \
  '(sk-[a-zA-Z0-9]{20,}|ghp_[a-zA-Z0-9]{36}|AKIA[A-Z0-9]{16}|xoxb-[0-9]+-[0-9]+)' \
  <repo-path> -n
```

---

### 5. Generic Credentials in Assignments

Broad pattern for hardcoded passwords/secrets in code and config files.

```regex
(?i)(password|passwd|secret|api_key|apikey|api_secret|auth_token|access_token|bearer_token|private_key|signing_key)\s*[=:]\s*["'][^"'$\{\}<]{8,}["']
```

**rg command:**
```bash
rg --glob '!.git' --glob '!*.example' --glob '!*_test.go' \
  '(?i)(password|passwd|secret|api_key|apikey|auth_token|access_token)\s*[=:]\s*["'"'"'][^"'"'"'$\{\}<]{8,}["'"'"']' \
  <repo-path> -n
```

**False positive reduction:**
- Ignore matches in `*_test.go`, `*_test.js`, `*.test.ts` files (test fixtures are acceptable)
- Ignore matches where value is a placeholder: `your_secret_here`, `changeme`, `<SECRET>`, `REPLACE_ME`, `example`, `TODO`

---

### 6. PEM Private Keys

```regex
-----BEGIN (RSA |EC |DSA |OPENSSH |ENCRYPTED )?PRIVATE KEY-----
```

**rg command:**
```bash
rg --glob '!.git' \
  '-----BEGIN (RSA |EC |DSA |OPENSSH |ENCRYPTED )?PRIVATE KEY-----' \
  <repo-path> -l
```

Also check for `.pem`, `.key`, `.p12`, `.pfx` files tracked in git:
```bash
git -C <repo-path> ls-files | grep -E '\.(pem|key|p12|pfx|cer|crt)$'
```

---

### 7. Database Connection Strings with Credentials

```regex
(postgres|postgresql|mysql|mariadb|mongodb|redis|amqp|rabbitmq):\/\/[^:@\s]+:[^@\s]{3,}@[^\s"']+
```

**rg command:**
```bash
rg --glob '!.git' \
  '(postgres|postgresql|mysql|mongodb|redis|amqp):\/\/[^:@\s]+:[^@\s]{3,}@' \
  <repo-path> -n
```

**Note:** `.env.example` connection strings are fine if they use placeholder credentials (e.g., `user:password@`). Flag only if actual credentials appear to be real.

---

### 8. `.env` Files Tracked in Git

```bash
# Check if any .env file (not .env.example) is tracked
git -C <repo-path> ls-files | grep -E '^(\.env|.*/\.env)$'
```

Also verify `.gitignore` includes `.env`:
```bash
grep -E '^\s*\.env\s*$' <repo-path>/.gitignore
```

Record as `[CRITICAL]` if any `.env` file is tracked in git (not just `.env.example`).

---

### 9. Internal URLs and Endpoints

Patterns that reveal internal infrastructure.

```regex
https?:\/\/(?:[\w-]+\.)*(?:internal|corp|intranet|staging|dev|local|localhost|127\.0\.0\.1)[:\/?#]
```

Also grep for common internal patterns:
```bash
rg --glob '!.git' \
  'https?:\/\/[a-z0-9.-]*(internal|staging|dev\.internal|corp\.|\.local)[:\/?]' \
  <repo-path> -n
```

---

### 10. TODO/FIXME with Internal Context

Comments referencing internal systems, ticket numbers, or people by name.

```regex
(?i)(TODO|FIXME|HACK|XXX):?\s*(ask|contact|see|cf\.?|ref:?|jira|linear|notion|confluence|slack|@[a-zA-Z]+)\s+\S+
```

```bash
rg --glob '!.git' \
  '(?i)(TODO|FIXME|HACK).*(@[a-zA-Z]+|jira\.company|linear\.app|notion\.so|slack\.com|confluence)' \
  <repo-path> -n
```

---

## Git History Patterns

Run these against the full git history to check for secrets ever committed:

```bash
# Suspicious commit messages
git -C <repo-path> log --all --oneline \
  --grep="secret\|password\|credential\|api.key\|private.key\|token\|passphrase" \
  --regexp-ignore-case | head -20

# Files with sensitive extensions ever added
git -C <repo-path> log --all --diff-filter=A --name-only --pretty=format: \
  | grep -E '\.(env|pem|key|p12|pfx|cer|crt|keystore|jks)$' | head -20

# .env files ever committed
git -C <repo-path> log --all --diff-filter=A --name-only --pretty=format: \
  | grep -E '(^|/)\.env$' | head -10
```

**Recommended deep scan tools:**
- `trufflehog git file://<repo-path> --only-verified` — finds verified active secrets
- `gitleaks detect --source <repo-path>` — broad pattern matching across history
- `git-secrets --scan-history` — AWS-focused but extensible

---

## False Positive Guidance

| Pattern | Common False Positives | How to Exclude |
|---------|----------------------|----------------|
| Email addresses | Author fields in package.json, license headers | Ignore `package.json` maintainer fields, `LICENSE` files |
| Hex secrets | Git SHA references, color codes, UUIDs | Cross-check against variable names |
| Private IPs | Docker subnet ranges, test fixtures | Check if in `docker-compose`, `*_test.*` files |
| DB connection strings | `.env.example` with placeholder creds | Flag only non-placeholder values |
| API key patterns | Test/fake keys in unit tests | Check `*_test.*` files separately |
