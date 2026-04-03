---
name: review-model-pr
description: Use when asked to review a pull request or MetaEd model changes for the Ed-Fi-Model repository. Triggers on tasks like "review this PR", "review my changes", "check my MetaEd files".
allowed-tools: Bash(gh *), Read, Glob, Grep
---

Review the pull request or staged changes using the Ed-Fi-Model checklist below. Run `gh pr diff` (or read changed files directly) to see what changed, then work through each category.

## How to Review

1. Get the diff: `gh pr diff <number>` or read changed `.metaed` files directly.
2. Work through every checklist section below.
3. Report findings grouped by category. Provide file + line context for each issue.
4. Distinguish blocking issues (must fix) from suggestions (consider fixing).

---

## Checklist

### 1. Build Validity — BLOCKING
- [ ] Does the MetaEd model build without errors? If you can run `npx metaed build` locally, do so. Otherwise flag any syntax issues you can spot (missing quotes, unclosed strings, mismatched keywords).
- [ ] Are all entity references in `.metaed` domain files actually defined somewhere in the model? (Missing references are a common build failure.)
- [ ] Are all quoted strings properly closed?

### 2. Typos and Grammar — BLOCKING
- [ ] Scan all `documentation` strings for typos (e.g., "TThe", "eleemnts", "to be receives").
- [ ] Check for missing words, duplicate words, or incomplete sentences.
- [ ] Check descriptor `documentation` strings for grammatical correctness.
- [ ] Flag database-centric language: prefer "staff member" over "staff record", "individual" over "person record", etc.

### 3. Documentation Completeness
- [ ] Does every new `DomainEntity`, `Association`, `Descriptor`, and `Common` have a meaningful `documentation` string?
- [ ] Are descriptor examples clear and realistic? (e.g., "Examples include: Home, Hospital, School" — not "Examples include: Example1, Example2")
- [ ] For date fields, is the standard date interpretation disclaimer present? (Check existing entities like `StudentSchoolAssociation` for the exact wording.)
- [ ] Are descriptions broad enough? Avoid language that unnecessarily narrows the intended use (e.g., "in becoming an educator" vs. "in achieving an educational goal").

### 4. Key / Identity Ordering
- [ ] In entities that involve a `Student`, does the Student key appear first in the natural key, before `EducationOrganization` or other references? (Drives handbook ordering.)
- [ ] Is `is identity` applied to elements that form the natural key where required?
- [ ] Is `required` vs. `optional` correct for each property? New identity/key fields are almost always `required`.

### 5. Naming Conventions
- [ ] Entity names are singular (MetaEd pluralizes at the API layer). Do not name an entity `StaffDemographics` — use `StaffDemographic`.
- [ ] Do not append "Information" to entity names unless there is a strong reason — the community prefers shorter names (e.g., `StudentDemographic` not `StudentDemographicInformation`).
- [ ] Descriptor file names match the descriptor name exactly.

### 6. Deprecation Discipline
- [ ] If replacing an existing field with a new entity/field, are the old fields marked `deprecated` rather than deleted outright? Prefer deprecation → future removal cycle.
- [ ] New entities should not ship with deprecated elements already in them ("deprecated out of the box").
- [ ] If parallel elements exist (old and new), are the old ones marked `deprecated` in their original location?

### 7. Domain Files
- [ ] Are all new `DomainEntity`, `Association`, and `Descriptor` types listed in the appropriate `.metaed` domain file?
- [ ] Within a subdomain list, are entries in alphabetical order (consistent with how other domains are maintained)?
- [ ] Does the subdomain `position` value make sense relative to other subdomains?

### 8. Shared Strings / Common Types
- [ ] If a `shared string` already exists for a property (e.g., `MiddleName`, `FirstName`), use it rather than redefining inline.
- [ ] If adding a collection, use `optional collection` (not just `optional`) when multiple items are expected.

### 9. Slowly Changing Dimensions — BLOCKING
- [ ] The Ed-Fi Data Standard does not track slowly changing dimensions. Do not add `BeginDate`/`EndDate` pairs to demographic or identity entities unless there is a clear calendar-based reason (e.g., enrollment, employment period). Flag any such dates for team discussion.
- [ ] If dates are added to a natural key, verify that the entity can still be queried and updated as intended.

### 10. Deprecated MetaEd Keywords
- [ ] Do not use `queryable` — it is deprecated and will be removed in a future MetaEd release.
- [ ] Do not include trailing numbers on property/association names (e.g., `ScoreResult1`) — these were deprecated and are now ignored.

### 11. Interchange Files
- [ ] If new `DomainEntity` or `Association` types were added, are they referenced in the appropriate `Interchange` file(s)?
- [ ] If new `Descriptor` types were added, are they included in the Descriptor interchange?

### 12. PR Hygiene
- [ ] PR title should follow the pattern `DATASTD-XXXX Short description` — this becomes the default squash commit message.
- [ ] Run spell-check (VS Code `cSpell` extension) before submitting. Flag obvious spelling issues found in the diff.
- [ ] Sample data: if an `Association` or `DomainEntity` was deleted, confirm there is no orphaned sample data referencing it.

### 13. GitHub Actions Workflows (if changed)
- [ ] Branch filter patterns are consistent with other workflows (e.g., `DS-*` not `DS*`).
- [ ] Avoid hard-coding branch names — use the current branch reference where possible.
- [ ] Environment variable names match their definitions (e.g., `ARTIFACTS_API_KEY` vs. `AZURE_ARTIFACT_NUGET_KEY` mismatch).
- [ ] Remove conditions that are always true or steps that are always skipped.
- [ ] Reusable workflow `uses:` references: pinning to SHA is best practice for supply chain safety (though this repo intentionally uses `@main` for internal workflows — note the intentional exception in comments).

---

## Output Format

```
## Review: PR #<number> — <title>

### Blocking Issues
- **[Category]** `path/to/file.metaed`: <description>

### Suggestions
- **[Category]** `path/to/file.metaed`: <description>

### Looks Good
- <items that were done well or checked and found clean>
```

If there are no issues in a category, skip it. Keep comments specific and actionable.
