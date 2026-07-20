# dev-workflow

A Claude Code plugin packaging a git & pull-request workflow: commit, open/review PRs in Azure DevOps, merge safely, and fix dependency vulnerabilities. Bundles a shared PR-review rubric skill and two least-privilege reviewer subagents.

## What's inside

### Commands
| Command | What it does |
|---|---|
| `/commit` | Runs build + unit tests, moves off protected branches to a feature branch, commits with a Conventional Commits message (+ Claude co-author trailer), then asks before pushing. |
| `/submit-pr` | Opens a PR for the current branch in Azure DevOps (via the ADO MCP server). Defaults the target to `develop`, surfaces env branches (dev/sit/uat/prod), always confirms. |
| `/review-pr <target>` | Reviews the branch's PR against `<target>` using two parallel reviewer subagents, then walks you through each proposed comment one at a time before anything is posted to ADO. |
| `/safe-merge [branch]` | Merges a branch (default `develop`) into the current branch, resolving conflicts carefully — preserving both sides' intent, stopping to ask on semantic conflicts. Stages but never commits. |
| `/snyk-sca [scope]` | Fixes **high/critical** third-party dependency vulnerabilities via Snyk SCA. Auto-applies patch/minor bumps; stops to ask before major bumps. Does **not** touch first-party code (no SAST). |

### Skill
- **`pr-review`** — the shared rubric: severity scale, output contract, tool routing, and the security/quality checklists. Loaded by both reviewer subagents so there is a single source of truth.

### Agents (subagents)
- **`security-reviewer`** — scoped to Snyk + SonarQube security rules + manual diff read. No edit/write, no PR-posting tools.
- **`quality-reviewer`** — scoped to SonarQube general rules, the test suite, diff coverage, and spec/Jira acceptance-criteria checks. No edit/write, no PR-posting tools.

`/review-pr` orchestrates: it fans out to both subagents in parallel, merges findings, and handles the human-in-the-loop posting to Azure DevOps.

## Required MCP servers

These commands/agents call tools from MCP servers that must be configured on the user's machine (this plugin does not bundle server credentials):

- **azure-devops** — `/submit-pr`, `/review-pr`
- **sonarqube** — `/review-pr` (both reviewers)
- **Snyk** — `/review-pr` (security reviewer), `/snyk-sca`
- **Atlassian (Rovo)** — `/review-pr` acceptance-criteria check (optional; skipped if absent)

## Install

Via a marketplace (recommended for teammates):

```
/plugin marketplace add yesmarket/claude-marketplace
/plugin install dev-workflow@yesmarket-claude-marketplace
```

Or point at this repo directly for local development:

```
/plugin marketplace add ~/repos/dev-workflow
```

## Note on the older personal copies

These commands previously lived in `~/.claude/commands/` (and the skill in `~/.claude/skills/pr-review/`). Once this plugin is enabled, delete those personal copies to avoid duplicate/conflicting slash commands:

```
rm ~/.claude/commands/{commit,submit-pr,review-pr,merge-with-conflicts,snyk-scan}.md
rm -r ~/.claude/skills/pr-review
```
