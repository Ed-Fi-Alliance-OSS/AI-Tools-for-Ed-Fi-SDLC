---
name: update-dependencies-agent
description: >
  Use when you want to autonomously update package dependencies in .NET,
  Python, or JavaScript projects. The agent handles the full workflow —
  auditing, auto-fixing, applying updates in logical commits, and opening a
  draft PR. For manual step-by-step guidance instead, use the
  update-dependencies skill.
---

# Update Dependencies Agent

## Overview

Invokes the `update-dependencies` agent to autonomously update package
dependencies. The agent audits, auto-fixes where possible, applies updates
in logical commit groups (verifying the build stays green after each),
handles major version research, and opens a draft PR.

For manual or inline dependency guidance, use the `update-dependencies`
skill instead.

## When to Use

- You want the agent to handle the full dependency update workflow without
  inline step-by-step guidance
- You're running a security sweep and want CVEs patched automatically
- You want a clean draft PR summarizing all updates without doing it yourself

## How to Invoke

Tell the agent which mode to run:

| Mode | What happens |
|---|---|
| `--security-only` | Patches vulnerable packages only; no minor/patch or major bumps |
| `--minor` | Security patches + minor/patch bumps; majors researched and reported in PR but not applied |
| `--all` | Full run; pauses to ask you about each major version bump before applying |

If you don't specify a mode, the agent will ask before proceeding.

**Example invocations:**

- "Patch all vulnerable packages with --security-only"
- "Update all dependencies, mode --minor"
- "Run dependency updates --all"

## What the Agent Produces

- A `deps/YYYY-MM-DD` branch (`deps/security-YYYY-MM-DD` for `--security-only`) with one commit per logical package group
- A draft PR with a summary table of all changes, CVEs patched, and any
  majors deferred
- A report of any packages that failed to update and why

## Notes for GitHub Copilot Users

Tool approval prompts (for npm, pip, dotnet, git, gh, etc.) may appear
during execution — this is expected. Approve them to allow the agent to
proceed.

## Reference

- Full agent workflow: [`agents/update-dependencies/AGENT.md`](../../agents/update-dependencies/AGENT.md)
- Manual step-by-step reference: [`skills/update-dependencies/SKILL.md`](../update-dependencies/SKILL.md)
