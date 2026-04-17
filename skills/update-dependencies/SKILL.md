---
name: update-dependencies
description: Use when updating package dependencies in .NET, Python, or JavaScript projects — for security patches, major version upgrades, or routine maintenance.
---

# Update Dependencies

## Overview

Systematically update dependencies while minimizing risk of build failures, broken APIs, and runtime regressions. Core principle: **verify before writing, triage before fixing, understand before removing.**

## Process

### 1. Discover Outdated Packages

| Ecosystem | Command |
|-----------|---------|
| .NET | `dotnet list package --outdated` |
| .NET (security) | `dotnet list package --vulnerable` |
| Python | `pip list --outdated` |
| Python (security) | `pip-audit` |
| JavaScript | `npm outdated` |
| JavaScript (security) | `npm audit` |

### 2. Try Auto-Fix First

Before touching individual packages, let the ecosystem resolve what it can automatically. Only proceed to manual updates for what remains.

| Ecosystem | Command |
|-----------|---------|
| JavaScript | `npm audit fix` |
| Python | `pip-audit --fix` |
| .NET | *(no equivalent — skip to step 3)* |

Run the security audit again after auto-fix to see what's left unresolved, then continue with the manual steps below only for those remaining packages.

### 3. Verify Versions Exist Before Writing Them (skip if auto-fix resolved everything)

**30-second check per package. Prevents multi-minute restore failures.**

| Ecosystem | Command |
|-----------|---------|
| .NET | `dotnet package search <Name> --exact-match` |
| Python | `pip index versions <package>` |
| JavaScript | `npm view <package> versions --json` |

**.NET version-lock rule:** `Microsoft.*` packages (e.g., all `Microsoft.Extensions.*`) version-lock together. Ecosystem packages (Npgsql, Serilog sinks, AutoMapper) follow independent cadences — verify each individually.

### 4. Triage Updates

Prioritize in this order:
1. **Security patches** — check advisory for the *first patched version*, jump there directly
2. **Major version bumps** — expect breaking changes; plan time for API migration
3. **Minor/patch bumps** — usually safe; batch these

**Security rule:** Do not probe incrementally (v13→v14→v15). Multiple consecutive major versions may carry the same CVE. Check the advisory and jump directly to the first resolved version.

### 5. Read Migration Guides Before Touching Code

For any major version bump, read the official migration guide or changelog *before* making changes — not after errors appear. Breaking changes are documented; discovering them reactively wastes time.

### 6. Apply Updates

**Batch strategy:** Update related packages together in one commit (e.g., all `Microsoft.Extensions.*`); update unrelated packages in separate commits. This keeps failures isolatable via `git bisect`.

**Branch discipline:** Dependency updates belong on their own branch/PR, separate from feature work. Mixed PRs make regressions hard to attribute.

| Ecosystem | Update command |
|-----------|----------------|
| .NET | `dotnet add package <Name> --version <Version>` |
| .NET (Central Package Management) | Edit `Directory.Packages.props` — if this file exists, version numbers live here, not in `.csproj`. Updating `.csproj` directly silently does nothing. |
| Python (`pip`) | Edit `requirements.txt`, then `pip install -r requirements.txt` |
| Python (`poetry`) | `poetry add <package>@<version>` or edit `pyproject.toml` + `poetry install` |
| Python (`uv`) | `uv add <package>==<version>` |
| JavaScript | `npm install <package>@<version>` |

**Lock files:** Commit `package-lock.json`, `poetry.lock`, etc. — they are part of the reproducible build. After updating, verify the lock file actually changed; a no-op lock file means the version constraint wasn't applied.

### 7. Handle Breaking Changes

**Collect all errors first. Categorize before touching code.**

Fix in this order:
1. **API removed** — find replacement, or remove the call if the modern stack now covers it
2. **Signature changed** — update call sites
3. **New implicit DI dependency** — add missing service registration
4. **Behavior made safe** — remove defensive workaround if it's now redundant
5. **Third-party minor-version regression** — pin to last working version; file an issue upstream

Do not jump between categories mid-fix. Complete each category before moving to the next.

### 8. Transitive Dependency Vulnerabilities

When a vulnerability exists in a transitive dependency (one you don't reference directly):

1. Check if the direct dependency has already released a fix — prefer waiting/upgrading the direct dep
2. If not, add a direct reference to the transitive package at the patched version to force the resolution
3. Remove that direct reference once the upstream dep ships the fix

**JS peer dependency conflicts:** `--legacy-peer-deps` suppresses the conflict; it does not resolve it. Find which packages require incompatible peer versions and upgrade or replace the conflicting one.

### 9. Commit Discipline


- **One logical group per commit** — one package or related group (e.g., all `Microsoft.Extensions.*`). Enables `git bisect` if a regression surfaces later.
- **Security patches:** Link to the CVE or advisory in the commit message.
- **Obsolete API removal:** Commit message must say *why removal is safe* — not just "removed obsolete X."

### 10. Obsolete API Removal

Before removing deprecated or obsolete code, answer:
- *Why was it originally added?*
- *Does the modern stack fully cover that case?*

Document the answers in the commit message.

### 11. Warnings Are Blockers

If the project enforces strict warnings (`TreatWarningsAsErrors`, strict TypeScript, or lint-as-errors), every warning is a build failure. Treat them as first-class errors from the start.

**Never suppress warnings** to get past an upgrade:
- .NET: no `<NoWarn>` in `.csproj`
- JavaScript: no `// eslint-disable`
- Python: no `# noqa` added for upgrade churn

Fix the root cause. Common .NET upgrade warnings: `NU1510`, `NU1605`, `NU1608`, `NU1903`, `IDE0040`, `CA2022`, `EF1003`.

### 12. Verify After Major Upgrades

Run the full test suite. For major DI framework or ORM upgrades, also check:

- **DI container completeness (.NET):** Tests using bare `new ServiceCollection()` may miss implicit registrations the full host provides. After any major library upgrade, grep for these patterns and verify implicit registrations are present (e.g., `services.AddLogging()`). Bare container tests that pass do not prove the hosted application wires correctly.
- **Integration tests**, not just unit tests — framework behavior changes surface here first.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Going package-by-package before trying auto-fix | Run `npm audit fix` / `pip-audit --fix` first; manually update only what remains |
| Writing version strings from memory | Always query the registry first |
| Probing incrementally through CVE-affected versions | Check advisory; jump to first patched version |
| Reading migration guide after errors appear | Read it before touching code |
| Mixing dependency updates with feature work | Separate branch/PR for updates |
| Updating `.csproj` when `Directory.Packages.props` exists | Version numbers live in `Directory.Packages.props` |
| Fixing errors as you encounter them | Collect all, categorize, then fix in order |
| Using `--legacy-peer-deps` for JS peer conflicts | Resolve the actual version conflict |
| Lock file doesn't change after version update | Version constraint wasn't applied — check the manifest |
| Suppressing warnings to unblock the build | Treat as first-class errors; fix root cause |
| Removing deprecated code without understanding its purpose | Answer why it was added, then why removal is safe |
| Testing DI with bare `ServiceCollection` after major upgrade | Verify implicit host registrations are present |
| Assuming version parity across ecosystem packages | Independent cadences — verify each package separately |
