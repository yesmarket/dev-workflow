---
description: Promote a change to the next environment branch (dev→sit→uat→prod) by cherry-picking the previous env PR's commits onto a fresh branch and opening a PR in Azure DevOps
argument-hint: "[target-env]  (optional; inferred from the current branch if omitted)"
allowed-tools: AskUserQuestion, Bash(git branch:*), Bash(git status:*), Bash(git fetch:*), Bash(git rev-parse:*), Bash(git merge-base:*), Bash(git log:*), Bash(git show:*), Bash(git cherry:*), Bash(git switch:*), Bash(git checkout:*), Bash(git cherry-pick:*), Bash(git push:*), Bash(git remote:*), mcp__azure-devops__core_list_projects, mcp__azure-devops__repo_get_repo_by_name_or_id, mcp__azure-devops__repo_list_repos_by_project, mcp__azure-devops__repo_list_branches_by_repo, mcp__azure-devops__repo_get_branch_by_name, mcp__azure-devops__repo_list_pull_requests_by_repo_or_project, mcp__azure-devops__repo_get_pull_request_by_id, mcp__azure-devops__repo_create_pull_request, mcp__azure-devops__wit_link_work_item_to_pull_request
---

## Context
- Current branch: !`git branch --show-current`
- Origin remote URL: !`git remote get-url origin`
- Working tree status: !`git status --porcelain=v1`

## What this command does

Promotes a change up the **environment-branching** ladder
`dev → sit → uat → prod`. It infers the current environment from the branch
name, works out the next environment, gathers the commits that went into the
**previous env's PR**, creates a fresh env branch off the target env, cherry-picks
those commits onto it, and opens a PR to the target env branch in Azure DevOps.

The commit set is defined by *what went into the previous env's PR* (not a local
`git log` range) on purpose: some commits may never have reached the target env
yet, and we want the full, correct change set.

## Step 1 — Parse the current branch

Env branches, in order: `dev → sit → uat → prod`.

From the current branch name (e.g. `feature/ABC-123-dev`), derive:
- **Ticket / base** — strip a trailing `-<env>` suffix if present:
  `feature/ABC-123-dev` → base `feature/ABC-123`. A bare `feature/ABC-123`
  (no suffix) is treated as env **`dev`**.
- **Current env** — the stripped suffix (`dev`/`sit`/`uat`), or `dev` if none.
- **Target env** — the next env after current, unless `$1` overrides it. If the
  current env is already `prod`, stop — there's nowhere to promote to.
- **Target branch name** — `<base>-<target-env>`, e.g. `feature/ABC-123-sit`.

If the current branch doesn't look like a feature branch on the ladder (e.g.
you're sitting on `dev`/`sit`/`main`), stop and ask me which feature branch and
target env you meant. If the working tree is dirty, stop and tell me.

Confirm the inferred **current env → target env** and target branch name with
`AskUserQuestion` before continuing.

## Step 2 — Identify the ADO org / project / repo

Parse the origin remote URL (same formats as `/submit-pr`):
- `https://dev.azure.com/{org}/{project}/_git/{repo}`
- `https://{org}@dev.azure.com/{org}/{project}/_git/{repo}`
- `git@ssh.dev.azure.com:v3/{org}/{project}/{repo}`

Confirm the repo with `repo_get_repo_by_name_or_id` (fall back to
`core_list_projects` / `repo_list_repos_by_project` if parsing is ambiguous).

## Step 3 — Determine the commits to cherry-pick

The change set is what went into the **previous env's PR** — i.e. the PR whose
source was the current-env feature branch (`<base>-<current-env>`, or `<base>`
if unsuffixed) and whose target was the current env branch (`<current-env>`).

1. `git fetch` so all env branches and refs are current.
2. Find that PR in ADO via `repo_list_pull_requests_by_repo_or_project`
   (filter by source/target branch; include completed PRs). Use it to confirm
   the source branch name and that it merged into the current env branch.
3. Enumerate the actual commits. The reliable local method (independent of
   whether the target env already has some of them) is the feature branch's own
   commits since it diverged from the current env branch:
   - `git merge-base origin/<current-env> HEAD` → the branch point.
   - `git log --no-merges --reverse <merge-base>..HEAD` → the commit list, oldest
     first (cherry-pick order).
   If the current branch isn't the PR's source (e.g. it was deleted), reconstruct
   the same range from the PR's source branch ref instead.

## Step 4 — Confirm the commit list

Show the ordered commits (short SHA + subject) that will be cherry-picked. Then,
via `AskUserQuestion`, let me:
- **Proceed** with the list as-is, or
- **Adjust** it — I can name additional commit SHAs to include, or ones to drop.

Re-display the final ordered list and get a final confirmation before making any
branch or applying any cherry-pick.

## Step 5 — Create the target branch and cherry-pick

1. Create the target branch fresh from the target env:
   `git switch -c <base>-<target-env> origin/<target-env>`.
   (If a local branch of that name already exists, ask whether to reset it to
   `origin/<target-env>` or pick a new name — don't silently overwrite.)
2. Cherry-pick the confirmed commits in order:
   `git cherry-pick <sha1> <sha2> …`.
3. On a conflict: **stop**. Show the conflicted files and what each side is
   doing, resolve carefully (same discipline as `/safe-merge` — preserve intent,
   never guess), `git add` the resolved files, then `git cherry-pick --continue`.
   Remind me `git cherry-pick --abort` is available if it's going badly.
4. Push: `git push -u origin <base>-<target-env>`.

## Step 6 — Open the PR to the target env

Create the PR with `repo_create_pull_request`:
- **Source:** `<base>-<target-env>`  →  **Target:** `<target-env>`.
- **Title:** carry the ticket + a Conventional-Commits-style summary, tagged with
  the target env, e.g. `feat(ABC-123): promote to sit`.
- **Description:** note this is an environment promotion (`<current-env>` →
  `<target-env>`), list the cherry-picked commits, and reference the previous
  env's PR. End the body with:
  ```
  🤖 Generated with [Claude Code](https://claude.com/claude-code)
  ```
- If the branch/commits reference a work item (e.g. `AB#1234` or the Jira id),
  offer to link it via `wit_link_work_item_to_pull_request`.

## Step 7 — Report

Print: inferred current → target env, the new branch, the commits cherry-picked
(and any conflicts you resolved), and the PR id + web URL. If there's a further
env to promote to afterwards, mention that `/promote` can be run again from the
new branch.
