# AI Tools for Ed-Fi SDLC

This repository contains prompts, skills, hooks, and agents for use in the Ed-Fi Alliance software development lifecycle. These are designed to assist developers in automating tasks, generating code, and improving productivity when working with Ed-Fi solutions.

To begin with, some of these files may be either Claude Code or GitHub Copilot specific. Over time we will develop a more standardized approach to how we structure and name these files, but for now you may find a mix of both types of files in this repository. We will be working to ensure that all files are clearly labeled and organized to make it easier for developers to find and use the resources they need.

## Repository Structure

```none
.
├── prompts/           # Claude prompts for various tasks
├── skills/            # Reusable skills for Claude Code
├── hooks/             # Shell hooks for automation
├── agents/            # Agent configurations and definitions
└── docs/              # Documentation and guides
```

## Getting Started

### Installing Skills

The [skills cli](https://skills.sh/) tool is an easy way to install and manage skills from this repository. For example, the following command helps perform a global install of any skill in this repository.

```bash
skills add https://github.com/Ed-Fi-Alliance-OSS/AI-Tools-for-Ed-Fi-SDLC -g --agent claude-code
```

> [!TIP]
> For Windows users, look at your `$env.PATH` variable to find a location for installing this and other CLI tools.
> If there is nothing obvious, a good path to consider is `$env.USERPROFILE\.local\bin`. You can add this to your PATH variable if it's not already included.

> [!WARNING]
> Make sure to review third-party skills carefully before installing, especially with global installations, to ensure they are from a trusted source and do not contain malicious code.

### Installing GitHub Copilot CLI Custom Agents

Agents in the `agents/` directory with an `.agent.md` extension can be used as [GitHub Copilot CLI custom agents](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/create-custom-agents-for-cli). To make an agent available globally across all your repositories, copy it to your user-level agents directory:

**macOS / Linux:**
```bash
mkdir -p ~/.copilot/agents
cp agents/update-dependencies/update-dependencies.agent.md ~/.copilot/agents/
```

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.copilot\agents"
Copy-Item agents\update-dependencies\update-dependencies.agent.md "$env:USERPROFILE\.copilot\agents\"
```

After copying, restart the Copilot CLI to load the new agent. You can then invoke it with `/agent` or by name:

```
copilot --agent update-dependencies --prompt "--minor"
```

### Authoring Skills

- All skills in this repository are intended for use in the Ed-Fi Alliance software development lifecycle.
- When authoring new skills, please follow the existing structure and naming conventions to maintain consistency.
- See [Agent Skills](https://agentskills.io/) for best practices on skill development. Also see: [Extend Claude with skills](https://code.claude.com/docs/en/skills).
- If the skill requires installation of another component, such as a CLI tool, add a README.md file to the skill directory with installation notes.

### Contributing

We welcome contributions! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

This project adheres to the [Contributor Code of Conduct](CODE_OF_CONDUCT.md). By participating, you agree to uphold this code.

## Legal Information

Copyright (c) 2026 Ed-Fi Alliance, LLC, and contributors.

This project is licensed under the [Apache License, Version 2.0](LICENSE). See the LICENSE file for details.

This product includes third-party software. See [NOTICES.md](NOTICES.md) for details.
