# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**AI Tools for Ed-Fi SDLC** is a library of reusable components for Claude Code and GitHub Copilot that support Ed-Fi Alliance development workflows. The repository contains:

- **Skills** — Documented reference guides for proven techniques, patterns, and workflows
- **Prompts** — Templates for AI assistants in specific scenarios
- **Hooks** — Shell scripts that automate tasks in the Claude Code harness
- **Agents** — Autonomous configurations with defined responsibilities

This is a tools repository, not a software product. The primary consumers are Claude Code and GitHub Copilot instances and developers working on Ed-Fi projects.

## Repository Structure

```
skills/
  {skill-name}/
    SKILL.md           # Skill documentation (required)
    README.md          # Usage notes and dependencies
    [supporting-files] # Tools, utilities, reusable scripts

prompts/
  {scenario}/
    prompt.md          # Prompt template

hooks/
  {trigger-name}/
    hook.sh            # Shell script for automation

agents/
  {agent-name}/
    agent.md           # Agent configuration and purpose
    [context-files]    # Context, instructions, tools

docs/
  # Architecture guides, decision records, etc.
```

## Working with Skills

Each skill directory contains:

- **SKILL.md** — Reference documentation with overview, when to use, implementation details, and common mistakes
- **README.md** — Dependencies and usage notes
- **Supporting files** (optional) — Reusable code or heavy reference material

### Skill Types

- **Discipline-enforcing** (e.g., TDD, verification-before-completion) — Enforce rules under pressure
- **Technique** (e.g., condition-based-waiting) — Concrete how-to guides with steps
- **Pattern** (e.g., flatten-with-flags) — Mental models and design principles
- **Reference** (e.g., API documentation) — Lookup guides and command references

## Key Files and Conventions

### EditorConfig (.editorconfig)

Standard Ed-Fi settings:

- 2-space indents (YAML, JSON, Markdown)
- 4-space indents (C#)
- UTF-8 encoding, LF line endings
- Trim trailing whitespace, insert final newline

### License and Legal

- All work is Apache 2.0 licensed
- NOTICES.md includes third-party attributions
- CODE_OF_CONDUCT.md defines community standards
- CONTRIBUTORS.md lists significant contributors

### Contributing Guidelines

See CONTRIBUTING.md for submission requirements:

- Linked Jira issue
- Clear description
- Documentation updates
- Tested thoroughly

## Jira Integration

Skills may reference or be developed in response to Jira issues (typically in the AI project). When creating or updating skills based on issues:

- Reference the issue in commit messages
- Update the issue with links to the skill
- Use the `jira` skill (via CLI `acli`) for issue management

See `skills/jira/` for Jira CLI reference.

## Architecture Decisions

The repository applies these architectural principles:

### Skills are Discoverable, Not Forced

- Skills are loaded when their description matches the task
- Descriptions must be specific to triggering conditions
- Skills stay lightweight and focused on one technique/pattern

### Skills are Bulletproof

- Discipline-enforcing skills resist rationalization
- Rationalizations are captured, then blocked explicitly
- Testing is thorough and documented

### Skills are Reference-First

- Skills are meant for humans AND agents to reference
- Examples are complete and runnable
- Common mistakes are called out explicitly

### Flat Structure, Rich Metadata

- All skills in one `skills/` namespace (not nested categories)
- Description field is the primary search mechanism
- Name uses only letters, numbers, hyphens (no special chars)

## Working with Other Contribution Types

### Prompts

- Store in `prompts/{scenario}/`
- Include instructions on how Claude should behave
- Document assumptions and constraints
- Test with actual usage scenarios

### Hooks

- Store in `hooks/{trigger-name}/`
- Use shell script format
- Make idempotent where possible
- Document what triggers the hook and what it does

### Agents

- Store in `agents/{agent-name}/`
- Document the agent's role and responsibilities
- List required permissions and context
- Provide example invocations
