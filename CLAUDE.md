# CLAUDE.md

## Project Overview

**AI Tools for Ed-Fi SDLC** — a library of reusable components (skills, prompts, hooks, agents) for Claude Code and GitHub Copilot supporting Ed-Fi Alliance workflows. This is a tools repository, not a software product.

## Repository Structure

- `skills/{name}/` — `SKILL.md` (required), optional `README.md` and supporting files
- `prompts/{scenario}/prompt.md`
- `hooks/{trigger}/hook.sh`
- `agents/{name}/agent.md`
- `docs/` — architecture guides, decision records

## Conventions

- Flat `skills/` namespace (no nested categories). Names use letters, numbers, hyphens only.
- Jira issues tracked via `acli` (see `skills/jira/`). Reference issue IDs in pull request titles, e.g. for Jira issue `ABC-133` create PR title with prefix `[ABC-123]`.
