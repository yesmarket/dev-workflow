---
description: Build, test, then stage/commit (Conventional Commits) and optionally push — moving off protected branches first
argument-hint: [optional scope or summary hint]
allowed-tools: AskUserQuestion, Bash(git add:*), Bash(git status:*), Bash(git diff:*), Bash(git log:*), Bash(git commit:*), Bash(git push:*), Bash(git branch:*), Bash(git switch:*), Bash(git checkout:*), Bash(git rev-parse:*), Bash(./gradlew:*), Bash(gradle:*), Bash(npm:*), Bash(yarn:*), Bash(pnpm:*), Bash(mvn:*), Bash(make:*), Bash(cargo:*), Bash(go:*), Bash(pytest:*), Bash(python:*), Bash(python3:*), Bash(dotnet:*)
---

## Context
- Current branch: !`git branch --show-current`
- Recent commits: !`git log --oneline -5`
- Git status: !`git status --porcelain=v1`
- Build files present: !`for f in gradlew build.gradle build.gradle.kts settings.gradle settings.gradle.kts pom.xml package.json yarn.lock pnpm-lock.yaml Cargo.toml go.mod pyproject.toml setup.py Makefile *.sln *.csproj; do [ -e "$f" ] && echo "$f"; done`
- Diff vs HEAD: !`git diff HEAD`

## Task

Work through these steps **in order**. Report the outcome (✅/❌) of each. If a step fails, **stop** and report — do not continue to later steps.

### 1. Build & unit tests (gate — must pass before committing)
Auto-detect the build system from "Build files present" above and run its build + unit tests. If `CLAUDE.md` documents specific build/test commands, prefer those. Common mappings:

| Detected | Command |
|---|---|
| `gradlew` / `build.gradle(.kts)` | `./gradlew build` (runs tests) |
| `pom.xml` | `mvn -B verify` |
| `package.json` | `npm run build` (if a `build` script exists) then `npm test` — use `yarn`/`pnpm` if that lockfile is present |
| `Cargo.toml` | `cargo build && cargo test` |
| `go.mod` | `go build ./... && go test ./...` |
| `pyproject.toml` / `setup.py` | `pytest` (if available) |
| `*.sln` / `*.csproj` | `dotnet build && dotnet test` |

If **no** recognizable build system is found, note that and continue (nothing to gate on). If the build or tests **fail**, stop and report the failure — do not commit.

### 2. Move off a protected/shared branch if needed
If the current branch is a shared/long-lived branch — `main`, `master`, `develop`, `dev`, `sit`, `uat`, `prod`, `staging`, `preprod`, `production`, or matches `release/*` — do **not** commit to it. Instead:
- Propose a feature branch name derived from the diff and `$ARGUMENTS` (e.g. `feature/short-kebab-summary`).
- Use `AskUserQuestion` to confirm the name (offer the suggestion as an option; the user can type their own via "Other").
- Create and switch with `git switch -c <name>`. **No stash needed** — uncommitted work follows you to the new branch.

If already on a feature branch, skip this step.

### 3. Stage
Run `git add -A`.

### 4. Commit (Conventional Commits + co-author trailer)
Generate a [Conventional Commits](https://www.conventionalcommits.org/) message: `type(scope): summary` where type ∈ feat/fix/chore/refactor/docs/test/perf/build/ci. Use `$ARGUMENTS` as a hint for scope/summary if provided. Add a short body explaining the *why* when the change isn't trivial.

Commit with the Claude co-author trailer (Claude contributed to this change):
```
git commit -m "<type(scope): summary>" -m "<optional body>" -m "Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

### 5. Push (ask first)
Use `AskUserQuestion` to ask whether to push to remote now (Yes / No).
- If yes and the branch is new (created in step 2): `git push -u origin <branch>`.
- If yes and the branch already tracks a remote: `git push`.
- If no: skip, and remind the user they can push later.

### 6. Report
Summarise: which branch you committed to, the commit message, whether tests passed, and push status. If a PR would be the natural next step, mention `/submit-pr`.
