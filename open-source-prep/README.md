<!-- Human documentation only. Not intended for LLM consumption. -->

# open-source-prep

A Claude Code skill that audits a repository for readiness to move from private to public. Runs a structured 7-phase evaluation and produces a prioritized findings report with specific remediation steps.

## Usage

```
/open-source-prep
/open-source-prep path/to/repo
```

## What It Checks

### Phase 1 — License
Detects the license file and identifies the SPDX type. If no license is present, interviews the user through a decision tree to recommend the right license (MIT, Apache-2.0, GPL-3.0, AGPL-3.0) based on their goals.

### Phase 2 — Community Health Files
Checks for presence and completeness of:
- `README.md` (with build, configure, run, and contribute sections)
- `CONTRIBUTING.md`
- `CODE_OF_CONDUCT.md`
- `SECURITY.md`
- `CHANGELOG.md`
- GitHub issue/PR templates
- `CODEOWNERS`

### Phase 3 — GitHub Repository Configuration
Evaluates all **24 OpenSSF GitHub Repository Best Practices** via the `gh` CLI:
- Branch protection (no force push, required reviews, deletion protection)
- Workflow token permissions (read-only)
- Vulnerability alerts and dependency review
- Webhook SSL and secrets
- Admin count
- Signed commits, linear history, conversation resolution

Also checks for Dependabot/Renovate config, CI/CD workflows, and hardcoded secrets in workflow YAML.

### Phase 4 — PII & Secrets Scan
Scans working tree with regex patterns for:
- Email addresses (non-placeholder domains)
- Internal/private hostnames (`*.internal`, `*.corp`, `*.local`)
- Private IP ranges
- Known API key formats (OpenAI `sk-`, GitHub `ghp_`, AWS `AKIA*`, Slack `xoxb-`, Stripe `sk_live_`, etc.)
- Generic credentials in assignments (`password = "..."`)
- PEM private keys
- Database connection strings with credentials
- Tracked `.env` files

Also audits git history for suspicious commit messages and sensitive file extensions ever committed. Recommends `trufflehog` or `gitleaks` for deep history scanning.

### Phase 5 — Dependency Review
Checks for dependency manifests and lock files. Flags missing lock files and potential license incompatibilities between dependencies and the repo's chosen license.

### Phase 6 — Code Security Review
Launches the `devsecops` subagent for an OWASP Top 10 review of the codebase. Findings are incorporated into the final report (no Linear issues created).

### Phase 7 — Report
Produces a prioritized report with a readiness score (0–100%) and findings bucketed as:
- **Critical** — blockers, must fix before going public
- **High** — strongly recommended before launch
- **Medium** — recommended, can be post-launch
- **Low** — nice-to-have

## Requirements

- `gh` CLI authenticated (`gh auth login`) — required for Phase 3 (GitHub config); other phases work without it
- `devsecops` subagent available in the Claude Code environment (standard in the base install)

## File Structure

```
open-source-prep/
  SKILL.md                        # Main skill — 7-phase orchestration workflow
  checks/
    openssf-checklist.md          # All 24 OpenSSF checks with gh CLI verification commands
    pii-patterns.md               # Regex patterns for PII/secrets scanning
  templates/
    report-template.md            # Structured report format with scoring rubric
```

## Additional Checks (Beyond the Basics)

Beyond the user-specified requirements, the skill also evaluates:

| Check | Severity |
|-------|----------|
| `CONTRIBUTING.md` presence | High |
| `CODE_OF_CONDUCT.md` presence | High |
| `SECURITY.md` / vulnerability disclosure policy | High |
| `CHANGELOG` presence | Medium |
| GitHub issue + PR templates | Medium |
| Dependabot / Renovate configuration | High |
| CI/CD pipeline present | High |
| `CODEOWNERS` | Medium |
| Git history secret audit | Critical |
| `.gitignore` completeness (covers `.env`) | High |
| Dependency license compatibility | High |
| Semantic versioning / release tags | Medium |
| DCO vs CLA strategy | Medium |
| TODO/FIXME referencing internal systems | Medium |

## References

- [OpenSSF GitHub Best Practices](https://best.openssf.org/SCM-BestPractices/github/)
- [Choosing a License](https://choosealicense.com/)
- [SPDX License List](https://spdx.org/licenses/)
- [Contributor Covenant](https://www.contributor-covenant.org/)
- [OSSF Scorecard](https://securityscorecards.dev/)
