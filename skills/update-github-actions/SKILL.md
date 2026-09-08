---
name: update-github-actions
description: Use when auditing or updating a repo's `.github/workflows` GitHub Actions `uses:` references against the Ed-Fi Alliance action allowlist (`approved.json`). Two modes share the same parsing/classification logic — Audit (read-only, compares against a pending/not-yet-merged allowlist branch or PR) and Update (writes the latest approved SHAs from the merged `main` allowlist, plus GitHub-native actions). Triggers when asked to check, audit, compare, update, upgrade, or bump GitHub Actions in workflow files, or to validate a pending allowlist PR against current workflows.
---

# Manage GitHub Actions Against the Ed-Fi Allowlist

## Overview

Both use cases start from the same place: parse every pinned `uses:` reference in this repo's
`.github/workflows`, fetch a version of the Ed-Fi Alliance action allowlist
(`action-allowedlist/approved.json` in `Ed-Fi-Alliance-OSS/Ed-Fi-Actions`), and classify each
reference against it. They diverge only in *which* allowlist to use and *what to do* with the
classification:

| | **Audit mode** | **Update mode** |
|---|---|---|
| Allowlist source | A **pending** branch/PR (not yet merged) — always ask the user for it | The **merged** `main` branch — fetch directly, no need to ask |
| `actions/*` / `github/*` | Excluded from findings (auto-approved, expected absent from list) | Resolved via GitHub releases, then updated like any other action |
| Writes files? | No — read-only report | Yes — edits `uses:` lines in place |
| Output | Report only | Report + modified workflow files |

## Step 0: Determine the mode

Infer from the request; ask only if genuinely ambiguous:

- **Audit mode** — user says "check", "audit", "compare", or mentions a "pending", "proposed",
  or not-yet-merged allowlist / PR / branch.
- **Update mode** — user says "update", "upgrade", or "bump" workflows to the latest approved
  versions.

If it's unclear which mode is wanted, ask the user directly rather than guessing — the two
modes have very different consequences (read-only report vs. writing to files).

## Step 1: Get the allowlist URL

**Update mode**: fetch directly, no need to ask.
```
https://raw.githubusercontent.com/Ed-Fi-Alliance-OSS/Ed-Fi-Actions/refs/heads/main/action-allowedlist/approved.json
```

**Audit mode**: **always ask the user for the raw GitHub URL** of the pending `approved.json`
before doing anything else — do not guess a branch name or assume `main`. The point of audit
mode is to check a list that is still in flight, so a stale or wrong guess defeats the purpose.
Expected shape:
```
https://raw.githubusercontent.com/Ed-Fi-Alliance-OSS/Ed-Fi-Actions/refs/heads/<branch>/action-allowedlist/approved.json
```
If the user gives a PR number or a `github.com/.../blob/...` URL instead of a raw URL, convert
it to the raw form yourself rather than asking again.

## Step 2: Fetch the list

Fetch the URL (curl or WebFetch) and parse the JSON array. Each entry has:
```json
{ "actionLink": "org/repo", "actionVersion": "<sha-or-null>", "tag": "v1.2.3", "deprecated": true }
```

If the fetch fails or returns invalid/empty content, **stop and tell the user** — do not fall
back to a cached or `main` copy silently (audit mode), and do not proceed with an empty map,
since that would cause every action to be silently skipped (update mode).

## Step 3: Collect `uses:` references from the repo

Search `.github/workflows/**/*.{yml,yaml}` (and `.github/actions/**/action.{yml,yaml}` if present) for lines
matching:
^\s*(?:-\s+)?uses:\s+([A-Za-z0-9_.-]+/[A-Za-z0-9_.-]+)(/[A-Za-z0-9_./-]+)?@([0-9a-fA-F]{40}|[A-Za-z0-9_.-]+)\s*(?:#\s*(\S+).*)?$
Group 1 is the action **repo** (`org/repo`, which is what `approved.json` keys on), group 2 is an optional action subpath (e.g. `/init`), group 3 is the pinned ref (SHA or tag/branch), group 4 is the trailing `# vX.Y.Z` comment if present. Deduplicate by `((repo + subpath), ref)` and track which files/lines each combo appears in.
When resolving against the allowlist or GitHub releases, use **group 1**; when updating a `uses:` line, preserve **group 2** if present.

Do not update reusable-workflow refs pointing at a `.github/workflows/*.yml` (or `.yaml`) path inside another repo — allowlist keys are action repos, not workflow files. Leave them unchanged and report them as **out of scope** (except for the two Step 4 standing exclusions, which should be filtered out entirely).

## Step 4: Apply the standing exclusion (both modes)

These two reusable-workflow refs are pinned to `@main` by design as an accepted risk in this
repo — never flag them as unpinned, deprecated, or missing, and never rewrite them, in either
mode:
- `Ed-Fi-Alliance-OSS/Ed-Fi-Actions/.github/workflows/repository-scanner.yml@main`
- `Ed-Fi-Alliance-OSS/Ed-Fi-Actions/.github/workflows/powershell-analyzer.yml@main`

Filter these out of the working set before Step 5.

**Audit mode only**: also exclude every `actions/*` and `github/*` action (e.g.
`actions/checkout`, `github/codeql-action/*`) from the working set here. These are
auto-approved by the Foundation's process and typically won't even appear in `approved.json` —
that absence is expected, not a finding.

**Update mode**: do **not** exclude `actions/*` / `github/*` — instead handle them specially in
Step 6B (they're resolved via GitHub releases rather than the allowlist, but they are still
updated).

---

## Audit mode: Steps 5A–6A

### Step 5A: Classify what's left

For each remaining `(actionLink, ref)` combo, look up an exact match on `(actionLink,
actionVersion)` in the fetched JSON:

- **Match found, `deprecated: true`** → report as **deprecated**. Also look for a
  non-deprecated entry with the same `actionLink` (last one in array order, Ed-Fi-Actions is
  append-only) and suggest it as the replacement tag/SHA if one exists; if every entry for that
  `actionLink` is deprecated, say so explicitly rather than suggesting a replacement.
- **No match at all** → report as **missing**. Note whether the `actionLink` appears in the
  list under *any* other version (helps distinguish "this action is unknown to the list" from
  "this exact pin is outdated but the action itself is tracked").
- **Match found, not deprecated** → no finding; do not include in the report body (a brief
  count is fine).
- **Ref is a tag/branch instead of a 40-char SHA** (and isn't a Step 4 exclusion) → report as
  **unpinned**, since the allowlist keys off exact SHAs and an unpinned ref can't be matched
  precisely.

### Step 6A: Report (read-only)

- **Deprecated** — table of action, current pin, files using it, suggested replacement (or "no
  non-deprecated replacement exists" if none).
- **Missing from list** — action, current pin, files using it, and whether the `actionLink` is
  tracked under a different version.
- **Unpinned** — action, ref, files using it.
- **Not flagged** — one-line summary count of everything that matched a non-deprecated entry,
  plus a reminder that `actions/*`, `github/*`, and the two `@main` reusable workflows were
  excluded by policy, not because they were checked and passed.

Keep it factual — audit mode reports differences, it doesn't recommend whether to merge the
pending allowlist or update the workflows, and it does not edit any files.

---

## Update mode: Steps 5B–8B

### Step 5B: Build the latest-version map

For each unique `actionLink` in the JSON array:
- Collect all entries with that `actionLink`
- Skip entries where `actionVersion` is `null` (local/relative actions)
- Filter out entries where `deprecated: true`
- The **last** remaining entry in array order is the latest approved version (allowlist is
  append-only and chronologically ordered)

If **every** entry for an `actionLink` is `deprecated: true`, record it as a **blocker** (no
non-deprecated replacement exists) and surface it in the final report — do not silently skip.

Result: a map `actionLink → { actionVersion, tag }`.

### Step 6B: Resolve GitHub-native action versions

For any action whose `actionLink` begins with `actions/` or `github/`, fetch the latest release
from GitHub instead of relying on the allowlist:

**Standard `actions/*` and `github/*` actions** (e.g., `actions/checkout`,
`actions/setup-dotnet`):
- Fetch `https://github.com/{actionLink}/releases/latest` (WebFetch) — it redirects to the
  actual release URL (e.g., `.../releases/tag/v4.2.2`); extract the tag and the commit SHA
  pinned to that release.

**`github/codeql-action` and `github/codeql-action-automation`** (special cases — always track
the `v4.*` line regardless of what's currently pinned):
- Fetch `https://github.com/{actionLink}/releases`, find the most recent release matching major
  `v4`, extract tag and SHA. Use this even if currently pinned to a different major — not
  treated as a major-version bump requiring review, since `v4` is the intended target.
- If no `v4.x` release exists at all, leave unchanged and record as a **fetch failure**.

Merge these resolved versions into the latest-version map from Step 5B.

### Step 7B: Scan and update each workflow file

For each match found in Step 3 (after Step 4's exclusion):
1. Extract `repo` (group 1), `subpath` (group 2, absent for most actions), and the current ref (group 3).
2. Look up `repo` in the latest-version map — the map is keyed on `approved.json`'s `actionLink` (`org/repo`), not on the subpath.
3. **Not found** → skip (not in the allowlist and not GitHub-native).
4. **Found, current SHA already matches latest** → skip (already up to date).
5. **Found, ref is a 40-char SHA that differs** → update the line, replacing both the SHA and
   the comment tag. Preserve original indentation:
   ```
         uses: <repo><subpath>@<newSHA> # <newTag>
   ```
6. **Ref is a tag/branch instead of a 40-char SHA** (e.g. `@v4`, `@main`) → do not rewrite
   automatically; record as an **unpinned reference** — a human should decide the correct
   SHA+tag.

### Step 8B: Validate, then report

Before handing off, validate locally that the edits are syntactically sound:
- Run `actionlint` against `.github/workflows/` if available
- Run `yamllint` if available

Local validation is necessary but not sufficient — "all updated workflows pass CI" ultimately
requires a real CI run after pushing. Note this in the report.

Report sections:
- **Updated** — files modified, with each old SHA/tag → new SHA/tag
- **Already latest** — references that matched the latest SHA (confirmation only)
- **Skipped (unknown)** — actions not in the allowlist and not `actions/*` / `github/*`
- **Blocked (deprecated with no replacement)** — requires human decision to replace or remove
- **Unpinned (tag/branch ref)** — drift from Ed-Fi pinning convention
- **Major-version bumps** — cross-major jumps flagged for review (e.g. a standard
  `actions/*`/`github/*` action whose latest release crosses a major version)
- **Fetch failures** — GitHub release pages that could not be retrieved; those refs left
  unchanged

## Edge Cases (Update mode)

| Situation | Behavior |
|-----------|----------|
| Action not in approved.json and not a GitHub-native action | Leave unchanged |
| GitHub-native action (`actions/*` or `github/*`) | Look up latest release on GitHub instead of allowlist |
| `github/codeql-action` / `github/codeql-action-automation` | Fetch releases page; always use latest `v4.x` release regardless of currently-pinned major |
| GitHub release page fetch fails | Leave unchanged; report the failure |
| Action already at latest SHA | Leave unchanged |
| All non-deprecated entries share same SHA | Still treat last as latest |
| `uses:` references a local path (e.g., `./action`) | Skip — no `@SHA` pattern |
| `uses:` references a reusable workflow (e.g., `org/repo/.github/workflows/foo.yml@SHA`) | Out of scope — allowlist keys are action repos, not workflow files. Leave unchanged and report. |
| `deprecated: true` with no non-deprecated alternative | Leave unchanged and record as a **blocker** |
| `uses:` pinned to a tag or branch instead of a 40-char SHA | Leave unchanged and report as **unpinned** |
| Allowlist WebFetch fails | Abort with error |

## Example (Update mode)

Workflow before:
```yaml
- uses: ossf/scorecard-action@62b2cac7ed8198b15735ed49ab1e5cf35480ba46 # v2.4.0
- uses: dawidd6/action-download-artifact@80620a5d27ce0ae443b965134db88467fc607b43 # v7
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
- uses: some-org/niche-action@abcdef0123456789abcdef0123456789abcdef01 # v1.0.0
```

Version sources:
- `ossf/scorecard-action` → allowlist: SHA `4eaacf0543bb3f2c246792bd56e8cdeffafb205a`, tag `v2.4.3`
- `dawidd6/action-download-artifact` → allowlist: SHA `ac66b43f0e6a346234dd65d4d0c8fbb31cb316e5`, tag `v11`
- `actions/checkout` → not in allowlist; GitHub release → SHA `de0fac2e4500dabe0009e67214ff5f5447ce83dd`, tag `v6.0.2`
- `some-org/niche-action` → not in allowlist and not a GitHub-native action → **skip** (left unchanged)

Workflow after:
```yaml
- uses: ossf/scorecard-action@4eaacf0543bb3f2c246792bd56e8cdeffafb205a # v2.4.3
- uses: dawidd6/action-download-artifact@ac66b43f0e6a346234dd65d4d0c8fbb31cb316e5 # v11
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
- uses: some-org/niche-action@abcdef0123456789abcdef0123456789abcdef01 # v1.0.0
```
