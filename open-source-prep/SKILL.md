---
name: "open-source-prep"
description: "Evaluate a repository for readiness to move from private to public. Scans license, community health files, GitHub repository configuration (OpenSSF best practices), PII/secrets in code and git history, dependencies, and code security. Produces a prioritized findings report with specific remediation steps. Use when preparing a repo to go open source, before making a private repo public, or auditing open-source readiness."
argument-hint: "[path-to-repo (default: current directory)]"
---

# Open Source Prep

Perform a comprehensive audit of a repository against open-source readiness criteria. Run all phases, then produce a single prioritized report. Do not stop early — complete all phases before reporting.

## Inputs

- `repo`: path to the repository root (default: current working directory `.`)
- `gh` authentication is required for Phase 3 (GitHub config checks); if unavailable, skip Phase 3 and note it

## Bash Execution Safety Rules

**Always follow these rules when generating bash commands to avoid cascade failures:**

1. **Never use `ls` to check for file existence.** `ls file1 file2` exits non-zero if ANY argument is missing, which cancels all sibling parallel tool calls. Use `[ -f file ]` tests or a loop instead.
2. **Add `|| true` to any command that may produce a non-zero exit on a valid "not found" result** (e.g. `grep`, `find`, `git log --grep`).
3. **Consolidate related checks into a single bash script** rather than multiple parallel calls — a single non-zero exit in a parallel batch cancels all siblings.
4. **`gh repo view --json` only accepts a specific set of fields.** If a field is rejected, the entire command fails. Use `gh api repos/<owner>/<repo>` for settings not in the known-safe field list below.
5. **rg patterns starting with `-` or `--` must use the `-e` flag:** `rg -e '-----BEGIN...'`

## Setup

1. Determine the repo root:
   - If an argument was provided, use it as the path. Otherwise use `.`
   - Confirm it is a git repo: `git -C <repo> rev-parse --show-toplevel`
2. Confirm `gh` authentication: `gh auth status`
   - If unauthenticated, note "GitHub config checks will be skipped" and proceed without Phase 3
3. Get repo context — run as a single script to avoid cascade failures:
```bash
echo "=== REPO CONTEXT ==="
git -C <repo> remote get-url origin
git -C <repo> symbolic-ref refs/remotes/origin/HEAD --short 2>/dev/null || echo "main"
for f in go.mod package.json requirements.txt Gemfile Cargo.toml; do
  [ -f "<repo>/$f" ] && echo "LANG_FILE: $f"
done
```
   - GitHub repo slug: parse `owner/repo` from the remote URL

---

## Phase 1: License Check

Look for a license file at the repo root:
```
LICENSE
LICENSE.md
LICENSE.txt
LICENSE.rst
COPYING
COPYING.md
```

**If found:**
- Read the first 5 lines to identify the license type
- Determine the SPDX identifier (MIT, Apache-2.0, GPL-3.0, AGPL-3.0, etc.)
- Record: `[PASS] License: <SPDX-ID> found at <filename>`

**If not found — interview the user:**

Ask the following questions in sequence to recommend a license:

1. "Is this a **library/framework** others will import, or an **application** end users run?"
   - Library → lean permissive (MIT, Apache-2.0) so it's broadly adoptable
   - Application → copyleft is viable (GPL, AGPL)

2. "Do you want anyone (including competitors) to be able to use this in proprietary products without releasing their changes?"
   - Yes → **MIT** or **Apache-2.0** (Apache adds patent grant)
   - No → **GPL-3.0** or **AGPL-3.0**

3. (If copyleft) "Does your software run as a network service that users connect to remotely?"
   - Yes → **AGPL-3.0** (closes the "SaaS loophole" — forces source release for network use)
   - No → **GPL-3.0**

4. "Do you want to grant an explicit patent license to users?" (usually yes for Apache-2.0)
   - Yes → **Apache-2.0** (permissive) or **GPL-3.0** (copyleft)

Summarize recommendation and ask user to confirm. Record: `[CRITICAL] No LICENSE file found. Recommended: <license>. Must be added before going public.`

---

## Phase 2: Community Health Files

Run as a **single consolidated script** to avoid parallel cascade failures:

```bash
echo "=== COMMUNITY HEALTH FILES ==="
for f in README.md README.rst README CONTRIBUTING.md CODE_OF_CONDUCT.md SECURITY.md CHANGELOG.md RELEASES.md CODEOWNERS; do
  [ -f "$f" ] && echo "FOUND: $f" || echo "MISSING: $f"
done
for f in .github/ISSUE_TEMPLATE .github/pull_request_template.md .github/CODEOWNERS; do
  [ -e "$f" ] && echo "FOUND: $f" || echo "MISSING: $f"
done
```

| File | Severity if missing | Purpose |
|------|--------------------|---------|
| `README.md` (or `README.rst`, `README`) | Critical | Project overview, build, configure, run, contribute |
| `CONTRIBUTING.md` | High | How to submit PRs, coding standards, dev setup |
| `CODE_OF_CONDUCT.md` | High | Community behavioral standards |
| `SECURITY.md` | High | Vulnerability disclosure policy and contact |
| `CHANGELOG.md` or `RELEASES.md` | Medium | Project change history |
| `.github/ISSUE_TEMPLATE/` (directory) | Medium | Guides bug reports and feature requests |
| `.github/pull_request_template.md` | Medium | PR hygiene and checklist |
| `CODEOWNERS` or `.github/CODEOWNERS` | Medium | Code review ownership |

**README completeness check** (if README exists):
Scan README content for evidence of each of these sections (keyword matching):
- Build/install: keywords like `install`, `build`, `make`, `npm install`, `go build`, `pip install`
- Configuration: keywords like `config`, `environment`, `.env`, `variables`, `setup`
- Running/operating: keywords like `run`, `start`, `usage`, `docker`, `deploy`
- Contributing: keywords like `contribut`, `pull request`, `PR`, `fork`
- License: keywords like `license`, `licensed under`, `SPDX`

Record findings for each missing section as `[HIGH] README missing <section> section`.

---

## Phase 3: GitHub Repository Configuration

Requires `gh` authentication. Skip if unauthenticated.

### 3a. Fetch repo settings

Run all GitHub API calls as a **single sequential script** to avoid parallel cascade failures. Note the known-safe field list for `gh repo view --json` — unknown fields cause a non-zero exit and kill the entire command.

```bash
echo "=== REPO SETTINGS ==="
# Known-safe fields only — do NOT add vulnerabilityAlertsEnabled or autoMergeAllowed
gh repo view --json name,description,isPrivate,defaultBranchRef,hasIssuesEnabled,deleteBranchOnMerge,squashMergeAllowed,mergeCommitAllowed,rebaseMergeAllowed,visibility,pushedAt,licenseInfo,isSecurityPolicyEnabled,primaryLanguage

echo "=== FORKING/SIGNOFF ==="
gh api repos/<owner>/<repo> --jq '{allow_forking: .allow_forking, web_commit_signoff_required: .web_commit_signoff_required, visibility: .visibility}'

echo "=== VULNERABILITY ALERTS ==="
# Returns HTTP 204 (exit 0) if enabled, 404 (exit non-zero) if disabled
gh api repos/<owner>/<repo>/vulnerability-alerts 2>&1; echo "EXIT:$?"

echo "=== BRANCH PROTECTION ==="
gh api repos/<owner>/<repo>/branches/<default-branch>/protection 2>&1 || echo "NO_PROTECTION"

echo "=== WORKFLOW PERMISSIONS ==="
gh api repos/<owner>/<repo>/actions/permissions/workflow 2>&1 || echo "CANNOT_FETCH_WORKFLOW_PERMS"

echo "=== ADMIN COUNT ==="
gh api repos/<owner>/<repo>/collaborators 2>/dev/null | python3 -c "
import sys,json
try:
  data=json.load(sys.stdin)
  admins=[u['login'] for u in data if u.get('permissions',{}).get('admin')]
  print(f'Admin count: {len(admins)}, logins: {admins}')
except: print('parse error')
" || echo "Could not fetch collaborators"

echo "=== WEBHOOKS ==="
gh api repos/<owner>/<repo>/hooks 2>&1 || echo "NO_WEBHOOKS"
```

**Interpreting vulnerability alerts:** EXIT:0 = alerts enabled (PASS). EXIT non-zero = alerts disabled (HIGH finding).

### 3b. Evaluate against OpenSSF checklist

Refer to `checks/openssf-checklist.md` for the full 24-check reference. Evaluate each check from the API responses above and record PASS/FAIL with the check name.

Key checks to evaluate from the data above:
- **Branch protection exists**: `protection != NO_PROTECTION` → `[HIGH]`
- **No force pushes**: `protection.allow_force_pushes.enabled == false` → `[HIGH]`
- **Requires PR reviews**: `protection.required_pull_request_reviews != null` → `[HIGH]`
- **Requires 2+ reviewers**: `required_approving_review_count >= 2` → `[MEDIUM]`
- **Branch deletion protection**: `protection.allow_deletions.enabled == false` → `[HIGH]`
- **Status checks required**: `protection.required_status_checks != null` → `[MEDIUM]`
- **Branches up to date before merge**: `protection.required_status_checks.strict == true` → `[MEDIUM]`
- **Signed commits required**: `protection.required_signatures.enabled == true` → `[MEDIUM]`
- **Vulnerability alerts enabled**: `vulnerabilityAlertsEnabled == true` → `[HIGH]`
- **Fewer than 3 admins**: `admin_count < 3` → `[MEDIUM]`
- **Default workflow permissions**: check `gh api /repos/<owner>/<repo>/actions/permissions/workflow` → `[HIGH]`
- **Webhooks use SSL**: all hooks have `ssl != "1"` → `[HIGH]`
- **Webhooks have secrets**: all hooks have `has_secret == true` → `[HIGH]`
- **Forking not allowed** (for private-origin repos): `allow_forking == false` → `[LOW]`

### 3c. Additional configuration checks

Use `[ -f ]` / `[ -d ]` tests — never `ls` — to avoid non-zero exits on missing files:

```bash
echo "=== ADDITIONAL CONFIG ==="

# Dependabot / Renovate
{ [ -f .github/dependabot.yml ] || [ -f .github/dependabot.yaml ]; } \
  && echo "FOUND: dependabot config" || echo "MISSING: dependabot config"
{ [ -f renovate.json ] || [ -f .renovaterc ] || [ -f renovate.json5 ] || [ -f .github/renovate.json ]; } \
  && echo "FOUND: renovate config" || echo "MISSING: renovate config"

# GitHub Actions
[ -d .github/workflows ] \
  && echo "FOUND: .github/workflows" && ls .github/workflows/ \
  || echo "MISSING: .github/workflows"

# Explicit permissions in workflow files
grep -r "^permissions:" .github/workflows/ 2>/dev/null || echo "No top-level permissions block in workflows"
```

Record:
- `[HIGH] No Dependabot or Renovate configuration found` — if neither exists
- `[HIGH] No CI/CD pipeline found in .github/workflows/` — if no workflow files
- `[MEDIUM] No CODEOWNERS file` — if missing

---

## Phase 4: PII & Secrets Scan

Use the patterns from `checks/pii-patterns.md`. Scan the working tree (exclude `.git/`, common build dirs: `node_modules/`, `vendor/`, `dist/`, `build/`, `bin/`, `*.pb.go`).

Run each pattern category and record any matches with file path and line number. Truncate output to first 10 matches per category.

**Pattern categories to scan:**
1. Email addresses (non-example domains): `[a-zA-Z0-9._%+-]+@(?!example\.com|example\.org|yourdomain\.com)[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}`
2. Internal/private hostnames: `\b\w+\.(internal|corp|local|intranet|lan)\b`
3. Private IP ranges in strings: `(192\.168\.|10\.\d+\.|172\.(1[6-9]|2[0-9]|3[01])\.)`
4. API key patterns: `(sk-[a-zA-Z0-9]{20,}|ghp_[a-zA-Z0-9]{36}|AKIA[A-Z0-9]{16}|xoxb-[0-9]+-[a-zA-Z0-9]+)`
5. Generic secrets in assignments: `(?i)(password|secret|api_key|apikey|token|auth_token|access_token)\s*[=:]\s*["'][^"']{8,}["']`
6. PEM private keys — **must use `-e` flag** (leading dashes are misread as rg flags without it): `rg -e '-----BEGIN (RSA |EC |DSA |OPENSSH )?PRIVATE KEY-----'`
7. Database connection strings with credentials: `(postgres|mysql|mongodb|redis):\/\/[^:]+:[^@]+@`
8. `.env` files with non-placeholder values: check if `.env` (not `.env.example`) is tracked: `git ls-files | grep '\.env$'`

**Git history audit:**
```bash
# Count commits
git log --all --oneline | wc -l

# Look for suspicious commit messages
git log --all --oneline --grep="secret\|password\|credential\|token\|api.key\|private.key" -i | head -20

# Check for .env files ever committed
git log --all --diff-filter=A --name-only --pretty=format: | grep -E '^\.env$' | head -5

# Check for private key files ever committed
git log --all --diff-filter=A --name-only --pretty=format: | grep -E '\.(pem|key|p12|pfx)$' | head -5
```

If the history has more than 100 commits or any suspicious history findings:
Record: `[CRITICAL] Git history audit recommended — run trufflehog or gitleaks before going public. History is permanent once the repo is public.`

Provide command: `trufflehog git file://<repo-path> --only-verified` or `gitleaks detect --source <repo-path>`

---

## Phase 5: Dependency Review

Detect dependency manifest files using `[ -f ]` tests — not `ls`:

```bash
echo "=== DEPENDENCY FILES ==="
for f in go.mod go.sum package.json package-lock.json yarn.lock requirements.txt Pipfile Gemfile Gemfile.lock Cargo.toml Cargo.lock pom.xml build.gradle; do
  [ -f "$f" ] && echo "FOUND: $f" || echo "MISSING: $f"
done
```

For each detected ecosystem:
- **Go**: `go.mod` present → check if `go.sum` is committed (should be)
- **Node.js**: `package.json` → check for `package-lock.json` or `yarn.lock`
- **Python**: `requirements.txt` or `Pipfile` → check if pinned versions
- **Ruby**: `Gemfile` → check for `Gemfile.lock`

Check for dependency license conflicts (if license was identified in Phase 1):
- If repo uses MIT/Apache-2.0: flag any GPL/AGPL dependencies as `[HIGH] Potential license incompatibility`
- Check `go.mod` for known copyleft packages, `package.json` for GPL-licensed npm packages (use `license-checker` or manual review note)

Record:
- `[HIGH] No dependency lock file committed` — if manifest exists but no lock file
- `[HIGH] Dependency license compatibility not verified` — always flag for manual review with tool suggestions

---

## Phase 6: Security Code Review

Launch the `devsecops` subagent to perform a security review of the codebase. Use this prompt:

> Perform a security review of the repository at `<repo-path>`. Focus on:
> - OWASP Top 10 vulnerabilities (injection, broken auth, XSS, IDOR, misconfig, etc.)
> - Hardcoded credentials or secrets in source code
> - Insecure cryptographic patterns (weak algorithms, hardcoded keys/IVs)
> - Exposed sensitive endpoints or admin routes without auth
> - Insecure dependencies (known CVEs)
> - Input validation gaps
>
> **IMPORTANT: Do NOT create any Linear issues.** Return findings as a structured list with severity (Critical/High/Medium/Low), description, file:line if applicable, and recommended fix. This output will be incorporated into a larger open-source readiness report.

Incorporate the subagent's findings into the final report under the appropriate severity buckets.

---

## Phase 7: Generate Report

Produce the final report using the structure from `templates/report-template.md`.

**Scoring:**
- Count total checks performed
- Count PASS vs findings
- Score = (PASS / total) × 100, rounded to nearest integer
- Readiness level:
  - 90-100%: Ready (with minor cleanup)
  - 70-89%: Nearly Ready (address High findings first)
  - 50-69%: Needs Work (address Critical and High before going public)
  - <50%: Not Ready (significant gaps)

**Severity definitions:**
- **Critical**: Blockers — MUST fix before going public (active secret in code, no license, history contains credentials)
- **High**: Strongly recommended before going public (missing CONTRIBUTING.md, no branch protection, no Dependabot)
- **Medium**: Recommended, can be addressed post-launch (issue templates, CODEOWNERS, signed commits)
- **Low**: Nice-to-have improvements

**Report structure:**
1. Header: repo name, audit date, readiness score and level
2. Executive summary table: each category with pass/fail/partial
3. Critical findings (if any) — each with: finding, file/location, remediation command or action
4. High findings
5. Medium findings
6. Low findings
7. Already passing (brief list)
8. Recommended next steps — ordered checklist, highest severity first

After the report, ask: "Would you like me to create a GitHub issue or checklist to track these remediations?"
