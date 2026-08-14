---
description: Launches a fleet of five specialist subagents to deeply review a pull request across security, functionality, maintainability, usability, and test coverage. Provide the PR number and optional related Issue number to get a severity-ranked summary of all findings.
name: team-pr-review
allowed-tools: Bash(gh *), Bash(git *), Read, Write, Glob, Grep
---

# Team PR Review

## Overview

Five specialist reviewers examine a pull request in parallel, each through a different lens. Every finding is rated by severity. The main thread aggregates all findings into a single severity-ranked report that the team can act on immediately.

**Trigger phrases:**

- "Review PR #42 against issue #17"
- "Run team-pr-review on PR #..."
- "Do a full review of this PR"

## When to Use

- Before merging any non-trivial PR
- When you want more than a single-axis review
- When a PR touches security-sensitive code, user-facing flows, or core business logic
- When you want a consistent, repeatable review standard across all team PRs

## Required Inputs

| Input                       | How to Provide                                                                     |
| --------------------------- | ---------------------------------------------------------------------------------- |
| **PR number**               | In the user's message: "PR #42"                                                    |
| **Issue number (optional)** | In the user's message: "Issue #17" — the requirement the PR is supposed to fulfill |

If PR number is missing, ask for it before proceeding. Do not guess.
If issue number is missing, continue in PR-only mode and note reduced confidence for requirement coverage.

## Severity Rubric

All five specialists use this shared rubric. Consistency matters more than individual judgment — anchor every rating to this table.

| Rating       | Meaning                                        | Example                                                                                 |
| ------------ | ---------------------------------------------- | --------------------------------------------------------------------------------------- |
| **Critical** | Blocks merge. Immediate fix required.          | Security vulnerability, data loss, auth bypass, broken core flow                        |
| **High**     | Significant problem. Should fix before merge.  | Major requirement not met, missing auth check, crash on likely input                    |
| **Medium**   | Real issue, but not merge-blocking if tracked. | Code smell that will compound, partial requirement coverage, test gap on important path |
| **Low**      | Minor issue. Fix if cheap, defer if not.       | Suboptimal approach with low risk, missing test for edge case                           |
| **Nit**      | Inconsequential. Author may ignore.            | Naming preference, style inconsistency, trivial suggestion                              |

## Process

### Tooling Constraints

- Use only `gh` and `git` CLI commands that are listed in this skill, plus the `Read`, `Write`, `Glob`, and `Grep` tools.
- The `Write` tool is used only to save the final report to a temporary local file before posting (Step 4); do not use it to modify repository files.
- Do not use tools outside the approved set unless the user explicitly asks.
- If a required command is unavailable, stop and report the missing prerequisite.

### Step 0: Preflight Checks

Before gathering review context, verify GitHub CLI access and PR visibility:

1. **GitHub CLI authentication**

```
gh auth status
```

2. **PR accessibility**

```
gh pr view {{PR_NUMBER}} --json title
```

If either check fails, stop and report the setup issue clearly.

### Step 1: Gather Context

Before launching any subagents, the main thread fetches:

1. **PR details** — title, description, diff, changed files:

   ```
   gh pr view {{PR_NUMBER}} --json title,body,files,commits
   gh pr diff {{PR_NUMBER}}
   ```

2. **Issue details (optional)** — title and body first; fetch comments only if needed for missing or ambiguous acceptance criteria:

```
gh issue view {{ISSUE_NUMBER}} --json title,body
```

Optional follow-up (only when needed):

```
gh issue view {{ISSUE_NUMBER}} --json comments
```

If no issue number is provided, skip this step and proceed in PR-only mode.

3. **Worktree checkout** — check out the PR in an isolated git worktree so specialists can read the full codebase in context without changing the current branch:
   ```
   git fetch origin pull/{{PR_NUMBER}}/head
   git worktree add --detach ../pr-{{PR_NUMBER}} FETCH_HEAD
   ```
   If worktree creation fails, proceed with the diff alone and note the limitation.

Store the PR diff, PR description, and issue body (if available) as variables to inject into each specialist's context.

### Step 2: Launch Five Specialist Agents in Parallel

Launch all five agents simultaneously. Do not wait for one before starting the next.

Each agent receives:

- The full PR diff
- The list of changed files
- The issue title and body (acceptance criteria), when available
- Their specialist prompt (see below)
- Access to the full worktree, when available

If no issue is provided, instruct all specialists to infer intent from PR title/body and explicitly state reduced confidence where requirements are ambiguous.

#### Specialist 1: Security Reviewer

**Goal:** Find vulnerabilities introduced or exposed by this PR.

**Prompt to use:**

```
You are a security-focused code reviewer. Your job is to find security vulnerabilities
in this pull request.

ISSUE CONTEXT:
{{ISSUE_BODY}}

PR DESCRIPTION:
{{PR_DESCRIPTION}}

PR DIFF:
{{PR_DIFF}}

Review the changes for:
- OWASP Top 10 vulnerabilities (injection, broken auth, sensitive data exposure,
  XXE, broken access control, security misconfiguration, XSS, insecure deserialization,
  vulnerable components, insufficient logging)
- Authentication and authorization: are the right checks in place? Can a user
  access or modify data they shouldn't?
- Input validation: is all user-supplied data validated before use?
- Secrets and credentials: are any secrets, tokens, or PII exposed in code or logs?
- Dependency risk: do any new dependencies introduce known vulnerabilities?
- Error handling: do error messages reveal internal details to users?
- Data at rest and in transit: is sensitive data handled safely?

For each finding, output:
**[SEVERITY]** — one-line summary
Detail: [what the issue is, where in the code, why it matters]
Fix: [concrete suggestion]

Use the severity rubric: Critical / High / Medium / Low / Nit

End with a one-line overall security assessment.
```

#### Specialist 2: Functionality Reviewer

**Goal:** Verify the PR fulfills the requirements in the linked issue.

**Prompt to use:**

```
You are a functionality-focused code reviewer. Your job is to verify that this
pull request correctly and completely implements the requirements in the linked issue.

ISSUE CONTEXT (these are the requirements):
{{ISSUE_BODY}}

PR DESCRIPTION:
{{PR_DESCRIPTION}}

PR DIFF:
{{PR_DIFF}}

Review the changes for:
- Requirements coverage: does the PR implement everything described in the issue?
  Flag any requirements that appear partially or not implemented.
- Correctness: does the implementation actually do what it claims?
- Edge cases: are boundary values, empty inputs, and unexpected states handled?
- Happy path and unhappy path: does error handling preserve the intent of the feature?
- Regression risk: could this change break existing functionality? Look for side effects
  in shared utilities or data models.
- Consistency: does the behavior match what a user reading the issue would expect?

For each finding, output:
**[SEVERITY]** — one-line summary
Detail: [what the issue is, where in the code, why it matters]
Fix: [concrete suggestion]

Use the severity rubric: Critical / High / Medium / Low / Nit

End with a verdict: COMPLETE / PARTIAL / INCOMPLETE — followed by one sentence of justification.
```

#### Specialist 3: Maintainability Reviewer

**Goal:** Identify code that will be painful to change, debug, or extend.

**Prompt to use:**

```
You are a maintainability-focused code reviewer. Your job is to find code that
will be difficult to maintain, extend, or debug over time.

ISSUE CONTEXT:
{{ISSUE_BODY}}

PR DIFF:
{{PR_DIFF}}

Review the changes for:
- Duplication: is non-trivial logic repeated? Would a future change require
  editing it in multiple places?
- Code smells: long functions doing too many things, deep nesting, magic numbers,
  misleading names, implicit coupling between modules
- Guard clauses: are there opportunities to fail fast and reduce nesting depth?
- Logging: are significant operations, errors, and state transitions logged
  at appropriate levels? Are log messages useful for debugging in production?
- Naming: do names communicate intent clearly, consistently with project conventions?
- Dead code: are there unused variables, unreachable branches, or commented-out code?
- Abstraction level: is the code pitched at the right level of abstraction?
  Avoid both over-engineering (premature abstraction) and under-engineering (no structure).
- Module boundaries: does the change respect existing layering rules and separation of concerns?

For each finding, output:
**[SEVERITY]** — one-line summary
Detail: [what the issue is, where in the code, why it matters]
Fix: [concrete suggestion]

Use the severity rubric: Critical / High / Medium / Low / Nit

End with a one-line overall maintainability assessment.
```

#### Specialist 4: Usability Reviewer

**Goal:** Evaluate the experience from the user's point of view.

**Prompt to use:**

```
You are a usability-focused code reviewer. Your job is to evaluate this pull request
from the perspective of the end user — the person who will interact with the software,
not the developer who built it.

ISSUE CONTEXT (what the user needs):
{{ISSUE_BODY}}

PR DESCRIPTION:
{{PR_DESCRIPTION}}

PR DIFF:
{{PR_DIFF}}

Review the changes for:
- Error surfaces: when something goes wrong, does the user receive a clear,
  actionable message? Or are they shown a generic error, a stack trace, or silence?
- Recovery paths: after an error or failure, can the user return to their goal?
  Is there a clear next step?
- Feedback and affordance: does the user receive appropriate confirmation
  when actions succeed? Is it clear what the system is doing?
- Consistency: does the interaction pattern match what users expect from the
  rest of the product?
- Accessibility: are there obvious a11y concerns (missing labels, keyboard traps,
  color-only information)?
- Edge cases from the user's view: what happens when the user submits an empty form,
  double-clicks, or uses the feature on a slow connection?
- Tone: are error and status messages written for humans, not engineers?

For each finding, output:
**[SEVERITY]** — one-line summary
Detail: [what the issue is, the user impact, why it matters]
Fix: [concrete suggestion]

Use the severity rubric: Critical / High / Medium / Low / Nit

End with a one-line overall usability assessment.
```

#### Specialist 5: Test Coverage Reviewer

**Goal:** Determine whether the tests adequately protect the changed behavior.

**Prompt to use:**

```
You are a test coverage-focused code reviewer. Your job is to evaluate whether
the tests accompanying this pull request adequately cover the new and changed behavior.

ISSUE CONTEXT:
{{ISSUE_BODY}}

PR DIFF:
{{PR_DIFF}}

Review the changes for:
- Coverage of the happy path: is the primary success scenario tested?
- Coverage of unhappy paths: are error conditions, invalid inputs, and
  failure modes tested?
- Edge cases: are boundary values (empty collections, zero, null, max values)
  tested where they matter?
- Test quality: do tests verify behavior (observable outcomes), or do they
  just verify implementation details (internal state, private methods)?
- Test names: do test descriptions communicate what the test proves?
  A good test name reads like a specification.
- Test isolation: do tests have unintended dependencies on each other or on
  external state?
- Regression risk: are there changed behaviors that lack a corresponding
  test update?
- Test-to-code ratio: is the amount of test code proportionate to the risk
  and complexity of the change?

For each finding, output:
**[SEVERITY]** — one-line summary
Detail: [what the issue is, which behavior is unprotected, why it matters]
Fix: [concrete suggestion — what test to add or change]

Use the severity rubric: Critical / High / Medium / Low / Nit

End with a coverage verdict: STRONG / ADEQUATE / WEAK — followed by one sentence of justification.
```

### Step 3: Aggregate and Summarize

Once all five agents have responded, the main thread produces a consolidated report.

**Format the report as follows:**

```markdown
# PR #{{PR_NUMBER}} Review — {{PR_TITLE}}

> Issue: {{ISSUE_REFERENCE_OR_NONE}}
> Reviewed by: {{LLM_MODEL}}
> Specialist agents: Security · Functionality · Maintainability · Usability · Test Coverage

## 🔴 Critical

[List all Critical findings from all reviewers, with reviewer label]

- **[Security]** Auth bypass in volunteer endpoint — missing role check on DELETE handler
  Fix: Add `requireRole('admin')` middleware before the handler

[Repeat for each Critical finding. If none: "None found."]

## 🟠 High

[All High findings, same format]

## 🟡 Medium

[All Medium findings]

## 🔵 Low

[All Low findings]

## ⚪ Nit

[All Nit findings, collapsed or summarized if numerous]

## Specialist Verdicts

| Reviewer        | Verdict                                                            |
| --------------- | ------------------------------------------------------------------ |
| Security        | {{SECURITY_ASSESSMENT}}                                            |
| Functionality   | COMPLETE / PARTIAL / INCOMPLETE — {{FUNCTIONALITY_VERDICT_REASON}} |
| Maintainability | {{MAINTAINABILITY_ASSESSMENT}}                                     |
| Usability       | {{USABILITY_ASSESSMENT}}                                           |
| Test Coverage   | STRONG / ADEQUATE / WEAK — {{TEST_COVERAGE_VERDICT_REASON}}        |

## Summary

{{SUMMARY_2_TO_3_SENTENCES}}
```

**Ordering rules within each severity tier:**

1. Security findings first
2. Functionality findings second
3. Maintainability, Usability, Test Coverage in any order

If a tier has no findings, print a single "None found." line. Do not skip the tier header.

### Step 4: Offer to Post

After displaying the report, ask the user:

> Would you like me to post this summary as a comment on PR #N?
> Options:
>
> - **Post as-is** — post immediately
> - **Edit first** — you can adjust the summary, then I'll post it
> - **Skip** — keep the report local only

If the user chooses to post, use the `Write` tool to save the report markdown to a temporary file (e.g. `{{TEMP_FILE}}` = `./.pr-review-{{PR_NUMBER}}.md` in the worktree or current working directory) to avoid line ending and quoting problems, then submit with:

```
gh pr comment {{PR_NUMBER}} -F {{TEMP_FILE}}
```

Delete the temporary file after posting.

If the user chooses to edit first, present the raw markdown and wait for their revised version before posting.

## Failure Handling

| Problem                                       | What to do                                                                                             |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Worktree checkout fails                       | Proceed with diff-only context; note the limitation in the report header                               |
| Issue not found                               | Ask the user to provide the issue body manually, or continue in PR-only mode (note reduced confidence) |
| A specialist agent fails or times out         | Note the failure in the relevant specialist's section; do not block the full report                    |
| PR has no diff (already merged, wrong number) | Stop and tell the user — do not fabricate a review                                                     |
| Model token not injected                      | Set `Reviewed by` to `Unknown model` and continue                                                      |

## Anti-Patterns to Avoid

- **Don't launch agents sequentially** — all five must run in parallel. The value of this skill is the parallel fleet.
- **Don't let severity inflation happen** — anchor every rating to the rubric. "I want to be thorough" is not a reason to call something Critical.
- **Don't omit the "None found" tiers** — a tier with no findings is signal, not noise. Show it.
- **Don't duplicate findings** — if Security and Functionality both flag the same issue, report it once with both reviewer labels.
- **Don't post without asking** — always offer the post option, never post automatically.

## Verification

After completing a review:

- [ ] All five specialists were launched and responded
- [ ] Every finding has a severity rating from the rubric
- [ ] The report groups findings by severity, not by reviewer
- [ ] Each finding has a concrete fix suggestion
- [ ] The Specialist Verdicts table is complete
- [ ] The user was offered the option to post the summary to the PR
