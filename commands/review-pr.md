---
description: >-
  Code-review the current branch's PR against a target branch, then walk through
  each proposed comment one at a time so you decide what actually gets posted to
  the PR in Azure DevOps. Run it from the source branch (e.g. after git checkout
  feature/xyz) and pass the PR's target branch as an argument, for example
  /review-pr develop. Nothing is posted without your explicit approval, comment
  by comment.
argument-hint: [target-branch]
---
You are reviewing the current git branch (the source/feature branch) against the target branch $1.
If $1 is empty, stop and ask the user which target branch the PR merges into (e.g. develop, main) before doing anything else.
Step 1 — Establish context

Run git branch --show-current to get the source branch name.
Run git fetch so the target branch ref is up to date.
Get the diff of the PR with: git diff origin/$1...HEAD (three-dot: changes on this branch since it diverged from $1).

Also run git log origin/$1..HEAD --oneline to see the commits in scope.
Note the list of changed files with git diff --name-only origin/$1...HEAD — you'll hand it to the review subagents.
If a CLAUDE.md exists in the repo root, read it — its conventions are part of the context package in Step 3.

Step 2 — Locate the PR in Azure DevOps
Using the azure-devops MCP server:

Find the open pull request whose source branch matches the current branch and whose target branch is $1.
If you can't determine the ADO project/repo from context, ask the user for the project name (and repo if ambiguous) before continuing.
Note the PR id — you'll need it to post comments.
If no matching PR exists yet, tell the user and ask whether to proceed as a local-only review (draft comments, nothing to post to).

Step 3 — Review the diff via two dedicated subagents (in parallel)

Do NOT review the diff inline. Instead, assemble one shared context package and dispatch two review subagents in parallel via the Task tool.

Context package (identical for both subagents):

The PR diff from Step 1.
The list of changed files from Step 1.
The commit log from Step 1.
The repo-root CLAUDE.md conventions, if present.

Dispatch, in a single message with two Task tool calls so they run concurrently:

- one Task with subagent_type: security-reviewer, and
- one Task with subagent_type: quality-reviewer.

Give both subagents the same context package. Each subagent already knows its scope and loads the pr-review skill itself — do NOT re-explain the rubric in the prompt; the skill and the subagent definitions are the source of truth. Tell each to PREPARE findings only and to post nothing to the PR.

Each subagent returns findings in the skill's output-contract format: file path, line(s), severity (blocker / important / minor / nit / question), review type (security or quality), and the exact comment text as it would appear on the PR. The quality subagent additionally reports whether the test suite passed and the measured diff-coverage percentage (and whether it met the 75% threshold).

After both subagents return, merge their findings into a single list:

Combine all findings from both subagents.
If a security finding and a quality finding land on the same file+line, do NOT merge them into one comment. Keep them as separate entries, but place them adjacently in the list so you can see the overlap when deciding.

Carry the quality subagent's test result and diff-coverage number forward to Step 6.

Step 4 — Present the full list first
Show the user a numbered summary of ALL proposed comments (file:line, severity, review type, one-line gist) so they can see the whole picture before deciding. Do not post anything yet.

Step 5 — Go through them one-by-one
For EACH proposed comment, in order:

Show the full comment: file, line(s), severity, review type, and the exact comment text.
Ask the user: "Post this comment to the PR? (yes / no / edit / skip rest)"
Act on their answer:

yes → post the comment to the PR (on the correct file + line thread) via the azure-devops MCP server, then confirm it posted and move to the next.
no / skip → do not post; move to the next.
edit → ask for the revised wording (or propose a revision), confirm, then post the edited version.
skip rest → stop the loop and go to Step 6.

Never post more than one comment per confirmation. Do not batch-post. Wait for the user's answer each time.

Step 6 — Wrap up
Summarise what happened: how many comments were posted, how many skipped, and list anything the user may still want to address manually. Include whether the test suite passed and whether diff coverage met the 75% threshold (report the actual percentage). Do not change the PR's vote/approval status unless the user explicitly asks.
