---
name: _pr-review-rubric
description: >-
  Rubric, severity scale, tool routing, and output contract for reviewing a
  pull-request (branch) diff. Load this whenever performing a security review
  (injection, secrets in code, authz/permission gaps, unsafe deserialization,
  insecure input handling, sensitive-data exposure), a quality review (code
  smells, complexity, duplication, test suite, diff coverage), or an intent
  review (does the change match the speckit spec / Jira acceptance criteria) of
  changed code — including when a security, quality, or intent review subagent
  needs the shared checklist and finding format.
---

# PR review

Shared rubric for reviewing a pull-request diff. A review is one of three types —
**security**, **quality**, or **intent** — and each finding records which type
produced it. Review only the changed lines and the code they directly touch; do
not audit the whole repository.

## Severity scale

Use exactly these five levels. Do not invent additional levels.

- **blocker** — must be fixed before merge; ships a bug, vulnerability, or breakage.
- **important** — should be fixed before merge; real defect or risk, but not release-stopping.
- **minor** — worth fixing; limited impact or easy to defer.
- **nit** — cosmetic or stylistic; take it or leave it.
- **question** — not a defect claim; asks the author to clarify intent or confirm an assumption.

## Output contract

Every finding must include all of the following:

- **File path** — repo-relative path.
- **Line(s)** — the specific line or line range on the changed diff.
- **Severity** — one of the five levels above.
- **Review type** — `security`, `quality`, or `intent` (which review produced it).
- **Comment text** — the exact text as it would appear on the PR. It must be
  self-contained (understandable without this rubric or the review conversation),
  actionable, and suggest a concrete fix where one is possible.

Prefer a small set of well-chosen findings over an exhaustive dump.

## Security review

Scope: **security-relevant correctness only.** In scope: injection, secrets
committed in code, authorization/permission gaps, unsafe deserialization,
insecure handling of untrusted input, and sensitive-data exposure. Out of scope:
general style, maintainability, or non-security smells (that is the quality review).

Do all three:

1. **Snyk MCP** — dependency and code vulnerability scanning. Use
   `mcp__Snyk__snyk_sca_scan` for dependency/SCA vulnerabilities and
   `mcp__Snyk__snyk_code_scan` for first-party code (SAST) vulnerabilities.
2. **SonarQube MCP, security-rule categories only** — do **not** pull general
   code smells here. Use `mcp__sonarqube__search_security_hotspots` /
   `mcp__sonarqube__show_security_hotspot` for hotspots, and
   `mcp__sonarqube__search_sonar_issues_in_projects` filtered to the
   **vulnerability** issue type. Scope to the PR/branch under review.
3. **Manual diff read** — read the diff yourself for authorization and
   business-logic flaws that scanners cannot catch (e.g. missing ownership
   checks, broken access control, logic that trusts client-supplied identity).

## Quality review

Scope: **code smells, complexity, and duplication**, plus test health.

Do all of the following:

1. **SonarQube MCP, general categories** — code smells, complexity, and
   duplication (this time not restricted to security rules). Use
   `mcp__sonarqube__search_sonar_issues_in_projects`,
   `mcp__sonarqube__search_duplicated_files`, and
   `mcp__sonarqube__get_duplications`, scoped to the PR/branch.
2. **Run the test suite** and confirm it passes. Report the result.
3. **Diff coverage** — coverage on the **changed/new lines only**, not overall
   repo coverage. Threshold is **75%**. If diff coverage is below 75%, raise an
   `important` finding. Always report the actual percentage. Use
   `mcp__sonarqube__get_file_coverage_details` and
   `mcp__sonarqube__search_files_by_coverage`.

## Intent review

Scope: **does the change actually do what it was supposed to do**, checked
against a speckit spec and/or the related Jira ticket's acceptance criteria.
This is not a code-quality or security check — a change can be clean and secure
while still not implementing what was asked, or silently doing more or less
than the ticket describes.

The orchestrating command resolves the spec/ticket reference **before**
dispatching this review — a speckit spec path if one exists, otherwise a Jira
key (derived from the branch name/git tags, or supplied by the user), otherwise
an explicit "none". This is handed to you in the context package. Do not
re-derive it yourself, and if the context package says none was found, **skip
this review silently** (no finding about the absence of a spec or ticket).

Do the applicable one(s):

1. **Speckit spec** — if a spec path was provided, read it and check the
   implementation against it. Flag divergences: missing behavior, extra
   untracked behavior, mismatched contracts.
2. **Jira ticket** — if a ticket key was provided, fetch it with
   `mcp__claude_ai_Atlassian_Rovo__getJiraIssue` and check the diff against its
   description / acceptance-criteria field. If the ticket links a Confluence
   spec, fetch it with `mcp__claude_ai_Atlassian_Rovo__getConfluencePage` and
   check against that too.
3. Report any divergence as a finding with review type `intent`.

## Shared conventions

- If a **CLAUDE.md** exists in the repo root, treat it as an override: where its
  guidance conflicts with anything above, follow CLAUDE.md.
- Prefer fewer, well-chosen findings over an exhaustive list.
- Suggested fixes must be concrete (name the change to make), not vague advice.
