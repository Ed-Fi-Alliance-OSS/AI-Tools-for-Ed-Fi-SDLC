# Contributing Guide

Thank you for your interest in contributing to AI Tools for Ed-Fi SDLC! This repository contains prompts, skills, hooks, and agents for use in Ed-Fi Alliance development workflows.

## Table of Contents

- [Getting Started](#getting-started)
- [Types of Contributions](#types-of-contributions)
- [Submitting Changes](#submitting-changes)
- [Code Style](#code-style)
- [Questions](#questions)

## Getting Started

1. Fork this repository
2. Clone your fork
3. Create a feature branch: `git checkout -b feature/your-feature-name`
4. Make your changes
5. Submit a pull request

## Types of Contributions

### Prompts

Prompts are templates for Claude to use in specific scenarios. When adding prompts:

- Place in the `prompts/` directory
- Use clear, descriptive naming
- Include comments explaining the prompt's purpose
- Ensure they work well with Claude models

### Skills

Skills are reusable capabilities for Claude Code. When adding skills:

- Place in the `skills/` directory
- Follow the skill template structure
- Ensure compatibility with Claude Code
- Document parameters and usage
- Test thoroughly before submitting

### Hooks

Hooks automate tasks in the Claude Code harness. When adding hooks:

- Place in the `hooks/` directory
- Use shell script format
- Ensure idempotency where applicable
- Document trigger conditions
- Test in the appropriate environment

### Agents

Agents are autonomous Claude configurations. When adding agents:

- Place in the `agents/` directory
- Define clear responsibilities
- Document required context and permissions
- Include example use cases
- Provide implementation guidance

## Submitting Changes

- **Linked issue**: Reference relevant issues if applicable
- **Clear description**: Explain what you're adding and why
- **Documentation**: Update README or docs if needed
- **Testing**: Verify your contribution works as intended

## Code Style

- Use consistent formatting
- Follow existing patterns in the repository
- Write clear, descriptive names for files and sections
- Include comments for complex logic
- Use markdown for documentation

## Security

- Do not include secrets, API keys, or credentials
- Use environment variables or configuration files for sensitive data
- Review your changes before submitting to ensure no sensitive information is exposed

## Questions

If you have questions or need clarification:

- Open an issue in the repository
- Check existing documentation in the `docs/` directory
- Reach out to the Ed-Fi community at [governance@ed-fi.org](mailto:governance@ed-fi.org)

Thank you for contributing!
