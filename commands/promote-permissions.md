---
description: Scan repos under a path for .claude/settings.local.json files and promote safe, reusable permissions to ~/.claude/settings.json to cut down on prompts
argument-hint: <path> e.g. ./repos
allowed-tools: Bash(find *), Bash(git *), Read, Edit, Grep, Glob, AskUserQuestion
---

## Context
- Target path: $ARGUMENTS
- Discovered local settings files: !`find "$ARGUMENTS" -type f -path "*/.claude/settings.local.json" 2>&1`
- Current global settings: !`cat ~/.claude/settings.json 2>&1`

## Task

Find `permissions.allow` entries across repos' `.claude/settings.local.json` that are safe and generic enough to promote into the user's global `~/.claude/settings.json`, so Claude Code stops re-prompting for the same thing in every repo. An entry qualifies based on what it is, not how many repos it appears in — a single-repo entry is promoted just the same as one repeated everywhere, as long as it's safe. Never edit code — this only touches settings files.

### 0. Gate
If `$ARGUMENTS` is empty or "Discovered local settings files" above is empty/errors, stop and ask the user for a valid path to scan.

### 1. Read the baseline
Parse "Current global settings" above to get the existing `permissions.allow`, `ask`, and `deny` arrays — you'll dedupe against these.

### 2. Read every discovered settings.local.json
For each file found in step 0:
- Read it. If it fails to parse as JSON, note it and skip — don't guess at broken files.
- Pull out `permissions.allow` (ignore `ask`/`deny` — those are the user's explicit "don't auto-approve" calls and are out of scope here).
- Check whether the file is tracked by git: `git -C <repo-root> ls-files --error-unmatch .claude/settings.local.json`. If it IS tracked, flag this loudly in your final report — `settings.local.json` is meant to be gitignored (it's per-machine/per-user), and a tracked copy is a bigger problem than any single permission entry, especially if it also contains secrets (see below).

### 3. Classify every allow entry
Sort each entry into one of these buckets. Judge by the entry's *content*, not just its tool prefix:

**Promote (generic, reusable, read-only or standard build/test — safe across any repo):**
- Read-only Bash: `git status`, `git diff *`, `git log *`, `git branch *`, `git show *`, `ls *`, `cat *` (no secret-looking paths), `grep *`, `rg *`, `find *`, `wc *`, `sort *`, `head *`, `tail *`
- Standard build/test invocations with no repo-specific args baked in: `npm install`, `npm ci`, `npm run *`, `npm test *`, `dotnet build:*`, `dotnet test:*`, `dotnet restore:*`, `mvn *`, `./gradlew *`, `cargo build`/`cargo test`, `go build ./...`, `pytest`, generic `python3 *`
- Read-only MCP tool calls that are broadly useful (e.g. `mcp__sonarqube__search_*`, `mcp__sonarqube__get_*`, `mcp__Snyk__snyk_code_scan`, `mcp__Snyk__snyk_sca_scan`, `mcp__azure-devops__repo_list_*`, `mcp__atlassian__get*`/`search*`)
- `Skill(...)` entries for globally-installed skills
- Broad `Read(...)` grants to system/toolchain paths (`/usr/**`, `/opt/**`, `~/.dotnet/**`, etc.) — read-only and not project-specific

**Do not promote — repo-specific:**
- Anything with the repo's own business text, feature descriptions, or project-unique scripts baked into the command string (e.g. `.specify/scripts/create-new-feature.sh "<long description>"`, custom `./scripts/foo.sh`)
- Repo-specific absolute paths or filenames

**Do not promote — too risky to blanket-allow globally, even if seen safely in one repo:**
- Anything mutating/destructive: `git push *`, `git commit *`, `rm *`, MCP tools that create/update/delete/transition/comment
- Note these separately as "seen locally, but not a candidate for global `allow`"

**Flag as a security concern regardless of promotion:**
- Any entry with what looks like an embedded credential, token, or API key (e.g. `Authorization: Bearer ...`, API keys in query strings). Call these out by repo and file path so the user can rotate/remove them — do not copy these into global settings under any circumstances, and do not print the secret value itself in your report (redact it).

### 4. Build the candidate list
For entries in the "Promote" bucket, dedupe against the global baseline (step 1) — skip anything already globally allowed. Promote every remaining entry that's genuinely safe and generic per step 3's criteria, regardless of how many repos it showed up in — repeat count across repos is not a gate, it's just context. A single-repo entry like `Bash(cargo test:*)` is just as valid a candidate as one seen in five repos, as long as it's read-only or a standard, non-destructive build/test/tool invocation with no repo-specific content baked in. Note the repo count per entry in your report for the user's information, but don't filter candidates by it.

### 5. Present findings and confirm
Show the user:
- The list of candidate entries to add to global `allow`, with a one-line reason each and which repos had them
- Any "not a candidate for global allow" mutating entries (informational only)
- Any tracked-in-git or embedded-secret flags from steps 2/3 (informational only — these need the user's manual follow-up, not an automated edit)

Use `AskUserQuestion` to confirm before writing anything: let the user approve the full list, or pick a subset.

### 6. Apply
If the user approves any entries:
- Back up `~/.claude/settings.json` (copy to `~/.claude/settings.json.bak-<repo-scan>` before editing, if not already backed up this session).
- Merge the approved entries into `permissions.allow`, then rewrite the *entire* array (not just the new entries — the whole thing, so it stays organized as it grows) in this canonical order:
  1. `Bash(...)` entries — read-only/inspection commands first (`git status`, `git diff *`, `git log *`, `git branch *`, `git show *`, `ls *`, `cat *`, `grep *`, `rg *`, `find *`, `wc *`, `sort *`, `head *`, `tail *`, `sed *`, `awk *`, `echo *`, `uniq *`, `cd *`, `lsof *`, etc.), then build/test/toolchain invocations (`npm *`, `dotnet *`, `gradle`/`./gradlew *`, `cargo *`, `go build/test`, `pytest`, `mvn *`, `java -version`, `docker --version`, etc.), then everything else Bash. Alphabetize within each of these three sub-groups.
  2. `Read(...)` entries, alphabetized.
  3. `Skill(...)` entries, alphabetized.
  4. `WebFetch(...)` / `WebSearch` entries, alphabetized.
  5. `mcp__<server>__...` entries, grouped by server name (servers alphabetized), tools alphabetized within each server's group.
  6. Anything not covered above, alphabetized, at the end.
- Apply the same canonical order to `permissions.ask` and `permissions.deny` too if present, so the whole permissions block reads consistently, not just what you added.
- Leave every other part of `settings.json` (env, model, enabledPlugins, extraKnownMarketplaces, etc.) untouched.

Ask separately (default: no) whether to also strip the now-redundant entries from each repo's local `settings.local.json`. Only do this if the user explicitly says yes, and only for entries that were actually promoted.

### 7. Report
Summarize: how many repos were scanned, how many entries were promoted, any mutating/risky entries seen but skipped, and any security flags (tracked-in-git files, embedded secrets) that need manual follow-up.
