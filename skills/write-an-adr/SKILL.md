---
name: write-an-adr
description: Use when writing, amending, or superseding an architecture decision record (ADR), or when a design choice, behavior change, or rejected alternative needs its reasoning recorded for future engineers and agents. Triggers on "write an ADR", "record this decision", "document why we chose X", or a change that is expensive to reverse.
---

# Write an ADR

## Overview

An ADR records *why* a decision was made: the context, the constraints, the
alternatives that were rejected, and what the decision costs. Code shows what
was built. The ADR stops the next engineer, or the next agent, from re-deciding
the question or quietly undoing the decision.

**Core principle:** every claim in an ADR must be traceable to code, a ticket, a
standard, or observed behavior. A polished ADR that misstates a status code or a
rule order is worse than no ADR.

## When to Use

- A decision that would be expensive to reverse: data model, wire format,
  caching, concurrency, authentication, or a major dependency
- Client or user-visible behavior that changes
- A security or disclosure contract that a later "fix" could silently undo
- A defect class that recurred in review, where the lesson must outlive the
  patch
- Choosing between competing approaches where the losing options looked
  reasonable

**When NOT to use:** routine refactors, bug fixes that restore documented
behavior, or decisions that are obvious from the code. For product requirements
use a PRD; for an implementation plan use a spec or task breakdown.

## Workflow

1. **Confirm an ADR is warranted.** Check the criteria above. If the user only
   needs a code comment or a changelog line, say so.
2. **Build a source inventory.** Gather what is available and note what is not:
   - the Jira ticket, including acceptance criteria and comments
   - GitHub Issues, including comments and linked pull requests
   - the branch diff or PR, the implementation, and its tests
   - the relevant RFCs or Ed-Fi specifications
   - related ADRs in `reference/`, and design docs in `docs/` or
     `reference/design/`

   Do not state facts from a source you could not read.
3. **Check for related ADRs.** If this decision amends or replaces an earlier
   one, follow the lifecycle rules below before writing a parallel ADR.
4. **Ask for what you cannot find.** Usually: the Jira key, the status, which
   service the decision applies to, the author, and the deciders. Ask in one
   message, not one question at a time.
5. **Assign the number.** Follow [Numbering](#numbering).
6. **Draft from the template.** Include only the optional sections the decision
   needs.
7. **Verify against the code.** Re-check every status code, header name, error
   string, rule order, limit, and test name against the implementation. Make
   sure each test in a `## Tests` section exists and covers what the ADR says it
   covers.
8. **Review the prose.** See [Prose review](#prose-review).
9. **Stop before committing.** Show the user the file path and a summary. Commit
   only when asked.

## ADR template

````markdown
# ADR-NNN: <The decision, stated as an outcome>

**Status:** Proposed | Accepted | Superseded by ADR-XXX | Deprecated \
**Date:** YYYY-MM-DD \
**Author:** <Name>, with drafting assistance from Claude Code.

## Executive summary

<What happened before the decision, and who it hurt. Name the concrete symptom: the status code,
the header, the client that broke.>

<What the decision is. Use a short numbered list when it has several rules. Link to the sections
that hold the detail, such as [Known limitations](#known-limitations).>

## Context

### <Standard or constraint>

<The RFC section, specification, or upstream contract that constrains the decision.>

## Decision

### <First rule>

<One subsection per rule or contract. State the rule, then the reason for it.>

### <Second rule>

## Alternatives considered

| Alternative | Outcome | Reason |
| --- | --- | --- |
| <Option> | Rejected | <Why, in one sentence.> |
| <Option> | **Chosen** | <Why, in one sentence.> |

## Consequences

- <Before and after for each client-visible change: "X used to get `400`. It now gets `401`.">
- <What is given up, and what would have to change to get it back.>
- <What is unaffected.>

## Known limitations

<Gaps you accept, with the reason each is accepted. Link follow-up tickets.>

## Tests

- `<TestClass>` (<project or layer>) covers <the rules it pins>.

## References

- [RFC NNNN §N.N](https://www.rfc-editor.org/rfc/rfcNNNN#section-N.N) — <what it establishes>
- [ADR-NNN](./adr-NNN-<related>.md) — <how it relates>
````

### Header block

- **Status** is (Proposed | Accepted | Superseded by ADR-XXX | Deprecated)
- **Date** is ISO 8601.
- **Deciders** is included when a group signed off: `Development team (signed
  off 2026-07-08).`
- **Author** names the person, never a guess. Take the name from the user or
  `git config user.name`. The line always discloses AI help, naming the tool and
  its role: `with drafting assistance from Claude Code` or `with analysis
  assistance from Claude Code`.

### Optional sections

Add these only when the decision calls for them:

| Section | Include when |
| --- | --- |
| `## Decision drivers` | Several options were viable and the criteria for choosing need to be explicit. |
| `## Considered options` | The trade-offs are too deep for a table row. Use `### Option N — <name>` with **Pros:** and **Cons:** bullets, and mark the winner `(CHOSEN)`. Replaces `## Alternatives considered`. |
| `## Scope boundaries` | It is not obvious which services, endpoints, or backends are covered. |
| `## Known deviations and open items` | Related pre-existing defects were deliberately left out of scope. Link each Jira ticket. |
| `## Open question` | Something must be confirmed before the decision is fully safe. Say what, and what happens if the answer is no. |
| `## Defects found and their lessons` | Review found the same class of mistake more than once. Number each defect and end it with a bold **Lesson:**. |

Skip `## Tests` when the ADR describes architecture rather than a code contract.
In that case, say near the top that code-level details are left out and that the
implementation is the source of truth for them.

## Writing conventions

- **Be concrete.** For example, when describing an HTTP interaction, name the
  status code, header, error `title`, configuration key, and limit: `400
  Malformed Authorization header`, not "an error".
- **Use backticks** for identifiers, paths, headers, query parameters, and
  literal values.
- **Write consequences as before and after.** Name who is affected, and say what
  stays the same.
- **Link precisely.** Link RFCs to the section. Pin source links to a
  commit SHA and line range. Link Jira keys as
  `[DMS-NNNN](https://edfi.atlassian.net/browse/DMS-NNNN)`. Use relative links
  between ADRs and to `docs/`.
- **Use GitHub alerts sparingly**, such as `> [!NOTE]`, to record who confirmed
  a fact and how.
- **Do not invent** rationale, deciders, sign-off dates, or test coverage. If
  something is unknown, leave a visible `TODO:` for the user to resolve, or ask.

## ADR lifecycle

- **Amend in place** when the decision stands but a detail changes before
  merging to `main`. Let git history record the evolution of the ADR rather than
  creating a new one or describing a change as an amendment.
- **Supersede** when the decision itself is reversed or changed after merging to
  `main`. Write a new ADR, with the next number, that links to the old one and
  explains what changed. Change the old ADR's status to `Superseded by
  [ADR-NNN](./adr-NNN-<new>.md).` Leave the rest of it as written, number
  included.
- **Never delete an ADR.** The rejected reasoning is part of the record.

## Prose review

Clearer wording must not change the decision or its limits.

1. Mark the details that must survive editing: names, numbers and units, status
   codes, paths, links, rule order, conditions, trade-offs, and stated
   uncertainty. Check them against the code and sources. Flag anything you
   cannot substantiate.
2. Replace stock openings, promotional adjectives, and repeated summaries with
   the information the reader needs. Do not replace an unsupported claim with an
   invented example or a stronger guarantee. Keep the project's established
   terms (DMS, CMS, ODS/API, `ContentVersion`).
3. Compare the edited draft with its sources. Restore any qualifier or caveat
   that was lost. If a factual question is still open, say so rather than
   polishing it away.

## Common rationalizations

| Rationalization | Reality |
| --- | --- |
| "The PR description already explains it" | PR descriptions are hard to find later and are never updated. The ADR lives next to the code. |
| "The code is self-documenting" | Code shows what. It does not show which alternatives were rejected or why. |
| "I'll add the ODS/API comparison later" | The comparison is usually the reason the decision exists. |
| "The tests section can be approximate" | A test named in an ADR that does not cover the rule misleads every later reader. |
| "ADRs are overhead" | `adr-oauth-upstream-error-disclosure.md` exists because, in one review cycle, each fix reintroduced the class of defect it replaced. |

## Red flags

- A title phrased as a question or a topic instead of a decision
- A number that repeats, skips ahead, restarts at `001`, or differs between
  filename and title
- A `docs/adr/` folder in this repository
- An author line that is guessed or that omits AI assistance
- Consequences that do not say what changed for clients
- Alternatives listed without the reason they lost
- A status code, rule order, or test name not checked against the code
- An ADR that silently contradicts an earlier one instead of amending or
  superseding it

## Verification

- [ ] The file is in `reference/` and named `adr-<NNN>-<topic>-<JIRA-KEY>.md`
- [ ] The number is one more than the highest on both the working tree and
  `origin/main`
- [ ] The `jira:` frontmatter matches the filename suffix
- [ ] The title is `ADR-<NNN>:`, with the filename's number, and states the
  decision as an outcome
- [ ] Status gives acceptance, implementation state, and scope
- [ ] The Author line names a real person and discloses AI assistance
- [ ] The executive summary covers both the problem and the decision
- [ ] Each rejected alternative has a reason
- [ ] Consequences give before and after for every client-visible change
- [ ] Every status code, identifier, limit, and test name was checked against
  the code
- [ ] Related ADRs are linked, and amended or superseded ADRs are updated
- [ ] Prose edits kept every qualifier, trade-off, and open question
- [ ] The user has reviewed the draft before it is committed
