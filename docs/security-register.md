# Security Register

## Gate Policy

| Job | Blocks on | Rationale |
|-----|-----------|-----------|
| Secret scan (gitleaks) | Any finding | A leaked secret has no safe threshold. |
| Image scan (Trivy) | CRITICAL with a published fix | Fixable criticals are actionable; unfixed ones would block with no remedy. |
| IaC scan (Checkov) | Any failed check | Misconfiguration is cheapest to fix before `apply`; accepted risks are suppressed inline with a justification. |
| Code scan (Bandit) | MEDIUM and above | Low findings (e.g. B105) are frequent false positives and stay as warnings, so the gate is not disabled. Real risks (B602 shell injection, B301 pickle) are MEDIUM/HIGH and block. |

## Findings

| ID | Finding | Source | Severity | Decision | Owner | Revisit | Justification |
|----|---------|--------|----------|----------|-------|---------|---------------|
| R-01 | Hardcoded `SECRET_KEY`, `campus_notifier/settings.py:27` (B105) | Bandit | Low | Remediated | El-Gibbor | 2026-12-10 | Now read from `DJANGO_SECRET_KEY`. Removed from the file, **not rotated**: the old key remains in git history and must be treated as compromised. Rotate before any production deployment. |
| R-02 | Django 5.2.7: 1 CRITICAL (CVE-2025-64459), 7 HIGH | Trivy | Critical | Remediated | El-Gibbor | — | Upgraded to Django 5.2.19; blocked by the image gate before the fix. |
| R-03 | Debian 13.7 base: 49 HIGH (1 fixable: `zlib1g`) | Trivy | High | Accepted risk | El-Gibbor | 2026-12-10 | Below the CRITICAL gate. 48 have no upstream fix; `zlib1g` is resolved by rebuilding on a refreshed `python:3.12-slim`. Re-scan monthly; evaluate a distroless base. |
| R-04 | ECR repository not KMS-encrypted (CKV_AWS_136), `terraform/main.tf` | Checkov | Medium | Remediated | El-Gibbor | — | Added `encryption_configuration { encryption_type = "KMS" }`. Re-scan: 4 passed, 0 failed. |

## Discussion

**Why run gitleaks in CI when a commit-time hook exists?** The hook is local and optional: it can be skipped (`--no-verify`) or never installed, and it sees only the staged diff. CI is enforced on every PR and scans the full history.

**Why did Bandit flag a key that gitleaks passed?** They use different detection methods. gitleaks matches credential patterns and entropy, and allowlists the `django-insecure-` placeholder. Bandit flags any string literal assigned to a password-like name (`SECRET_KEY`), whatever the value.
