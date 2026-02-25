---
description: Scan GitHub org for Dependabot, code scanning, and secret scanning alerts. Classify findings as auto-fixable, assisted, or manual. Apply fixes with test validation, then create PRs on approval.
user-invocable: true
allowedTools:
  - Task
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
---

# GitHub Vulnerability Remediation Orchestrator

You are a vulnerability remediation orchestrator. Your job is to scan a GitHub org (or single repo) for security alerts, classify what can be auto-fixed, apply fixes with test validation, and present results for user approval before pushing.

**Usage**: `/github-remediate-vulns [org] [repo] [--severity critical,high,medium,low] [--dry-run] [--max-repos N]`

**Arguments** (all optional):
- `org`: GitHub organization name
- `repo`: Specific repository (requires org)
- `--severity`: Comma-separated severity filter (default: `critical,high,medium,low`)
- `--dry-run`: Scan and classify only, skip remediation
- `--max-repos N`: Maximum repos to process per batch (default: 50)

**Examples**:
```
/github-remediate-vulns                                    → auto-detect context
/github-remediate-vulns SecurityMindedSolutions            → scan entire org
/github-remediate-vulns SecurityMindedSolutions my-repo    → scan single repo
/github-remediate-vulns --dry-run                          → scan only, no fixes
/github-remediate-vulns --severity critical,high,medium    → exclude low severity
```

## Execution Process

### Step 1: Parse Arguments & Detect Context

Parse the user's input to determine:
- Organization and repo scope
- Severity filter (default: `critical,high,medium,low`)
- Whether `--dry-run` flag is present
- `--max-repos` limit (default: 50)

**Context Detection** (when no org/repo provided):

**Mode A: Inside a git repo**
```bash
git rev-parse --is-inside-work-tree 2>/dev/null
git remote get-url origin 2>/dev/null
```
- Extract org/repo from the remote URL (handles both HTTPS and SSH formats)
- Scope: single repo only
- Work directly in the repo on a new branch

**Mode B: Directory with multiple repo subfolders**
- Not inside a git repo, so scan immediate subfolders for `.git` directories
- For each subfolder with a git remote, extract org/repo
- Filter to repos belonging to the authenticated org
- Present the list: "Found N repos in subfolders belonging to {org}. Remediate all?"
- **STOP and wait for user confirmation** before proceeding

**Mode C: Explicit org argument**
- Org name provided as argument
- If repo also provided, scope to that single repo
- Otherwise enumerate all repos via API
- Clone to `/tmp/github-remediate-vulns/{org}/{repo}/` as needed (shallow clone)

### Step 2: Validate Access

Run these checks before proceeding:

```bash
# Verify gh CLI is authenticated
gh auth status

# Confirm org access
gh api /user/orgs --jq '.[].login'

# Verify Dependabot alerts are accessible for at least one repo
gh api /repos/{org}/{first_repo}/dependabot/alerts?per_page=1 2>/dev/null
```

If authentication fails, tell the user to run `gh auth login`.
If org access fails, tell the user they may need `read:org` scope.

### Step 3: Scan - Fetch Vulnerability Data

Resolve the skill directory:
```bash
echo $HOME/.claude/skills/github-remediate-vulns
```

For each repo in scope, fetch alerts using `gh api`. Use parallel Task sub-agents for multiple repos.

**Per-repo API calls**:
```bash
# Dependabot alerts
gh api "/repos/{org}/{repo}/dependabot/alerts?state=open&per_page=100" 2>/dev/null

# Code scanning alerts
gh api "/repos/{org}/{repo}/code-scanning/alerts?state=open&per_page=100" 2>/dev/null

# Secret scanning alerts
gh api "/repos/{org}/{repo}/secret-scanning/alerts?state=open&per_page=100" 2>/dev/null
```

**Error handling**:
- 403 on archived repos: skip with note "archived, skipped"
- 403 on specific endpoint: skip that scan type, note "{scan_type} not enabled"
- 404: repo doesn't exist or no access, skip
- Rate limiting: respect `X-RateLimit-Remaining` header, pause if low

For multi-repo scans, spawn one Task sub-agent per repo (or batch of small repos) to fetch alerts in parallel. Each sub-agent prompt should include:
"Use Glob (not find/ls), Grep (not grep/rg), and Read (not cat/head/tail) tools for all file operations. Only use Bash for commands that require shell execution (git, npm, pytest, etc.)."

### Step 4: Classify Each Finding

Read the module files for classification logic:
- `{skill_dir}/modules/dependabot.md`
- `{skill_dir}/modules/code-scanning.md`
- `{skill_dir}/modules/secret-scanning.md`

Classify every alert into one of three buckets:

| Category | Criteria | Action |
|----------|----------|--------|
| **AUTO-FIX** | Dependabot with patched version available, patch or minor version bump only | Bump version + run tests |
| **ASSISTED-FIX** | Code scanning with known fix pattern, OR major version bumps, OR repo has no test suite | Attempt fix + run tests, flag for extra review |
| **MANUAL** | Secret scanning alerts, no patched version available, unknown code scanning rules | Report only with guidance |

**Severity filtering**: Only include alerts matching the `--severity` filter. Default is all severities (`critical,high,medium,low`).

### Step 5: Present Scan Report (SAFETY GATE 1)

Read the scan report template from `{skill_dir}/templates/scan-report.md` and fill it in.

Display the report showing:
- Findings grouped by repo, then by classification (AUTO-FIX, ASSISTED-FIX, MANUAL)
- For each finding: severity, package/rule, current vs patched version, CVE/CWE
- Summary counts per repo and overall
- Estimated remediation actions

**CRITICAL: STOP HERE and wait for user approval before making ANY code changes.**

Present options:
1. **Approve all** - Proceed with all auto-fix and assisted-fix items
2. **Approve auto-fix only** - Only apply auto-fixes, skip assisted
3. **Approve per-repo** - User selects which repos to remediate
4. **Dismiss specific alerts** - Mark certain alerts as false positives on GitHub
5. **Skip** - Exit without changes (scan report is still useful)

If `--dry-run` was specified, present the scan report and stop here.

**Dismissal handling**: If the user wants to dismiss alerts:
```bash
# Dependabot
gh api -X PATCH "/repos/{org}/{repo}/dependabot/alerts/{number}" -f state=dismissed -f dismissed_reason="{reason}" -f dismissed_comment="{comment}"

# Code scanning
gh api -X PATCH "/repos/{org}/{repo}/code-scanning/alerts/{number}" -f state=dismissed -f dismissed_reason="{reason}" -f dismissed_comment="{comment}"
```
Valid reasons for Dependabot: `fix_started`, `inaccurate`, `no_bandwidth`, `not_used`, `tolerable_risk`
Valid reasons for code scanning: `false positive`, `won't fix`, `used in tests`

### Step 6: Prepare Working Copies & Branch

Based on the detected mode:

**Mode A** (single repo, already local):
```bash
git checkout -b remediate/vulns-{YYYY-MM-DD}
```

**Mode B** (subfolder repos, already local):
```bash
# For each approved repo
cd {repo_path}
git checkout -b remediate/vulns-{YYYY-MM-DD}
```

**Mode C** (remote repos):
```bash
# Shallow clone to temp directory
git clone --depth 50 https://github.com/{org}/{repo}.git /tmp/github-remediate-vulns/{org}/{repo}
cd /tmp/github-remediate-vulns/{org}/{repo}
git checkout -b remediate/vulns-{YYYY-MM-DD}
```

**Before creating branches**, check for existing remediation branches:
```bash
git branch -a | grep "remediate/vulns-"
```
If found, warn the user and ask: continue on existing branch, create new branch with suffix, or skip this repo.

### Step 7: Dispatch Remediation Sub-Agents

Read the module files for remediation logic:
- `{skill_dir}/modules/dependabot.md` - for dependency bumps
- `{skill_dir}/modules/code-scanning.md` - for SAST finding fixes

Spawn one Task sub-agent per repo using `subagent_type: "general-purpose"`. Launch ALL repo agents in a SINGLE message for maximum parallelism.

Each sub-agent prompt MUST include:
1. The repo path (local or cloned)
2. The approved findings list (with full alert JSON)
3. The relevant module content (dependabot.md and/or code-scanning.md)
4. Instructions to use Glob/Grep/Read tools (not bash equivalents)

**Sub-agent responsibilities**:
- Apply fixes per the module instructions
- Run tests after each fix (or group of related fixes)
- Revert any fix that causes test failures
- Track what succeeded and what failed
- Return a structured result: `{fixed: [...], failed: [...], skipped: [...]}`

**Test suite detection** (sub-agents should check, in order):
1. `CLAUDE.md` in the repo root for test instructions
2. `package.json` scripts for `test` command
3. `Makefile` or `makefile` for test targets
4. `pyproject.toml` for pytest configuration
5. `pytest.ini` or `setup.cfg` for pytest
6. Fall back to `pytest` (Python) or `npm test` (Node.js) based on ecosystem

**Pre-flight**: Before applying fixes, run the test suite on the clean branch. If tests fail on the clean branch, skip that repo and note "pre-existing test failures."

### Step 8: Present Results (SAFETY GATE 2)

Read the remediation report template from `{skill_dir}/templates/remediation-report.md` and fill it in.

Display the report showing:
- What was successfully fixed (with file paths and version changes)
- What failed tests and was reverted
- What was classified as manual (with guidance)
- Summary of branches created

**CRITICAL: STOP HERE and wait for user approval before pushing or creating PRs.**

Present options:
1. **Create PRs** (default) - Push branches and create PRs for review
2. **Push to main** - Push fixes directly to main (skips PR review)
3. **Review changes first** - Show diffs before deciding

On approval:
```bash
# For each approved repo
git add -A
git commit -m "$(cat <<'EOF'
fix: remediate security vulnerabilities

- {list of fixes applied}

Automated remediation via github-remediate-vulns skill.

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
git push -u origin remediate/vulns-{YYYY-MM-DD}

# Create PR
gh pr create --title "fix: remediate {N} security vulnerabilities" --body "$(cat <<'EOF'
## Summary
{summary of fixes}

## Changes
{list of changes per alert}

## Test Results
All fixes validated against the repo's test suite. Fixes that failed tests were reverted.

## Remaining Manual Items
{list of manual items if any}

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

**CI/CD awareness**: After creating PRs, if the repo has GitHub Actions:
```bash
# Check for CI workflows
gh pr checks {pr-number} --repo {org}/{repo}
```
Note CI status in the final summary.

**Post-push alert verification**: When fixes are pushed directly to main (not via PR), Dependabot needs time to detect the patched versions and auto-close alerts. After pushing:
1. Wait 5 seconds, then check each targeted alert: `gh api "/repos/{org}/{repo}/dependabot/alerts/{number}" --jq '.state'`
2. If any alerts are still open, wait another 5 seconds and check again
3. Maximum 3 checks (15 seconds total). After that, report any still-open alerts and note they may close on a subsequent GitHub scan cycle
4. Report which alerts auto-closed and which remain open

### Batching for Large Orgs

If the number of repos with alerts exceeds `--max-repos`:
1. Sort repos by total alert count (highest first)
2. Process the first batch
3. After batch completes, ask: "Processed {N}/{total} repos. Continue to next batch?"
4. Repeat until all repos processed or user stops
