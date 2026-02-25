# Claude Code GitHub Vulnerability Remediation Skill

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that scans a GitHub org for Dependabot, code scanning, and secret scanning alerts, then automatically remediates what it can and reports what it can't.

## What It Does

1. **Scans** your GitHub org (or a single repo) for open security alerts via the GitHub API
2. **Classifies** each finding as auto-fixable, assisted, or manual
3. **Presents a report** and waits for your approval before touching any code
4. **Applies fixes** with test validation - failed tests trigger automatic revert
5. **Creates PRs** (or pushes to main) on your approval

### Classification

| Category | What | Action |
|----------|------|--------|
| AUTO-FIX | Dependabot alerts with patch/minor version bumps | Bump version + run tests |
| ASSISTED-FIX | Code scanning with known CodeQL patterns, major version bumps | Fix + test + flag for review |
| MANUAL | Secrets, no patches available, unknown code scanning rules | Report with guidance |

### Two Safety Gates

The skill stops and asks for approval at two points:
- **Before any code changes** - you see exactly what will be attempted
- **Before pushing** - you see exactly what changed and can review diffs

## Supported Fix Patterns

### Dependabot (dependency bumps)
- npm (`package.json`)
- pip (`requirements.txt`, `pyproject.toml`)
- Go (`go.mod`)
- Ruby (`Gemfile`)
- Rust (`Cargo.toml`)
- Java (`pom.xml`, `build.gradle`)

### Code Scanning (CodeQL rules)
- `py/incomplete-url-substring-sanitization` - URL parsing fix
- `py/weak-sensitive-data-hashing` - SHA-256/bcrypt upgrade
- `py/sql-injection` - parameterized queries
- `js/xss` - output encoding / DOMPurify
- `py/command-injection` - subprocess with list args

### Secret Scanning
- Report-only with provider-specific rotation checklists
- Supports: GitHub tokens, AWS keys, GCP keys, Azure secrets, Slack tokens, generic API keys

## Installation

Clone the repo and symlink the skill directory:

```bash
git clone https://github.com/SecurityMindedSolutions/claude-github-vuln-remediation-skill.git ~/.claude/skills/claude-github-vuln-remediation-skill
ln -s ~/.claude/skills/claude-github-vuln-remediation-skill/github-remediate-vulns ~/.claude/skills/github-remediate-vulns
```

Or copy just the skill folder:

```bash
cp -R claude-github-vuln-remediation-skill/github-remediate-vulns ~/.claude/skills/github-remediate-vulns
```

### Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed
- `gh` CLI installed and authenticated (`gh auth login`)
- Org access with `read:org` scope
- Dependabot alerts enabled on target repos
- CodeQL configured for code scanning alerts (optional)

### Directory Structure

```
claude-github-vuln-remediation-skill/
├── README.md
├── LICENSE
└── github-remediate-vulns/   # ← symlink or copy this folder to ~/.claude/skills/
    ├── SKILL.md              # Orchestrator prompt (8-step workflow)
    ├── modules/
    │   ├── dependabot.md     # Dependency bump remediation logic
    │   ├── code-scanning.md  # SAST finding fix patterns
    │   └── secret-scanning.md # Secret triage (report-only)
    └── templates/
        ├── scan-report.md        # Phase 1 output template
        └── remediation-report.md # Phase 2 output template
```

## Usage

Inside any Claude Code session:

```
# Auto-detect context (inside a repo, multi-repo directory, or specify org)
/github-remediate-vulns

# Scan an entire org
/github-remediate-vulns SecurityMindedSolutions

# Scan a specific repo
/github-remediate-vulns SecurityMindedSolutions my-repo

# Dry run (scan and classify only, no fixes)
/github-remediate-vulns --dry-run

# Filter by severity (default: all severities)
/github-remediate-vulns --severity critical,high
```

### Context Detection

The skill auto-detects how to run based on where you invoke it:

| Mode | Context | Behavior |
|------|---------|----------|
| A | Inside a git repo | Scans just that repo, works directly in it |
| B | Directory with repo subfolders | Finds all repos belonging to the org, asks which to remediate |
| C | Explicit org argument | Enumerates repos via API, clones to `/tmp/` as needed |

### Options

| Flag | Default | Description |
|------|---------|-------------|
| `--severity` | `critical,high,medium,low` | Comma-separated severity filter |
| `--dry-run` | false | Scan and classify only, no fixes |
| `--max-repos` | 50 | Maximum repos per batch |

## How It Works

### 8-Step Workflow

1. **Detect context** - determines scope (single repo, multi-repo, or org-wide)
2. **Validate access** - checks `gh` CLI auth and org permissions
3. **Scan** - fetches Dependabot, code scanning, and secret scanning alerts via GitHub API
4. **Classify** - sorts each finding into AUTO-FIX, ASSISTED-FIX, or MANUAL
5. **Present scan report** (Safety Gate 1) - shows findings, waits for approval
6. **Prepare branches** - creates `remediate/vulns-{date}` branches
7. **Remediate** - dispatches parallel sub-agents per repo, applies fixes with test validation
8. **Present results** (Safety Gate 2) - shows what changed, waits for push/PR approval

### Error Handling

- **Archived repos**: skipped with note
- **403 on scanning endpoints**: skipped (scan type not enabled for that repo)
- **No test suite found**: downgraded to ASSISTED-FIX with warning
- **Tests fail on clean checkout**: repo skipped (pre-existing failures)
- **Rate limiting**: respects GitHub API limits, `--max-repos` for batching

## Customization

- **Add CodeQL fix patterns** - edit `modules/code-scanning.md` to add new rule-to-fix mappings
- **Add ecosystems** - edit `modules/dependabot.md` to support additional package managers
- **Change report format** - edit templates in `templates/`
- **Add secret providers** - edit `modules/secret-scanning.md` to add rotation checklists for new providers

## License

[MIT](LICENSE)
