---
name: initialize-ed-fi-repository
description: Use when setting up boilerplate files for a new Ed-Fi repository to ask which source repository to copy from instead of hardcoding a template
---

# Initialize Ed-Fi Repository

## Overview

When bootstrapping a new Ed-Fi repository, copy boilerplate files (README, LICENSE, CODE_OF_CONDUCT, CONTRIBUTING, NOTICES, .editorconfig, .vscode/settings.json) from an existing Ed-Fi repository instead of hardcoding the source.

**Core principle:** Ask the user which reference repository to use as the template.

## When to Use

- Starting a new Ed-Fi Alliance repository
- Need to copy standard boilerplate files and configurations
- Want consistency across multiple repositories without maintenance burden

**When NOT to use:**
- Repository already has boilerplate files (skip initialization)
- Creating project-specific templates (use CLAUDE.md instead)

## Workflow

```
User requests: "Initialize this repo with boilerplate"
          ↓
Ask: "Which repository has your template files?"
  (offer choices: Fiona, Fiona-New, ODS-API, AdminApp, etc.)
          ↓
User selects source repo
          ↓
Read boilerplate files from source
  - README.md (extract legal info section)
  - CODE_OF_CONDUCT.md
  - CONTRIBUTORS.md
  - NOTICES.md
  - LICENSE
  - .editorconfig
  - .vscode/settings.json
          ↓
Adapt to new repo name (README title, NOTICES header, etc.)
          ↓
Write files to target repository
```

## Implementation Steps

### 1. Discover Available Sources

Find repositories that can serve as templates:

```bash
# Look for sibling directories or known locations
ls ../*/README.md ../*/LICENSE 2>/dev/null | cut -d/ -f2 | sort -u
```

### 2. Ask User for Selection

```
Which repository should I use as a template?
1. Fiona-New
2. Fiona
3. ODS-API
4. Other (provide path)
```

Use AskUserQuestion to let user choose.

### 3. Read Source Files

For each boilerplate file:
- Read from selected source
- Skip if not found (use Glob to check existence first)
- Preserve exact formatting

**Files to copy:**
- README.md (extract "Legal Information" section + adapt intro/description)
- CODE_OF_CONDUCT.md (unchanged)
- CONTRIBUTORS.md (adapt repository name reference)
- NOTICES.md (adapt repository name)
- LICENSE (unchanged - Apache 2.0)
- .editorconfig (unchanged)
- .vscode/settings.json (unchanged)

### 4. Adapt to Target Repository

For files that reference the repository name:
- Find: `# {SourceRepoName}` in README → Replace: `# {TargetRepoName}`
- Find: `Notices for {SourceRepo}` in NOTICES → Replace: `Notices for {TargetRepo}`
- Find: `Contributors List` section in CONTRIBUTORS → Keep generic reference to current repo

Use Glob to detect current directory name, or ask user:
```
Target repository name (detected: "AI-Tools-for-Ed-Fi-SDLC"): 
```

### 5. Write to Target

Use Write tool for each file. Check for existing files first:
- If README exists: Ask before overwriting
- If LICENSE exists: Skip (licensing is deliberate choice)
- If other files exist: Overwrite (they're boilerplate)

### 6. Verify

After writing, list files created:
```
Created:
  ✓ README.md
  ✓ CODE_OF_CONDUCT.md
  ✓ CONTRIBUTORS.md
  ✓ NOTICES.md
  ✓ LICENSE
  ✓ .editorconfig
  ✓ .vscode/settings.json
```

## Common Mistakes

**Mistake:** Ask user to manually select source repository path instead of discovering them
- **Fix:** Use Glob to discover available repositories first, offer numbered list

**Mistake:** Hardcode "Fiona-New" as source (the original problem)
- **Fix:** Always ask user which repository to use

**Mistake:** User mentions a source ("copy from Fiona-New") so you skip Step 2 (asking)
- **Fix:** Always ask explicitly, even if user pre-answered. Asking confirms choice and discovers what's available.

**Mistake:** Copy all files indiscriminately
- **Fix:** Only copy the 7 boilerplate files listed above

**Mistake:** Don't adapt repository name in README and NOTICES
- **Fix:** Replace source repo name with target repo name in title/headers

**Mistake:** Create .vscode directory if it doesn't exist
- **Fix:** Use Bash mkdir before writing .vscode/settings.json

## Red Flags - STOP if You See These

- Thinking: "User already said Fiona-New, so I'll just use that"
  → **STOP.** Always ask explicitly (Step 2). Asking discovers available options and confirms choice.

- Thinking: "Hardcoding Fiona-New will be faster"
  → **STOP.** The whole point of this skill is to ask. Speed now = maintenance burden later.

- Thinking: "I can skip asking and just copy from the source they mentioned"
  → **STOP.** You must use Step 2 (Ask User for Selection) with AskUserQuestion.

## Example

```
User: "Initialize this repository with boilerplate files from another repo"

You:
1. Discover available sources
2. Ask: "Which repository template would you like to copy from?"
3. User selects "Fiona-New"
4. Read 7 boilerplate files from ../Fiona-New/
5. Ask: "Repository name for this project?"
6. User types "My-New-Project"
7. Adapt README title to "# My-New-Project"
8. Adapt NOTICES header to "# Notices for My-New-Project"
9. Write all files to current repository
10. Verify with file listing
```

## Reference

**Tools used:** Glob (discover sources), Read (source files), Bash (mkdir), Write (target files), AskUserQuestion (user choices)

**Files copied:** README.md, CODE_OF_CONDUCT.md, CONTRIBUTORS.md, NOTICES.md, LICENSE, .editorconfig, .vscode/settings.json

**Adaptation strategy:** Repository name only (in README title and NOTICES header)
