---
description: Merge a branch into the current branch, resolving conflicts carefully
argument-hint: [branch-name]
allowed-tools: Bash(git:*), Bash(npm:*), Read, Edit, Grep, Glob
---

Merge `$1` (default to `develop` if no argument was given) into the current branch.

## Context gathering (do this BEFORE touching any conflict)

- `git status` and `git branch --show-current` to confirm where I am. If the working tree is dirty, STOP and tell me.
- `git fetch` first so the target branch is current.
- `git log --oneline <target>..HEAD` - what my branch did
- `git log --oneline HEAD..<target>` - what the target branch did
- Read enough of the actual code on both sides to understand the intent behind each change. Do not resolve from the conflict markers alone.

## Do the merge

Run `git merge <target>`. If it merges cleanly, say so, run the tests, and stop.

If it conflicts:

- List every conflicted file first, with a one-line summary of what the conflict is about.
- For each file, `git log -p --merge -- <file>` to see the competing changes.
- Resolve one file at a time. For each, tell me what each side was trying to do BEFORE you write the resolution.

## Resolution rules

- Preserve BOTH sides' intent. Never delete a change just to make the markers go away.
- If both sides changed the same logic in ways that cannot be trivially reconciled, STOP and ask me. Do not guess. Do not pick the side that looks newer.
- Test files: conflicting tests usually mean conflicting behaviour. Treat these as semantic conflicts, not formatting ones.
- Lockfiles (package-lock.json, yarn.lock): do not hand-edit. Take the target branch's version, then regenerate.
- Never invent code that exists on neither side unless it is genuinely required to reconcile the two, and if you do, call it out explicitly.

## After resolving

- `git add` the resolved files but DO NOT COMMIT.
- Run the test suite (check package.json scripts for the right command).
- Run the linter and typecheck if they exist.
- Show me `git diff --cached` so I can review the resolution.
- Summarise: files touched, any judgement calls you made, anything you were unsure about.

If anything goes badly wrong, remind me I can `git merge --abort`.

## Cleanup

Check for and remove any `*_BACKUP_*`, `*_BASE_*`, `*_LOCAL_*`, `*_REMOTE_*` mergetool leftovers.
