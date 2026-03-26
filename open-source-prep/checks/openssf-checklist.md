# OpenSSF GitHub Repository Best Practices Checklist

Source: https://best.openssf.org/SCM-BestPractices/github/ (Repository section)

All 24 checks with verification method and remediation. Used by the `open-source-prep` skill in Phase 3.

---

## How to Verify

Most checks require GitHub API access via `gh`. Replace `<owner>/<repo>` with the repo slug.

```bash
# Get branch protection for default branch
gh api repos/<owner>/<repo>/branches/<default-branch>/protection

# Get repo-level settings
gh api repos/<owner>/<repo>

# Get workflow permissions
gh api repos/<owner>/<repo>/actions/permissions/workflow

# Get collaborators with admin role
gh api repos/<owner>/<repo>/collaborators?permission=admin --jq 'length'

# Get webhooks
gh api repos/<owner>/<repo>/hooks
```

---

## The 24 Checks

### 1. Repository Should Be Updated At Least Quarterly
**Severity:** Medium
**Check:** `gh api repos/<owner>/<repo> --jq '.pushed_at'` — must be within last 90 days
**Remediation:** Ensure the project is actively maintained. If abandoned, add an ARCHIVED notice to README.

---

### 2. Workflows Should Not Be Allowed To Approve Pull Requests
**Severity:** High
**Check:** `gh api repos/<owner>/<repo>/actions/permissions/workflow --jq '.can_approve_pull_request_reviews'` should be `false`
**Remediation:** Go to Settings → Actions → General → Workflow permissions → uncheck "Allow GitHub Actions to create and approve pull requests"

---

### 3. Default Branch Should Require Code Review
**Severity:** High
**Check:** `gh api repos/<owner>/<repo>/branches/<default>/protection --jq '.required_pull_request_reviews'` should not be null
**Remediation:** Settings → Branches → Branch protection rules → "Require a pull request before merging" → "Require approvals"

---

### 4. Default Branch Should Require Linear History
**Severity:** Medium
**Check:** `gh api repos/<owner>/<repo>/branches/<default>/protection --jq '.required_linear_history.enabled'` should be `true`
**Remediation:** Settings → Branches → Branch protection → "Require linear history"

---

### 5. Default Workflow Token Permission Should Be Set To Read Only
**Severity:** High
**Check:** `gh api repos/<owner>/<repo>/actions/permissions/workflow --jq '.default_workflow_permissions'` should be `"read"`
**Remediation:** Settings → Actions → General → Workflow permissions → "Read repository contents and packages permissions"

---

### 6. OSSF Scorecard Score Should Be Above 7
**Severity:** Medium
**Check:** Run `scorecard --repo github.com/<owner>/<repo>` (requires `scorecard` CLI) or check https://securityscorecards.dev
**Remediation:** Address findings from `scorecard` output. Common improvements: add CI, branch protection, dependency update tooling.

---

### 7. Default Branch Should Require Code Review By At Least Two Reviewers
**Severity:** Medium
**Check:** `gh api repos/<owner>/<repo>/branches/<default>/protection --jq '.required_pull_request_reviews.required_approving_review_count'` should be `>= 2`
**Remediation:** Settings → Branches → Branch protection → "Require approvals" → set to 2+

---

### 8. Default Branch Should Require All Checks To Pass Before Merge
**Severity:** Medium
**Check:** `gh api repos/<owner>/<repo>/branches/<default>/protection --jq '.required_status_checks'` should not be null, with context checks listed
**Remediation:** Settings → Branches → Branch protection → "Require status checks to pass before merging" → add CI check names

---

### 9. Default Branch Should Require Branches To Be Up To Date Before Merge
**Severity:** Medium
**Check:** `gh api repos/<owner>/<repo>/branches/<default>/protection --jq '.required_status_checks.strict'` should be `true`
**Remediation:** Settings → Branches → Branch protection → "Require branches to be up to date before merging"

---

### 10. Default Branch Should Not Allow Force Pushes
**Severity:** High
**Check:** `gh api repos/<owner>/<repo>/branches/<default>/protection --jq '.allow_force_pushes.enabled'` should be `false`
**Remediation:** Settings → Branches → Branch protection → ensure "Allow force pushes" is unchecked

---

### 11. Default Branch Should Be Protected
**Severity:** High
**Check:** `gh api repos/<owner>/<repo>/branches/<default>/protection` returns a protection object (not 404)
**Remediation:** Settings → Branches → Add branch protection rule for `main` (or default branch)

---

### 12. Default Branch Deletion Protection Should Be Enabled
**Severity:** High
**Check:** `gh api repos/<owner>/<repo>/branches/<default>/protection --jq '.allow_deletions.enabled'` should be `false`
**Remediation:** Settings → Branches → Branch protection → ensure "Allow deletions" is unchecked

---

### 13. GitHub Advanced Security – Dependency Review Should Be Enabled For A Repository
**Severity:** High
**Check:** `gh api repos/<owner>/<repo> --jq '.security_and_analysis.dependency_review_enforcement.status'` should be `"enabled"`
**Note:** Requires GitHub Advanced Security (free for public repos; paid for private)
**Remediation:** Settings → Code security and analysis → Dependency review → Enable. Also add the `dependency-review-action` to CI workflow.

---

### 14. Vulnerability Alerts Should Be Enabled
**Severity:** High
**Check:** `gh api repos/<owner>/<repo> --jq '.vulnerability_alerts_enabled'` should be `true` (or use `gh repo view --json vulnerabilityAlertsEnabled`)
**Remediation:** Settings → Code security and analysis → Dependabot alerts → Enable

---

### 15. Forking Should Not Be Allowed for This Repository
**Severity:** Low
**Check:** `gh api repos/<owner>/<repo> --jq '.allow_forking'` — for public OSS repos forking is expected; this check is more relevant for private org repos
**Note:** For open-source repos going public, forking is typically desired. Mark as N/A or Low for public repos.
**Remediation (if needed):** Settings → General → "Allow forking" → uncheck

---

### 16. Default Branch Should Require All Conversations To Be Resolved Before Merge
**Severity:** Medium
**Check:** `gh api repos/<owner>/<repo>/branches/<default>/protection --jq '.required_conversation_resolution.enabled'` should be `true`
**Remediation:** Settings → Branches → Branch protection → "Require conversation resolution before merging"

---

### 17. Webhooks Should Be Configured With A Secret
**Severity:** High
**Check:** `gh api repos/<owner>/<repo>/hooks --jq '.[] | {url: .config.url, has_secret: (.config.secret != null and .config.secret != "")}'` — all hooks should have `has_secret: true`
**Remediation:** Settings → Webhooks → edit each webhook → add/update the Secret field

---

### 18. Default Branch Should Require All Commits To Be Signed
**Severity:** Medium
**Check:** `gh api repos/<owner>/<repo>/branches/<default>/protection --jq '.required_signatures.enabled'` should be `true`
**Remediation:** Settings → Branches → Branch protection → "Require signed commits". Ensure contributors have GPG/SSH signing configured.

---

### 19. Default Branch Should Require New Code Changes After Approval To Be Re-Approved
**Severity:** Medium
**Check:** `gh api repos/<owner>/<repo>/branches/<default>/protection --jq '.required_pull_request_reviews.dismiss_stale_reviews'` should be `true`
**Remediation:** Settings → Branches → Branch protection → "Dismiss stale pull request approvals when new commits are pushed"

---

### 20. Default Branch Should Restrict Who Can Push To It
**Severity:** Medium
**Check:** `gh api repos/<owner>/<repo>/branches/<default>/protection --jq '.restrictions'` should not be null
**Remediation:** Settings → Branches → Branch protection → "Restrict who can push to matching branches" → add authorized users/teams

---

### 21. Repository Should Have Fewer Than Three Admins
**Severity:** Medium
**Check:** `gh api repos/<owner>/<repo>/collaborators?permission=admin --jq 'length'` should be `< 3`
**Remediation:** Settings → Collaborators and teams → review admin roles; downgrade to "Write" where admin is not necessary

---

### 22. Default Branch Should Limit Code Review to Code-Owners
**Severity:** Low
**Check:** `gh api repos/<owner>/<repo>/branches/<default>/protection --jq '.required_pull_request_reviews.require_code_owner_reviews'` should be `true`
**Requires:** `CODEOWNERS` file must exist
**Remediation:** Add a `CODEOWNERS` file, then enable "Require review from Code Owners" in branch protection

---

### 23. Default Branch Should Restrict Who Can Dismiss Reviews
**Severity:** Medium
**Check:** `gh api repos/<owner>/<repo>/branches/<default>/protection --jq '.required_pull_request_reviews.dismissal_restrictions'` should not be null
**Remediation:** Settings → Branches → Branch protection → under "Require approvals" → "Restrict who can dismiss pull request reviews" → add authorized users/teams

---

### 24. Webhooks Should Be Configured To Use SSL
**Severity:** High
**Check:** `gh api repos/<owner>/<repo>/hooks --jq '.[] | {url: .config.url, ssl_disabled: (.config.insecure_ssl == "1")}'` — all hooks should have `ssl_disabled: false`
**Remediation:** Settings → Webhooks → edit each webhook → check "Enable SSL verification"

---

## Quick Summary Table

| # | Check | Severity | API Field |
|---|-------|----------|-----------|
| 1 | Updated quarterly | Medium | `pushed_at` |
| 2 | Workflows can't approve PRs | High | `actions/permissions/workflow.can_approve_pull_request_reviews` |
| 3 | Requires code review | High | `protection.required_pull_request_reviews` |
| 4 | Requires linear history | Medium | `protection.required_linear_history.enabled` |
| 5 | Workflow token read-only | High | `actions/permissions/workflow.default_workflow_permissions` |
| 6 | OSSF Scorecard > 7 | Medium | External tool |
| 7 | Requires 2+ reviewers | Medium | `required_approving_review_count` |
| 8 | All checks must pass | Medium | `protection.required_status_checks` |
| 9 | Branch up to date | Medium | `protection.required_status_checks.strict` |
| 10 | No force pushes | High | `protection.allow_force_pushes.enabled == false` |
| 11 | Branch protected | High | Protection object exists |
| 12 | No branch deletion | High | `protection.allow_deletions.enabled == false` |
| 13 | Dependency review enabled | High | `security_and_analysis.dependency_review_enforcement` |
| 14 | Vulnerability alerts on | High | `vulnerabilityAlertsEnabled` |
| 15 | Forking restricted | Low | `allow_forking` (N/A for OSS) |
| 16 | Conversations resolved | Medium | `protection.required_conversation_resolution.enabled` |
| 17 | Webhooks have secrets | High | `hooks[].config.secret` |
| 18 | Signed commits required | Medium | `protection.required_signatures.enabled` |
| 19 | Stale reviews dismissed | Medium | `protection.required_pull_request_reviews.dismiss_stale_reviews` |
| 20 | Push restricted | Medium | `protection.restrictions` |
| 21 | Fewer than 3 admins | Medium | collaborators count with admin role |
| 22 | CODEOWNERS review required | Low | `require_code_owner_reviews` |
| 23 | Review dismissal restricted | Medium | `protection.required_pull_request_reviews.dismissal_restrictions` |
| 24 | Webhooks use SSL | High | `hooks[].config.insecure_ssl != "1"` |
