# Security Register

| ID | Finding | Source | Severity | Status | Justification / Remedy |
|----|---------|--------|----------|--------|------------------------|
| R-01 | Hardcoded `SECRET_KEY` in `campus_notifier/settings.py:27` (B105) | Bandit, PR #1 | Low | Mitigated | Moved to `DJANGO_SECRET_KEY` in `.env`. Removed from the file, **not rotated**: the old key remains in git history and must be treated as compromised. |
| R-02 | Django 5.2.7: 1 CRITICAL (CVE-2025-64459, SQL injection), 7 HIGH | Trivy, image `campus-notifier:dev` | Critical | Open | Fix available. Pin `Django>=5.2.17` in `requirements.txt` and rebuild. |
| R-03 | Debian 13.7 base (`python:3.12-slim`): 49 HIGH, 0 CRITICAL | Trivy, image `campus-notifier:dev` | High | Open | 1 fixable (`zlib1g`): rebuild on a refreshed base. The rest (util-linux, sqlite, ncurses, systemd) have no upstream fix: accept and re-scan, or move to a smaller base image. |
| R-04 | ECR repository not encrypted with KMS (CKV_AWS_136), `terraform/main.tf` | Checkov, PR #1 | Medium | Fixed | Added `encryption_configuration { encryption_type = "KMS" }` (AWS-managed `aws/ecr` key). Re-scan: 4 passed, 0 failed. |

## Discussion

**Why run gitleaks in CI when a commit-time hook exists?** The hook is local and optional: it can be skipped (`--no-verify`) or never installed, and it sees only the staged diff. CI is enforced on every PR and scans the full history.

**Why did Bandit flag a key that gitleaks passed?** They use different detection methods. gitleaks matches credential patterns and entropy, and allowlists the `django-insecure-` placeholder. Bandit flags any string literal assigned to a password-like name (`SECRET_KEY`), whatever the value.
