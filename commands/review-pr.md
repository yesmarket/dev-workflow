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

Resolve the spec/ticket reference for the intent review, in this order, stopping at the first hit:
1. Check whether a speckit spec exists for this change (per the project's speckit conventions). If so, note its path — this is the reference.
2. Otherwise, try to derive a Jira issue key. Jira keys look like ABC-123. Look for one in the branch name (from git branch --show-current, e.g. feature/ABC-123-add-widget) and in any git tags on the change (git tag --points-at HEAD, git describe --tags --abbrev=0). Extract the first [A-Z][A-Z0-9]+-\d+ match.
3. If neither a speckit spec nor a derivable Jira key was found, ask the user directly: "I couldn't find a speckit spec or a Jira key from the branch name/tags — do you have a Jira ticket ID for this change? (or say 'none' to skip the intent review)". Use whatever they provide as the reference, or record "none" if they say to skip.

Record the resolved reference (spec path, Jira key, or "none") — you'll include it in the intent-reviewer's context package in Step 3.

Step 2 — Locate the PR in Azure DevOps
Using the azure-devops MCP server:

First, infer the Azure DevOps project and repository from git:
- Run `git remote -v` to get the remote URL(s).
- Parse the Azure DevOps URL (format: `dev.azure.com/<organization>/<project>/_git/<repository>` or similar) to extract the project and repository names.
- If the remote URL doesn't contain clear project/repo information or you can't parse it, ask the user.

Find the open pull request whose source branch matches the current branch and whose target branch is $1.
Note the PR id — you'll need it to post comments.
If no matching PR exists yet, tell the user and ask whether to proceed as a local-only review (draft comments, nothing to post to).

Step 3 — Review the diff via three dedicated subagents (in parallel)

Do NOT review the diff inline. Instead, assemble one shared context package and dispatch three review subagents in parallel via the Task tool.

Context package (identical for all three subagents, plus one extra item for intent-reviewer):

The PR diff from Step 1.
The list of changed files from Step 1.
The commit log from Step 1.
The repo-root CLAUDE.md conventions, if present.
For intent-reviewer only: the resolved spec/ticket reference from Step 1 (spec path, Jira key, or "none").

Dispatch, in a single message with three Task tool calls so they run concurrently:

- one Task with subagent_type: security-reviewer,
- one Task with subagent_type: quality-reviewer, and
- one Task with subagent_type: intent-reviewer.

Give all three subagents the shared context package (intent-reviewer additionally gets the resolved reference). Each subagent already knows its scope and loads the pr-review skill itself — do NOT re-explain the rubric in the prompt; the skill and the subagent definitions are the source of truth. Tell each to PREPARE findings only and to post nothing to the PR.

Each subagent returns findings in the skill's output-contract format: file path, line(s), severity (blocker / important / minor / nit / question), review type (security, quality, or intent), and the exact comment text as it would appear on the PR. The quality subagent additionally reports whether the test suite passed and the measured diff-coverage percentage (and whether it met the 75% threshold).

After all three subagents return, merge their findings into a single list:

Combine all findings from all three subagents.
If findings from more than one review type land on the same file+line, do NOT merge them into one comment. Keep them as separate entries, but place them adjacently in the list so you can see the overlap when deciding.

Carry the quality subagent's test result and diff-coverage number forward to Step 6.

Step 4 — Present the full list first
Display the test results and coverage (from the quality subagent). Then show ALL proposed comments in a table with columns: File/Location | Severity | Type | Description. Include a count by severity below the table. This gives the user the whole picture before deciding. Do not post anything yet.

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
