---
description: Open a pull request for the current branch in Azure DevOps via the ADO MCP server
argument-hint: [optional target branch or PR title hint]
allowed-tools: AskUserQuestion, Bash(git branch:*), Bash(git status:*), Bash(git log:*), Bash(git diff:*), Bash(git rev-parse:*), Bash(git remote:*), Bash(git push:*), mcp__azure-devops__core_list_projects, mcp__azure-devops__repo_get_repo_by_name_or_id, mcp__azure-devops__repo_list_repos_by_project, mcp__azure-devops__repo_list_branches_by_repo, mcp__azure-devops__repo_create_pull_request, mcp__azure-devops__repo_get_pull_request_by_id, mcp__azure-devops__repo_update_pull_request_reviewers, mcp__azure-devops__wit_link_work_item_to_pull_request
---

## Context
- Current branch: !`git branch --show-current`
- Origin remote URL: !`git remote get-url origin`
- Local branches: !`git branch --format='%(refname:short)'`
- Unpushed commits (vs upstream): !`git log --oneline @{upstream}..HEAD 2>&1 | head -20`
- Recent commits on this branch: !`git log --oneline -15`

## Task

Open a pull request in Azure DevOps for the current branch. Work through these steps and report the resulting PR URL at the end.

### 1. Identify the ADO org / project / repo
Parse the origin remote URL. Azure DevOps URLs look like:
- `https://dev.azure.com/{org}/{project}/_git/{repo}`
- `https://{org}@dev.azure.com/{org}/{project}/_git/{repo}`
- `git@ssh.dev.azure.com:v3/{org}/{project}/{repo}`

Extract `org`, `project`, and `repo`. Confirm the repo with `repo_get_repo_by_name_or_id` (fall back to `core_list_projects` / `repo_list_repos_by_project` if parsing is ambiguous).

### 2. Ensure the source branch is on the remote
A PR needs the source branch pushed. If "Unpushed commits" above shows commits (or there is no upstream), ask via `AskUserQuestion` whether to `git push -u origin <current-branch>` now. If the user declines, stop — a PR can't be created without the branch on the remote.

### 3. Choose the target branch
List remote branches with `repo_list_branches_by_repo`. Determine the default target:
- Prefer **`develop`** if it exists.
- If the repo uses **environment branching** (branches like `dev`, `sit`, `uat`, `prod`), surface those as options — the correct target is usually the lowest environment (`dev`/`develop`), not `main`/`prod`.
- Otherwise fall back to `main`/`master`.

Always confirm the target with `AskUserQuestion` (offer the detected default first; allow "Other"). If `$ARGUMENTS` names a target branch, use it as the default. Never target the same branch as the source.

### 4. Draft title & description
Summarise the commits between the target branch and HEAD. Use the branch's commit history (`git log <target>..HEAD` conceptually — the "Recent commits" context is a starting point).
- **Title:** a Conventional-Commits-style one-liner (use `$ARGUMENTS` as a hint if given).
- **Description:** a short summary, a bullet list of notable changes, and a testing note. End the body with:
  ```
  🤖 Generated with [Claude Code](https://claude.com/claude-code)
  ```

### 5. Create the PR
Call `repo_create_pull_request` with the org/project/repo, source branch, target branch, title, and description. If commit messages or the branch name reference a work item (e.g. `AB#1234`), offer to link it via `wit_link_work_item_to_pull_request`.

### 6. Report
Print the PR ID and web URL, the source → target branches, and whether the branch was pushed as part of this run.
