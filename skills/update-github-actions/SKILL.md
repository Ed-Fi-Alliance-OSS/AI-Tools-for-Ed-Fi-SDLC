---
name: update-github-actions
description: Use when updating GitHub Actions workflow `uses:` references to the latest approved versions from the Ed-Fi Alliance allowed list. Triggers when asked to update, upgrade, or bump GitHub Actions in workflow files.
---

# Update GitHub Actions to Approved Versions

## Overview

Update `uses:` references in GitHub Actions workflow files to the latest non-deprecated versions. Two sources are consulted:

1. **Ed-Fi Alliance approved allowlist** — for third-party actions
2. **GitHub releases** — for actions published by GitHub itself (`actions/*` and `github/*`)

Actions not found in either source are left untouched.

## Steps

### 1. Fetch the Approved List

Retrieve the current allowlist using WebFetch:

```
URL: https://raw.githubusercontent.com/Ed-Fi-Alliance-OSS/Ed-Fi-Actions/refs/heads/main/action-allowedlist/approved.json
```

### 2. Build the Latest-Version Map

For each unique `actionLink` in the JSON array:
- Collect all entries with that `actionLink`
- **Skip entries where `actionVersion` is `null`** (these are local/relative actions)
- Filter out entries where `deprecated: true`
- The **last** remaining entry in array order is the latest approved version

The result is a map: `actionLink → { actionVersion, tag }`

Example: for `ossf/scorecard-action`, the array has 6 entries — 3 deprecated, then 3 non-deprecated. The latest is the last non-deprecated one: SHA `4eaacf0543bb3f2c246792bd56e8cdeffafb205a`, tag `v2.4.3`.

### 3. Resolve GitHub-Native Action Versions

For any action found in the workflow files whose `actionLink` begins with `actions/` or `github/`, fetch the latest release from GitHub instead of relying on the allowlist.

**Standard `actions/*` actions** (e.g., `actions/checkout`, `actions/setup-dotnet`):

- Fetch `https://github.com/{actionLink}/releases/latest` using WebFetch
- The page redirects to the actual release URL (e.g., `.../releases/tag/v4.2.2`) — extract the tag from that URL
- Find the commit SHA pinned to that release tag (shown on the release page)

**`github/codeql-action`** (special case — multiple major versions maintained simultaneously):

- Fetch `https://github.com/github/codeql-action/releases` using WebFetch
- Find the most recent release whose tag matches `v4.x` (i.e., major version 4)
- Extract that release's tag and commit SHA

Once resolved, these GitHub-native versions are merged into the latest-version map and used in the same update pass as allowlist entries.

### 4. Find Workflow Files

Search for all `*.yml` files under `.github/workflows/` (and any subdirectories).

### 4. Scan and Update Each File

For each file, find lines matching this pattern:

```
uses: <actionLink>@<SHA> # <tag>
```

The regex to match `uses:` lines (with optional leading whitespace):

```
^\s*uses:\s+([^@\s]+)@([a-f0-9]{40})\s*(?:#\s*(.+))?$
```

For each match:
1. Extract `actionLink` (group 1) and current SHA (group 2)
2. Look up `actionLink` in the latest-version map (allowlist entries + GitHub-native entries resolved in step 3)
3. **If not found** → skip (action is not in the allowlist and is not a GitHub-native action)
4. **If found and current SHA matches latest SHA** → skip (already up to date)
5. **If found and SHA differs** → update the line, replacing both the SHA and the comment tag

**Replacement format** (preserve original indentation):
```
      uses: <actionLink>@<newSHA> # <newTag>
```

### 5. Report Changes

After processing all files, report:
- Which files were modified
- Which action references were updated (old SHA/tag → new SHA/tag)
- Which actions were already at the latest version (optional, for confirmation)
- Which actions in the files had no allowlist entry (skipped)

## Edge Cases

| Situation | Behavior |
|-----------|----------|
| Action not in approved.json and not a GitHub-native action | Leave unchanged |
| GitHub-native action (`actions/*` or `github/*`) | Look up latest release on GitHub instead of allowlist |
| `github/codeql-action` | Fetch releases page; use latest `v4.x` release |
| GitHub release page fetch fails | Leave unchanged; report the failure |
| Action already at latest SHA | Leave unchanged |
| All non-deprecated entries share same SHA | Still treat last as latest |
| `uses:` references a local path (e.g., `./action`) | Skip — no `@SHA` pattern |
| `uses:` references a reusable workflow (e.g., `org/repo/.github/workflows/foo.yml@main`) | Look up full path as `actionLink` |
| `deprecated: true` with no non-deprecated alternative | No update target exists; leave unchanged |

## Example

Workflow before:
```yaml
- uses: ossf/scorecard-action@62b2cac7ed8198b15735ed49ab1e5cf35480ba46 # v2.4.0
- uses: dawidd6/action-download-artifact@80620a5d27ce0ae443b965134db88467fc607b43 # v7
- uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
```

Version sources:
- `ossf/scorecard-action` → allowlist: SHA `4eaacf0543bb3f2c246792bd56e8cdeffafb205a`, tag `v2.4.3`
- `dawidd6/action-download-artifact` → allowlist: SHA `ac66b43f0e6a346234dd65d4d0c8fbb31cb316e5`, tag `v11`
- `actions/checkout` → not in allowlist; GitHub release `https://github.com/actions/checkout/releases/latest` → SHA `11bd71901bbe5b1630ceea73d27597364c9af683`, tag `v4.2.2`

Workflow after:
```yaml
- uses: ossf/scorecard-action@4eaacf0543bb3f2c246792bd56e8cdeffafb205a # v2.4.3
- uses: dawidd6/action-download-artifact@ac66b43f0e6a346234dd65d4d0c8fbb31cb316e5 # v11
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```
