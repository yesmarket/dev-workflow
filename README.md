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
| `/snyk-sca [severity] [scope]` | Fixes third-party dependency vulnerabilities via Snyk SCA at a chosen `--severity-threshold` (default **high**, i.e. high + critical; pass `medium`/`critical`/etc. to change the floor). Auto-applies patch/minor bumps; stops to ask before major bumps. Does **not** touch first-party code (no SAST). |
| `/sonar-scan [severity] [type]` | Runs a **local** SonarQube scan of the current branch (so it sees uncommitted work), groups findings into per-type tables (security/hotspots, bugs, code smells, coverage, duplications), then walks you through each one — **nothing is changed automatically**. Severity is a floor (default **high** = high + blocker); optional `type` narrows to one section. |
| `/promote [target-env]` | Promotes a change up the env-branching ladder `dev→sit→uat→prod`. Infers the env from the branch name, gathers the commits from the previous env's PR, cherry-picks them onto a fresh `feature/…-<env>` branch off the target env, and opens the PR in Azure DevOps. |

### Skill
- **`pr-review`** — the shared rubric: severity scale, output contract, tool routing, and the security/quality checklists. Loaded by both reviewer subagents so there is a single source of truth.

### Agents (subagents)
- **`security-reviewer`** — scoped to Snyk + SonarQube security rules + manual diff read. No edit/write, no PR-posting tools.
- **`quality-reviewer`** — scoped to SonarQube general rules, the test suite, diff coverage, and spec/Jira acceptance-criteria checks. No edit/write, no PR-posting tools.

`/review-pr` orchestrates: it fans out to both subagents in parallel, merges findings, and handles the human-in-the-loop posting to Azure DevOps.

## Prerequisites

Install these once per machine before enabling the plugin:

- **Node.js 18+ / `npx`** — both bundled MCP servers launch via `npx`, and several commands shell out to `npm`.
- **git** — used by every command.
- **Azure CLI (`az`)** — the azure-devops MCP server authenticates with `-a azcli`, so run `az login` once. Used by `/submit-pr` and `/review-pr`.
- **Snyk auth** — the Snyk MCP server runs via `npx` (no global install needed), but you must authenticate: run `snyk auth` (or `npx snyk auth`), or export `SNYK_API` (and optionally `SNYK_CFG_ORG`) in your shell. Used by `/snyk-sca` and the security reviewer.
- **SonarQube** — install the `sonarqube@claude-plugins-official` plugin and run its **`sonar-integrate`** skill. That installs the **`sonar` CLI** (which backs the SonarQube MCP server) and writes a `.sonarlint/connectedMode.json` with the project key. `/review-pr`'s reviewers *read existing analysis* over MCP — the branch/PR must already have been analysed by your CI (via `sonar-scanner`, or the Gradle/Maven Sonar plugin). `/sonar-scan`, by contrast, *runs* a local scan, so it additionally needs a working scanner in the repo (a Gradle/Maven Sonar task or `sonar-project.properties`) plus a reachable SonarQube server URL and token in your scanner config/env. Used by `/review-pr` and `/sonar-scan`.
- **A per-repo build/test toolchain** — `/commit` and the quality reviewer run the project's build and unit tests, so the relevant tool must be present in each repo you use them in (e.g. `./gradlew`, `mvn`, `npm`, `cargo`, `go`, `dotnet`).

## MCP servers

This plugin **bundles** two MCP servers in `.mcp.json`. No secrets are committed — each teammate authenticates themselves:

- **azure-devops** — used by `/submit-pr` and `/review-pr`. Runs `npx -y @azure-devops/mcp flexigroup -a azcli`, so it authenticates via the Azure CLI: run `az login` once. (`flexigroup` is the ADO org — edit `.mcp.json` if yours differs.)
- **Snyk** — used by `/snyk-sca` and the security reviewer. Runs `npx -y snyk@latest mcp -t stdio`; authenticate with `snyk auth`, or export `SNYK_API` (and optionally `SNYK_CFG_ORG`) in your shell (the server inherits it).

Two more servers are **prerequisites** you set up separately (deliberately not bundled):

- **sonarqube** — used by `/review-pr` (both reviewers) and `/sonar-scan`. Provided by the `sonarqube@claude-plugins-official` plugin: install it and run its `sonar-integrate` skill, which installs the `sonar` CLI and registers the MCP server. Not bundled here to avoid a duplicate/conflicting server definition. `/sonar-scan` also invokes the plugin's `sonar-fix-issue` skill when you choose to fix a finding.
- **Atlassian (Rovo)** — optional, for the `/review-pr` acceptance-criteria check. Uses the claude.ai Atlassian connector (`mcp__claude_ai_Atlassian_Rovo__*`); skipped silently if absent.

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
