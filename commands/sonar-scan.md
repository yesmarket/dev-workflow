---
description: Run a local SonarQube scan of the current branch and walk through findings one-by-one (nothing is changed automatically)
argument-hint: "[severity] [type]  e.g. high  |  medium smells  |  critical security"
allowed-tools: AskUserQuestion, Skill, Read, Edit, Bash(git branch:*), Bash(git rev-parse:*), Bash(git symbolic-ref:*), Bash(sonar-scanner:*), Bash(sonar:*), Bash(./gradlew:*), Bash(gradle:*), Bash(mvn:*), Bash(npm:*), Bash(yarn:*), Bash(pnpm:*), Bash(dotnet:*), Bash(pytest:*), Bash(go:*), mcp__sonarqube__search_my_sonarqube_projects, mcp__sonarqube__list_branches, mcp__sonarqube__search_sonar_issues_in_projects, mcp__sonarqube__search_security_hotspots, mcp__sonarqube__show_security_hotspot, mcp__sonarqube__show_rule, mcp__sonarqube__get_component_measures, mcp__sonarqube__search_files_by_coverage, mcp__sonarqube__get_file_coverage_details, mcp__sonarqube__search_duplicated_files, mcp__sonarqube__get_duplications, mcp__sonarqube__get_project_quality_gate_status, mcp__sonarqube__change_sonar_issue_status, mcp__sonarqube__change_security_hotspot_status
---

## Context
- Current branch: !`git branch --show-current`
- Build files present: !`for f in gradlew build.gradle build.gradle.kts settings.gradle settings.gradle.kts pom.xml package.json yarn.lock pnpm-lock.yaml go.mod pyproject.toml *.sln *.csproj sonar-project.properties; do [ -e "$f" ] && echo "$f"; done`
- Sonar connected-mode config: !`cat .sonarlint/connectedMode.json 2>&1 | head -20`

## What this command does

Runs a **local** SonarQube analysis of the code currently on disk (so it
reflects uncommitted / not-yet-PR'd work — which the server otherwise never
sees, since CI only analyses at PR/merge time), then presents the findings in
**separate tables per finding type** and **walks through each one with you**.

Unlike `/snyk-sca`, this command changes **nothing automatically**. Every fix,
status change, or dismissal happens only after you say so, finding by finding.

## Arguments

Parse `$ARGUMENTS` loosely (order-independent, both optional):

- **severity** — one of `blocker`, `high`, `medium`, `low`, `info`. This is a
  **floor**: the scan includes that level **and everything more severe**.
  SonarQube's scale, most to least severe, is `BLOCKER > HIGH > MEDIUM > LOW > INFO`.
  So:
  - `critical` / `blocker` → `BLOCKER`
  - `high` (**default when omitted**) → `HIGH`, `BLOCKER`
  - `medium` → `MEDIUM`, `HIGH`, `BLOCKER`
  - `low` → `LOW`, `MEDIUM`, `HIGH`, `BLOCKER`
  - `info` → all severities
  (Severity gates issues and hotspots. Coverage and duplication have no
  severity — they are always reported against their quality-gate thresholds.)
- **type** — narrows to a single section. One of: `security` (vulnerabilities +
  hotspots), `bugs` (reliability), `smells` (maintainability), `coverage`,
  `duplications`. If omitted, report **all** types.

## Step 1 — Resolve the project key

Resolve in this order (stop at the first hit):
1. `sonar.projectKey` in `.sonarlint/connectedMode.json` (see Context).
2. `sonar.projectKey` in `sonar-project.properties`, `pom.xml`, `build.gradle(.kts)`, or `package.json`.
3. `search_my_sonarqube_projects` (match by repo name) — ask me to pick if ambiguous.

If no key can be found, stop and tell me to run the `sonarqube` plugin's
`sonar-integrate` skill first.

## Step 2 — Choose the analysis branch name

Use the **current git branch name** as `sonar.branch.name` — feature-branch
analyses are attributable and don't collide with the eventual PR analysis.

**Exception:** if the current branch is a protected / long-lived branch
(`main`, `master`, `develop`, `dev`, `sit`, `uat`, `prod`, `staging`,
`preprod`, `production`, or `release/*`), publishing under it would clobber the
real analysis. In that case, use a scratch name
`scratch/sonar-local-<sanitized-branch>` instead and tell me you did.

## Step 3 — Run the local scan

Detect the scanner from the build files and run it, passing the branch name and
project key. Prefer commands documented in `CLAUDE.md` if present.

| Detected | Scan command |
|---|---|
| `build.gradle(.kts)` | `./gradlew sonar -Dsonar.branch.name=<branch>` (Gradle SonarQube plugin) |
| `pom.xml` | `mvn -B sonar:sonar -Dsonar.branch.name=<branch>` |
| `sonar-project.properties` / other | `sonar-scanner -Dsonar.branch.name=<branch>` |

- **Coverage:** SonarQube ingests a coverage report; it does not run tests. If
  coverage is in scope (type omitted or `coverage`), run the project's
  test+coverage task **first** so a report exists (e.g. `./gradlew test jacocoTestReport`,
  `mvn verify`, `npm test -- --coverage`), then scan. If no coverage report can
  be produced, say so and report coverage as "not available this run".
- If the scanner isn't configured/authenticated (no server URL or token), stop
  and report exactly what's missing — don't guess at credentials.
- Wait for the scan to finish and be ingested before reading results. Confirm
  the branch appears via `list_branches`; pass its name as the `branch`
  parameter on every read below.

## Step 4 — Gather findings into tables

Read results over MCP for the analysis branch. Build a table **per type**
(skip types excluded by the `type` arg). Apply the severity floor from the
arguments to issues and hotspots.

- **🔒 Security vulnerabilities** — `search_sonar_issues_in_projects` with
  `impactSoftwareQualities:['SECURITY']`, `issueStatuses:['OPEN','CONFIRMED']`,
  `severities:[<floor set>]`. **Plus Security Hotspots** —
  `search_security_hotspots` with `status:'TO_REVIEW'`.
- **🐞 Bugs (reliability)** — `search_sonar_issues_in_projects` with
  `impactSoftwareQualities:['RELIABILITY']`, same status/severity filters.
- **🧹 Code smells (maintainability)** — `impactSoftwareQualities:['MAINTAINABILITY']`,
  same status/severity filters.
- **🧪 Coverage** — overall/new-code coverage via `get_component_measures`
  (`coverage`, `new_coverage`, `uncovered_lines`); worst files via
  `search_files_by_coverage`.
- **📋 Duplications** — density via `get_component_measures`
  (`duplicated_lines_density`, `duplicated_blocks`); files via
  `search_duplicated_files`.

Render each as its own markdown table. Suggested columns:
- Issues/hotspots: `# | Severity | Rule | File:line | Message`
- Coverage: `# | File | Coverage % | Uncovered lines`
- Duplications: `# | File | Duplicated % | Blocks`

Print the **quality-gate status** (`get_project_quality_gate_status`) as a
one-line header. Then a totals summary per type. If everything is clean at the
requested severity, say so and stop.

## Step 5 — Walk through each finding (nothing automatic)

Go through the findings **one at a time**, most severe first, across all tables.
For each finding:

1. Show the detail: severity, rule (use `show_rule` / `show_security_hotspot`
   for the "why it matters" and remediation guidance), `file:line`, and the
   message. Read the actual source around that line so your recommendation is
   grounded.
2. Ask me what to do via `AskUserQuestion`. Offer the actions that apply:
   - **Fix now** — apply a code fix. For a rule-based issue, delegate to the
     `sonarqube` plugin's **`sonar-fix-issue`** skill (via the Skill tool) with
     the rule key + location. For **coverage**, "fix" means add tests for the
     uncovered lines. For **duplication**, "fix" means refactor the duplicated
     block. Show me the change before saving; make only the edit for this
     finding.
   - **Change status in SonarQube** — for an issue, `change_sonar_issue_status`
     with `accept` (won't fix) or `falsepositive`. For a hotspot,
     `change_security_hotspot_status` → `REVIEWED` with resolution `SAFE` or
     `ACKNOWLEDGED` (ask for a short justification comment). Not applicable to
     coverage/duplication.
   - **Skip** — leave it, move on.
   - **Stop** — end the walk-through and jump to the summary.

Never batch. Act on exactly one finding per confirmation, then move to the next.

## Step 6 — Report

Summarise: per type, how many were fixed / status-changed / skipped, and the
quality-gate status. List any code fixes applied (files touched) and remind me
they're **uncommitted** — suggest `/commit` when I'm ready. Note that
server-side status changes are already live.
