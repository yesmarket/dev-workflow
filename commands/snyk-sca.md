---
description: Fix high and critical Snyk SCA vulns in third-party deps
allowed-tools: Bash, Read, Edit
---

Run Snyk SCA (the `snyk_sca_scan` tool) on this project to find security
vulnerabilities in **third-party dependencies only** — open-source
libraries, SDKs, and transitive deps declared in the manifest files
(package.json, pnpm-lock.yaml, pom.xml, build.gradle, requirements.txt,
poetry.lock, go.mod, Cargo.toml, Gemfile, .csproj, packages.config,
Directory.Packages.props, etc.).

Do NOT run Snyk SAST (`snyk_code_scan`). This task is exclusively about
third-party dependency vulnerabilities, not first-party source code.

Use `--severity-threshold=high` so only **high and critical** severity
issues are reported. Ignore medium and low.

If the current working directory is not yet trusted by Snyk, trust it and
proceed with the scan.

For each high or critical finding, classify the required version bump
using semver:

- **Patch bump** (e.g. 1.2.3 → 1.2.4) — apply automatically.
- **Minor bump** (e.g. 1.2.3 → 1.3.0) — apply automatically.
- **Major bump** (e.g. 1.2.3 → 2.0.0) or any change flagged by Snyk as
  potentially breaking — STOP and ask before applying. List all such
  fixes together with the current version, target version, and Snyk's
  breaking-change notes, and wait for explicit approval.

For each fix you apply automatically:
1. Edit the relevant manifest file (and the lockfile, if the ecosystem
   uses one — package-lock.json, pnpm-lock.yaml, poetry.lock, Cargo.lock,
   Gemfile.lock, packages.lock.json, etc.).
2. Run the appropriate install / restore command for this ecosystem:
   - npm: `npm install`
   - pnpm: `pnpm install`
   - yarn: `yarn install`
   - poetry: `poetry lock --no-update`
   - pip: regenerate as appropriate for the project
   - Go: `go mod tidy`
   - Cargo: `cargo update -p <crate>`
   - Bundler: `bundle update <gem>`
   - .NET / NuGet: `dotnet restore` (and `dotnet list package --vulnerable`
     to confirm)
   - Maven: `mvn dependency:resolve`
   - Gradle: `./gradlew dependencies --refresh-dependencies`
3. Do NOT modify any first-party source code as part of this task.

After all auto-applied (patch + minor) fixes are in place, re-run
`snyk_sca_scan` with `--severity-threshold=high` to verify the issues
are resolved. Report:
- Before/after counts of high and critical issues
- Any auto-applied fixes
- Any major-bump fixes pending your approval
- Any issues that could not be fixed (e.g. no patched version available)

Optional scope: $ARGUMENTS
