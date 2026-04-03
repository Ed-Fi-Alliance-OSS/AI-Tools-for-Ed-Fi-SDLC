---
name: jira
description: Use when working with Jira via the acli jira CLI — viewing, creating, editing, transitioning, searching, and commenting on issues. Triggers on tasks like "create a Jira issue", "transition ticket to In Progress", "search Jira for open bugs", "add a comment to PROJ-123".
allowed-tools: Bash(acli jira *)
---

Use `acli jira <group> <subcommand> [flags]` for all Jira operations. Connection settings are pre-configured.

## Work Items

### View
```bash
acli jira workitem view KEY-123
acli jira workitem view KEY-123 --json
acli jira workitem view KEY-123 --fields summary,status,assignee,description
acli jira workitem view KEY-123 --web
```

### Search (JQL)
```bash
acli jira workitem search --jql "project = PROJ AND status = 'In Progress'"
acli jira workitem search --jql "assignee = currentUser() AND resolution = Unresolved ORDER BY priority"
acli jira workitem search --jql "project = PROJ AND issuetype = Bug" --limit 20
acli jira workitem search --jql "project = PROJ" --fields "key,summary,status,assignee"
acli jira workitem search --jql "project = PROJ" --json
acli jira workitem search --jql "project = PROJ" --csv
acli jira workitem search --jql "project = PROJ" --paginate   # fetch all pages
acli jira workitem search --jql "project = PROJ" --count      # count only
```

### Create
```bash
# Basic
acli jira workitem create --project PROJ --type Story --summary "Add login page"

# With description
acli jira workitem create --project PROJ --type Bug \
  --summary "Crash on login" --description "Steps to reproduce: ..."

# With description from file (preferred for long/multi-line text)
acli jira workitem create --project PROJ --type Task \
  --summary "Write docs" --description-file ./description.txt

# With labels and assignee
acli jira workitem create --project PROJ --type Story \
  --summary "New feature" --label "frontend,api" --assignee user@example.com

# Self-assign
acli jira workitem create --project PROJ --type Task \
  --summary "My task" --assignee @me

# With parent (sub-task)
acli jira workitem create --project PROJ --type Sub-task \
  --summary "Write unit tests" --parent PROJ-123

# Generate JSON template to fill in
acli jira workitem create --generate-json

# Create from completed JSON file
acli jira workitem create --from-json workitem.json

# Bulk create from file
acli jira workitem create-bulk --from-json workitems.json
```

### Edit
```bash
# Edit summary
acli jira workitem edit --key PROJ-123 --summary "Updated title" --yes

# Edit description from file
acli jira workitem edit --key PROJ-123 --description-file ./new-description.txt --yes

# Reassign
acli jira workitem edit --key PROJ-123 --assignee user@example.com --yes
acli jira workitem edit --key PROJ-123 --assignee @me --yes
acli jira workitem edit --key PROJ-123 --remove-assignee --yes

# Edit labels
acli jira workitem edit --key PROJ-123 --labels "bug,regression" --yes
acli jira workitem edit --key PROJ-123 --remove-labels "regression" --yes

# Edit multiple items via JQL
acli jira workitem edit --jql "project = PROJ AND status = 'To Do' AND assignee is EMPTY" \
  --assignee jsmith@example.com --yes

# Change type
acli jira workitem edit --key PROJ-123 --type Bug --yes
```

### Transition (Status Changes)
```bash
# Transition a single item
acli jira workitem transition --key PROJ-123 --status "In Progress" --yes
acli jira workitem transition --key PROJ-123 --status "Done" --yes

# Transition multiple items
acli jira workitem transition --key "PROJ-123,PROJ-124,PROJ-125" --status "Done" --yes

# Transition via JQL
acli jira workitem transition \
  --jql "project = PROJ AND assignee = currentUser() AND status = 'In Review'" \
  --status "Done" --yes

# Transition via filter ID
acli jira workitem transition --filter 10001 --status "To Do" --yes
```

### Comments
```bash
# Add a comment
acli jira workitem comment create --key PROJ-123 --body "Reviewed and looks good."

# Add comment from file
acli jira workitem comment create --key PROJ-123 --body-file comment.txt

# Comment on multiple items via JQL
acli jira workitem comment create \
  --jql "project = PROJ AND fixVersion = 'v2.0'" \
  --body "Included in v2.0 release."

# List comments
acli jira workitem comment list --key PROJ-123

# Update a comment
acli jira workitem comment update --key PROJ-123 --id 10001 --body "Updated note"

# Delete a comment
acli jira workitem comment delete --key PROJ-123 --id 10001

# Set comment visibility to a role
acli jira workitem comment update --key PROJ-123 --id 10001 --body "Internal note" --visibility-role "Administrators"

# Set comment visibility to a group
acli jira workitem comment update --key PROJ-123 --id 10001 --body "Team update" --visibility-group "dev-team"
```

### Links
```bash
# See available link types
acli jira workitem link type

# Create a link
acli jira workitem link create --key PROJ-123 --link "blocks" --target PROJ-456

# List links on an item
acli jira workitem link list --key PROJ-123

# Delete a link
acli jira workitem link delete --key PROJ-123 --link "blocks" --target PROJ-456
```

### Attachments
```bash
# List attachments
acli jira workitem attachment list --key PROJ-123

# Delete an attachment
acli jira workitem attachment delete --key PROJ-123 --id 10001
```

### Clone / Archive
```bash
acli jira workitem clone --key PROJ-123
acli jira workitem archive --key PROJ-123
acli jira workitem unarchive --key PROJ-123
```

## Projects

```bash
acli jira project list
acli jira project list --paginate       # all projects
acli jira project list --recent         # recently viewed
acli jira project list --limit 50 --json
```

## Boards & Sprints

```bash
acli jira board list
acli jira sprint list --board 42
```

## Auth

```bash
acli jira auth login
acli jira auth status
acli jira auth logout
acli jira auth switch
```

## Tips

- Use `--json` for machine-readable output, `--csv` for spreadsheet export.
- Use `--description-file` instead of `--description` for multi-line or special-character content.
- Use `--yes` / `-y` on edit and transition to skip interactive prompts in scripts.
- Use `--paginate` on `search` to retrieve all results beyond the default page size.
- JQL tip: `issuetype in (Bug, Story) AND sprint in openSprints() AND assignee = currentUser()`

## Description Formatting

Write description bodies in **plain Markdown**, not Atlassian Wiki Markup:

| Use this (Markdown) | Not this (Atlassian) |
|---------------------|----------------------|
| `## Heading`        | `h2. Heading`        |
| `**bold**`          | `*bold*`             |
| `*italic*`          | `_italic_`           |
| `` `code` ``        | `{{code}}`           |
| `- item`            | `* item`             |
| ` ```lang ``` `     | `{code:lang}...{code}` |
