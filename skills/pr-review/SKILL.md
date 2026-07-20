---
name: pr-review
description: >-
  Rubric, severity scale, tool routing, and output contract for reviewing a
  pull-request (branch) diff. Load this whenever performing a security review
  (injection, secrets in code, authz/permission gaps, unsafe deserialization,
  insecure input handling, sensitive-data exposure) or a quality review (code
  smells, complexity, duplication, test suite, diff coverage) of changed code —
  including when a security or quality review subagent needs the shared checklist
  and finding format.
---

# PR review

Shared rubric for reviewing a pull-request diff. A review is one of two types —
**security** or **quality** — and each finding records which type produced it.
Review only the changed lines and the code they directly touch; do not audit the
whole repository.

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
- **Review type** — `security` or `quality` (which review produced it).
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
4. **Spec / acceptance-criteria check** — in this order:
   - If a **speckit spec** exists for the change, check the implementation
     against it and flag divergences.
   - Otherwise, if the **Atlassian Rovo MCP** server is connected (tools
     prefixed `mcp__claude_ai_Atlassian_Rovo__`), try to find the related Jira
     ticket and check the change against its acceptance criteria:
     - **Derive the issue key.** Jira keys look like `ABC-123`. Look for one in
       the branch name (`git branch --show-current`, e.g.
       `feature/ABC-123-add-widget`) and in any git tags on the change
       (`git tag --points-at HEAD`, `git describe --tags --abbrev=0`). Extract
       the first `[A-Z][A-Z0-9]+-\d+` match.
     - **Fetch the ticket.** With a key, call
       `mcp__claude_ai_Atlassian_Rovo__getJiraIssue`. If no key can be derived
       but the intent is clear, fall back to
       `mcp__claude_ai_Atlassian_Rovo__searchJiraIssuesUsingJql` (e.g. search
       recent issues by summary text) to identify the likely ticket.
     - **Check acceptance criteria.** Read the ticket's description /
       acceptance-criteria field and flag any divergence between it and the
       change as a `quality` finding. If the ticket links a Confluence spec,
       fetch it with `mcp__claude_ai_Atlassian_Rovo__getConfluencePage` and
       check against that too.
   - If neither a speckit spec nor a derivable Jira ticket exists, **skip this
     step silently** (do not emit a finding about the absence of a spec or ticket).

## Shared conventions

- If a **CLAUDE.md** exists in the repo root, treat it as an override: where its
  guidance conflicts with anything above, follow CLAUDE.md.
- Prefer fewer, well-chosen findings over an exhaustive list.
- Suggested fixes must be concrete (name the change to make), not vague advice.
