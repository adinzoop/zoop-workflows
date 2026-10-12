# Zoop Reusable Workflows

Three GitHub Actions workflows to add to any Zoop repo. Drop the files into `.github/workflows/` and add one secret — done.

---

## Workflows

### 1. `zoop-security-scan.yml` — runs on every PR
Scans for security issues in code changes.

**What it runs:**
- **Gitleaks** — detects hardcoded API keys, passwords, tokens (in code + git history)
- **Semgrep** — SAST: SQL injection, XSS, command injection, auth bypass (OWASP Top 10 rules)
- **Checkov** — IaC: misconfigured Dockerfiles, Terraform, Kubernetes YAML
- **GLM 5.3-flash** — reads all scanner output, writes plain-English remediation for every finding

**Output:** Structured comment posted on the PR with severity badges, file + line, OWASP tag, and exact fix instructions.

---

### 2. `zoop-cleanup-scan.yml` — runs on every PR
Detects code that shouldn't have been added.

**What it runs:**
- **knip** (JS/TS) — unused exports, unused dependencies, dead files
- **vulture** (Python) — unused functions, variables, imports
- **deadcode** (Go) — functions defined but never called
- **go mod tidy diff** — packages in go.mod but never imported
- **GLM 5.3-flash** — synthesizes and explains each issue in plain English

**Output:** PR comment listing every piece of dead or unnecessary code, with instructions to remove it.

---

### 3. `zoop-org-cleanup.yml` — runs every Monday 9am UTC
Weekly org-wide hygiene scan.

**What it checks:**
- Open PRs with no activity in > 30 days
- Branches with no commits in > 60 days
- Repos with no pushes in > 6 months

**Output:** Creates a GitHub Issue in the repo with a full table of everything stale. Optionally posts a summary to Slack.

---

## Setup (per repo)

**Step 1 — Copy the workflow files**

```
.github/
  workflows/
    zoop-security-scan.yml    ← copy from here
    zoop-cleanup-scan.yml     ← copy from here
    zoop-org-cleanup.yml      ← copy from here
```

**Step 2 — Add one secret**

Go to: `Repo → Settings → Secrets and variables → Actions → New repository secret`

| Secret name | Value |
|---|---|
| `OPENROUTER_API_KEY` | (get from Adin) |

**Step 3 — Done.** Open a PR and the scans run automatically.

---

## Optional secrets (for extra features)

| Secret | What it enables |
|---|---|
| `SLACK_BOT_TOKEN` | Slack alert on Monday cleanup report |
| `SLACK_CLEANUP_CHANNEL` | Which channel to post to (default: `#dev-ops`) |
| `ORCHESTRATOR_TUNNEL_URL` | Sync findings to central dashboard |

---

## Everything is report-only

No workflow blocks a merge. Findings are comments and issues — developers read them and decide. Nothing automatic is deleted or closed.
