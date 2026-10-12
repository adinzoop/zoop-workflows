# Zoop Workflows — How It Works

---

## Big Picture

```
Developer opens a PR
        │
        ▼
  GitHub detects it
        │
        ├──────────────────────────┬─────────────────────────────┐
        ▼                          ▼                             ▼
zoop-security-scan.yml    zoop-cleanup-scan.yml        (runs on schedule)
                                                      zoop-org-cleanup.yml
        │                          │                             │
        ▼                          ▼                             ▼
  GitHub spins up a         GitHub spins up a           Every Monday 9am
  fresh Linux machine       fresh Linux machine          GitHub spins up
  (free, ~2 min)            (free, ~1 min)               a fresh machine
        │                          │                             │
        │                          │                             │
  ┌─────┴──────┐            ┌──────┴──────┐              Scans the repo for:
  │  Gitleaks  │            │   vulture   │               - Open PRs > 30 days
  │  Semgrep   │            │   knip      │               - Branches > 60 days
  │  Checkov   │            │   deadcode  │               - Repos > 6 months
  └─────┬──────┘            └──────┬──────┘                     │
        │                          │                             │
        ▼                          ▼                             ▼
  GLM 5.3-flash reads       GLM 5.3-flash reads          Creates a GitHub
  scanner output +          scanner output +              Issue with a table
  writes plain-English      explains what's              of everything stale
  fix for each finding      unnecessary + how
        │                   to remove it                  (+ Slack message
        │                          │                       if token is set)
        ▼                          ▼
  Posts comment on PR        Posts comment on PR
  with full findings         with cleanup items
        │                          │
        ▼                          ▼
   Developer reads          Developer reads
   and fixes issues         and cleans up
        │                          │
        └──────────┬───────────────┘
                   ▼
            PR gets merged
       (nothing is blocked —
        report-only always)
```

---

## Workflow 1 — Security Scan (every PR)

```
PR opened / updated
        │
        ▼
Step 1: Get the diff
        What changed in this PR vs main branch
        (only changed lines — not the whole codebase)
        │
        ▼
Step 2: Gitleaks scans the diff
        Looking for: API keys, passwords, tokens, secrets
        hardcoded anywhere in the code or git history
        │
        ▼
Step 3: Semgrep scans the diff
        Looking for: SQL injection, XSS, command injection,
        broken auth — using OWASP Top 10 rules
        │
        ▼
Step 4: Checkov scans the repo
        Looking for: misconfigured Dockerfiles, Terraform,
        Kubernetes YAML, GitHub Actions
        │
        ▼
Step 5: All scanner output → GLM 5.3-flash
        GLM reads what Gitleaks + Semgrep + Checkov found
        and writes a plain-English explanation + exact fix
        for each issue. Deduplicates overlapping findings.
        │
        ▼
Step 6: Post PR comment
        ┌──────────────────────────────────────────┐
        │ ❌ Security Scan — 3 findings             │
        │ 🔴 CRITICAL: 1  🟠 HIGH: 2               │
        │                                          │
        │ 🔴 CRITICAL — Hardcoded SMTP password    │
        │ Scanner: Gitleaks · OWASP: A07:2021      │
        │ File: scripts/import-users.py · line 122 │
        │                                          │
        │ Password "Zoop@SendGrid#Prod99" is        │
        │ hardcoded directly in source code.        │
        │                                          │
        │ Fix: Remove the constant, rotate the     │
        │ credential immediately, and load it      │
        │ from an environment variable instead.    │
        └──────────────────────────────────────────┘
```

**What each scanner catches:**

| Scanner | Finds |
|---|---|
| Gitleaks | Hardcoded API keys, passwords, tokens, connection strings |
| Semgrep | SQL injection, XSS, command injection, path traversal, insecure deserialization |
| Checkov | Open S3 buckets, missing TLS, privileged containers, weak IAM policies |
| GLM 5.3 | Anything the scanners miss + writes the fix for everything |

---

## Workflow 2 — Cleanup Scan (every PR)

```
PR opened / updated
        │
        ▼
Step 1: Detect language(s) in the repo
        Go? Python? JavaScript/TypeScript?
        Runs only the relevant scanner for each.
        │
        ▼
Step 2: Run language-specific dead code scanners
        ┌──────────┬────────────┬───────────────────┐
        │ Python   │ JS / TS    │ Go                │
        │ vulture  │ knip       │ deadcode          │
        │          │            │ go mod tidy diff  │
        └──────────┴────────────┴───────────────────┘
        Each scanner finds functions, variables,
        imports, and packages that exist but are
        never used anywhere.
        │
        ▼
Step 3: Scanner output → GLM 5.3-flash
        GLM reads the dead code findings + the diff
        and explains WHY each item is unnecessary
        and exactly how to remove it.
        │
        ▼
Step 4: Post PR comment
        ┌──────────────────────────────────────────┐
        │ 🧹 Cleanup Scan — 3 items to clean up    │
        │                                          │
        │ 💀 Dead code — log_campaign_results      │
        │ Scanner: vulture                         │
        │ File: scripts/import-users.py · line 96  │
        │                                          │
        │ This function is defined but never       │
        │ called anywhere in the codebase.         │
        │                                          │
        │ Fix: Delete the function entirely.       │
        │ If it was meant to be used, wire it      │
        │ into main() before merging.              │
        └──────────────────────────────────────────┘
```

**What gets flagged:**

| Category | Example |
|---|---|
| Dead code | Function defined, never called |
| Unused import | `import os` but `os` never used |
| Unused dependency | `requests` in requirements.txt, never imported |
| Dead endpoint | Route registered in router, no caller wires to it |
| Duplicate logic | Same 10-line block exists in 3 places |

---

## Workflow 3 — Org Cleanup (every Monday)

```
Monday 9am UTC — GitHub triggers automatically
        │
        ▼
Step 1: Scan all open PRs in the repo
        Flag any PR that has had no activity
        (no commits, no comments, no reviews)
        for more than 30 days.
        │
        ▼
Step 2: Scan all branches
        Flag any branch (except main/master/develop)
        that has had no new commits
        for more than 60 days.
        │
        ▼
Step 3: Scan all org repos (if org access)
        Flag any repo that has had no pushes
        for more than 6 months.
        │
        ▼
Step 4: Create GitHub Issue
        ┌──────────────────────────────────────────┐
        │ 🧹 Weekly Cleanup Report — 2026-10-12    │
        │                                          │
        │ ## Stale PRs (open > 30 days)            │
        │ | PR  | Title       | Author | Days |    │
        │ | #42 | Add Q4 ...  | @dev   | 35d  |    │
        │                                          │
        │ ## Stale Branches (no activity > 60d)    │
        │ | Branch              | Last commit |    │
        │ | feature/old-templ.. | 2026-07-31  |    │
        │                                          │
        │ Nothing is deleted automatically.        │
        │ Review and close/delete manually.        │
        └──────────────────────────────────────────┘
        │
        ▼
Step 5: (Optional) Post summary to Slack
        🧹 Weekly Org Cleanup Report
        2 stale PRs · 1 stale branch · 0 stale repos
        Check the GitHub issue for the full breakdown.
```

---

## What a developer experiences

```
Day in the life of a developer at Zoop:

1. Developer writes code and opens a PR
   → No extra steps. Just open the PR as normal.

2. Within seconds: GitHub shows "Security Scan — in progress..."
   and "Cleanup Scan — in progress..."

3. Within ~2 minutes: Both checks complete.

   If everything is clean:
   ┌──────────────────────────────────┐
   │ ✅ Security Scan — Clean         │
   │ ✅ Cleanup Scan — Clean          │
   └──────────────────────────────────┘
   → Developer merges normally.

   If issues are found:
   ┌──────────────────────────────────┐
   │ ⚠️ Security Scan — 2 findings    │
   │ ⚠️ Cleanup Scan — 1 item         │
   └──────────────────────────────────┘
   → Developer reads the comment, fixes the issues,
     pushes again → scans re-run on the new commit.

4. PR gets merged when ready.
   Nothing is ever blocked automatically.
```

---

## How to add to any repo

```
Any repo
├── .github/
│   └── workflows/
│       ├── zoop-security-scan.yml   ← copy from adinzoop/zoop-workflows
│       ├── zoop-cleanup-scan.yml    ← copy from adinzoop/zoop-workflows
│       └── zoop-org-cleanup.yml     ← copy from adinzoop/zoop-workflows
└── ... rest of repo


Repo Settings → Secrets → New secret:
  OPENROUTER_API_KEY = (get from Adin)


Done. Open any PR and it runs.
```

**Optional extras:**

| Secret | What it adds |
|---|---|
| `SLACK_BOT_TOKEN` | Monday cleanup report posts to Slack |
| `SLACK_CLEANUP_CHANNEL` | Which Slack channel (default: #dev-ops) |

---

## Cost

| Item | Cost |
|---|---|
| GitHub Actions (2000 min/month free) | $0 |
| OpenRouter / GLM 5.3-flash | ~$2–5/month across all repos |
| Everything else | $0 |
| **Total** | **~$2–5/month** |

Each PR scan uses ~$0.001–0.003 of GLM credits. At 100 PRs/month across all repos, that's under $1.

---

## What's NOT needed

- No server to host
- No database
- No tunnel
- No dashboard
- No Docker
- No deployment pipeline
- No GitHub App

Just 3 files + 1 secret per repo.
