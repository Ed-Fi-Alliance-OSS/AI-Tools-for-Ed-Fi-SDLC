---
name: audit-actions-allowlist
description: Use when auditing this repo's `.github/workflows` for GitHub Actions that are deprecated or missing from a pending Ed-Fi Alliance action allowlist update. Triggers when asked to check, audit, or compare workflow actions against an allowlist, approved-list, or pending allowlist PR/branch.
---

# Audit Workflows Against a Pending Action Allowlist

## Overview

Compares every pinned `uses:` reference in this repo's `.github/workflows` against a
**pending** version of the Ed-Fi Alliance action allowlist (`approved.json`) — typically a
branch or PR that hasn't merged to `main` yet. Reports any action pin that is:

- **(a) deprecated** in the pending list, or
- **(b) missing from the pending list entirely** — implying the pinned version is no longer
  supported.

This is a read-only audit. It does not edit workflow files (see the `update-github-actions`
skill for that).

## Step 1: Get the allowlist URL

**Always ask the user for the raw GitHub URL** of the pending `approved.json` before doing
anything else — do not guess a branch name or assume `main`. The point of this skill is to
check a list that is still in flight, so a stale or wrong guess defeats the purpose.

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

If the fetch fails or returns invalid JSON, stop and tell the user — do not fall back to a
cached or `main` copy silently.

## Step 3: Collect `uses:` references from the repo

Search `.github/workflows/**/*.yml` (and `.github/actions/**/*.yml` if present) for lines
matching:
```
uses:\s*([A-Za-z0-9_.-]+/[A-Za-z0-9_./-]+)@([0-9a-f]{40}|[A-Za-z0-9_.\-]+)\s*(?:#\s*(\S+))?
```
Group 1 is `actionLink`, group 2 is the pinned ref (SHA or tag/branch), group 3 is the trailing
`# vX.Y.Z` comment if present. Deduplicate by `(actionLink, ref)` and track which files/lines
each combo appears in.

Skip local/relative references (`uses: ./...`) — these aren't external actions and have no
allowlist entry to check.

## Step 4: Apply the standing exclusions

Two categories are **never reported**, regardless of what the fetched list says:

1. **`actions/*` and `github/*` actions** (e.g. `actions/checkout`, `github/codeql-action/*`).
   These are auto-approved by the Foundation's process and typically won't even appear in
   `approved.json` — that absence is expected, not a finding.
2. **These two specific reusable-workflow refs**, pinned to `@main` by design as an accepted
   risk — do not flag them as unpinned, deprecated, or missing:
   - `Ed-Fi-Alliance-OSS/Ed-Fi-Actions/.github/workflows/repository-scanner.yml@main`
   - `Ed-Fi-Alliance-OSS/Ed-Fi-Actions/.github/workflows/powershell-analyzer.yml@main`

Filter these out of the working set before classifying anything in Step 5.

## Step 5: Classify what's left

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
- **Ref is a tag/branch instead of a 40-char SHA** (and isn't one of the Step 4 exclusions) →
  report as **unpinned**, since the allowlist keys off exact SHAs and an unpinned ref can't be
  matched precisely.

## Step 6: Report

Structure the output as:

- **Deprecated** — table of action, current pin, files using it, suggested replacement (or "no
  non-deprecated replacement exists" if none).
- **Missing from list** — action, current pin, files using it, and whether the `actionLink` is
  tracked under a different version.
- **Unpinned** — action, ref, files using it.
- **Not flagged** — one-line summary count of everything that matched a non-deprecated entry,
  plus a reminder that `actions/*`, `github/*`, and the two `@main` reusable workflows were
  excluded by policy, not because they were checked and passed.

Keep it factual — this skill reports differences, it doesn't recommend whether to merge the
pending allowlist or update the workflows.
