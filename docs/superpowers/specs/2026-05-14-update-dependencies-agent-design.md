# Design: Update Dependencies Agent

**Date:** 2026-05-14
**Status:** Approved
**Branch:** dependencies-agent

## Problem

The `update-dependencies` skill is a 12-step reference guide for humans performing dependency updates inline. It is thorough but purely instructional — the human runs every command, reads every output, and makes every decision. For the mechanical, well-defined portions of the workflow (audit, auto-fix, minor/patch updates, commit grouping, PR creation), an autonomous agent can do the work faster and more consistently, keeping the main conversation clean.

## Proposed Approach

Introduce a new agent (`agents/update-dependencies/AGENT.md`) as the canonical autonomous runner, with a thin skill wrapper (`skills/update-dependencies-agent/SKILL.md`) as the invocation entry point for both Claude Code and GitHub Copilot. The existing `skills/update-dependencies/SKILL.md` is unchanged — it remains the inline reference guide for manual or guided work.

## Architecture

```
agents/
  update-dependencies/
    AGENT.md               ← canonical agent spec (logic lives here)

skills/
  update-dependencies/
    SKILL.md               ← unchanged; inline reference guide
  update-dependencies-agent/
    SKILL.md               ← thin wrapper; invocation contract for both platforms
```

### Responsibilities

| File | Role |
|------|------|
| `agents/update-dependencies/AGENT.md` | Full agent workflow, mode definitions, handoff protocol, git conventions, PR template |
| `skills/update-dependencies-agent/SKILL.md` | When to invoke, what context to provide, how to interpret results; delegates to `AGENT.md` |
| `skills/update-dependencies/SKILL.md` | Manual/inline reference; unchanged |

## Agent Workflow

The agent executes the following phases in order:

### Phase 1: Baseline Check
Run the project's build and test suite. If not green, abort immediately and report the failures. No changes are made.

### Phase 2: Ecosystem Detection
Scan the project for:
- `package.json` → JavaScript (npm)
- `requirements.txt`, `pyproject.toml` → Python (pip / poetry / uv)
- `*.csproj`, `Directory.Packages.props` → .NET

All detected ecosystems are processed in a single run.

### Phase 3: Discover
Run the appropriate audit/outdated command per ecosystem:

| Ecosystem | Command |
|-----------|---------|
| JavaScript | `npm outdated` + `npm audit` |
| Python | `pip list --outdated` + `pip-audit` |
| .NET | `dotnet list package --outdated` + `dotnet list package --vulnerable` |

### Phase 4: Auto-Fix
Run ecosystem auto-fix tools before any manual updates:

| Ecosystem | Command |
|-----------|---------|
| JavaScript | `npm audit fix` |
| Python | `pip-audit --fix` |
| .NET | *(no equivalent; skip)* |

Re-audit after auto-fix. Continue only with what remains unresolved.

### Phase 5: Verify Versions
For each package that needs a manual update, query the registry to confirm the target version exists before writing it. Never use version strings from memory.

### Phase 6: Triage
Categorize remaining updates:
1. Security patches (CVE-linked)
2. Minor/patch bumps
3. Major version bumps

Which categories proceed is controlled by the **mode flag** (see below).

### Phase 7: Apply & Verify
Apply updates in logical commit groups (related packages together; e.g., all `Microsoft.Extensions.*` in one commit). After each group:
- Run build + tests
- If green: continue
- If broken: revert the group, log the failure, continue with the next group

Commit message format:
- Security: `deps: patch <package> <old>→<new> (CVE-XXXX-XXXXX)` with link to advisory
- Minor/patch: `deps: update <package-or-group> <old>→<new>`

### Phase 8: Major Version Research *(--minor and --all modes)*
For each major bump not handled in Phase 7:
1. Fetch the official changelog or migration guide
2. Identify breaking changes relevant to the project
3. **In `--all` mode** — pause and present to the user:
   > "Found major bump: `<package>` v`X` → v`Y`. Key breaking changes: [summary]. Apply, skip, or abort?"
   - If approved: apply, verify, commit. If skipped: note in PR body. If aborted: stop and report.
4. **In `--minor` mode** — do not apply or prompt. Record the research summary in the PR body under "Majors available but not applied."

### Phase 9: Push & Draft PR
Push the branch and open a draft PR. The PR body includes:
- Summary table: package, old version, new version, type (security/minor/major)
- CVEs patched (with advisory links)
- Majors deferred (with reason: skipped by user, mode excluded, or research failed)
- Any packages that failed to update (with failure reason)

## Mode Parameters

The agent requires a mode to be specified at invocation. The skill wrapper surfaces these in plain language.

| Mode | Behavior |
|------|----------|
| `--security-only` | Phases 1–7 for vulnerable packages only. Majors skipped entirely. |
| `--minor` | Phases 1–7 for security + minor/patch bumps. Majors researched (Phase 8) and summarized in the PR body, but not applied. |
| `--all` | Full run: security + minor/patch applied autonomously; majors trigger Phase 8 human handoff. |

## Git Conventions

- **Branch naming:** `deps/YYYY-MM-DD` (general) or `deps/security-YYYY-MM-DD` (`--security-only`)
- **One logical group per commit** — enables `git bisect` if a regression surfaces
- **Lock files committed** — `package-lock.json`, `poetry.lock`, etc. are part of the reproducible build
- **No mixed PRs** — dependency updates are isolated from feature work by definition (agent creates its own branch)

## Error Handling

| Scenario | Behavior |
|----------|----------|
| Baseline build/tests fail | Abort immediately; report failures; no changes made |
| Build/tests fail after a commit group | Revert that group; log which packages caused the failure; continue with next group |
| Version not found in registry | Skip that package; log a warning in the PR body |
| Major version changelog unreachable | Flag in the Phase 8 handoff message; user is informed the summary may be incomplete |
| PR creation fails | Report the branch name and a full summary so the user can open the PR manually |

## Tool Trust Configuration

### Claude Code (AGENT.md frontmatter)

The agent requires several CLI tools to run without user approval at each invocation. This is configured in two complementary ways in the `AGENT.md` frontmatter:

**`permissionMode: bypassPermissions`** — Since this agent is explicitly invoked by the user for a well-defined purpose, bypassing per-call permission prompts is appropriate. The user's act of invoking the agent is the approval.

**`tools` allowlist** — Restricts the agent to only the tools it actually needs, preventing scope creep:

```yaml
---
name: update-dependencies
description: ...
permissionMode: bypassPermissions
tools: >
  Bash(npm *),
  Bash(pip *),
  Bash(pip-audit *),
  Bash(poetry *),
  Bash(uv *),
  Bash(dotnet *),
  Bash(git *),
  Bash(gh *),
  Read,
  Write,
  Edit,
  Glob,
  Grep
---
```

The combination of `bypassPermissions` + explicit `tools` allowlist means: all calls within that list are auto-approved; calls outside it are denied.

### GitHub Copilot

GitHub Copilot skills do not have an equivalent per-command trust mechanism. The thin `SKILL.md` wrapper should note that users may be prompted to approve tool calls during execution, and that this is expected behavior.

## Files Created

| Path | Description |
|------|-------------|
| `agents/update-dependencies/AGENT.md` | Canonical agent spec |
| `skills/update-dependencies-agent/SKILL.md` | Thin invocation wrapper (both platforms) |
| `docs/superpowers/specs/2026-05-14-update-dependencies-agent-design.md` | This document |

## Out of Scope

- Updating `skills/update-dependencies/SKILL.md` (unchanged)
- Support for ecosystems beyond .NET, Python, and JavaScript
- Automated rollback of the entire branch (individual group reverts only)
- Scheduling or CI integration (the agent is invoked on-demand)
