---
name: write-a-prd
description: Use when writing, editing, extending, or reverse-engineering product requirements documents (PRDs), including PRDs informed by Jira issues, web docs, existing PRDs, user interviews, or codebase behavior.
---

# Write a PRD

## Overview

Create PRDs by building a traceable product understanding before drafting. A good PRD explains the product context, users, jobs to be done, scope, requirements, architecture implications, non-functional requirements, open questions, and explicit exclusions.

**Core principle:** Do not turn a thin prompt into a polished-looking PRD. First classify the PRD situation, inventory sources, interview for gaps, and keep factual claims traceable.

## When to Use

Use this skill for:

- Writing a brand new PRD for a new application or product area.
- Editing an existing PRD without rewriting still-valid content.
- Creating a new PRD that builds on an existing PRD for a later release.
- Writing a PRD for an existing application by reverse engineering current functionality.
- PRDs that depend on mixed inputs: prompt text, user answers, Jira issues, web pages, product docs, existing PRDs, or source code.

Do not use this for a one-paragraph feature request, implementation plan, design spec, or ticket unless the user wants PRD-level product requirements.

## Start by Classifying the Mode

| Mode | Trigger | Default approach |
| --- | --- | --- |
| New application | No prior PRD or app exists | Interview heavily; define product vision, users, workflows, requirements, architecture assumptions, and launch boundaries. |
| Edit existing PRD | User asks to modify an existing PRD | Preserve structure, terminology, and valid content; make targeted changes with a change summary. |
| Next release PRD | User wants a new PRD based on an earlier one | Copy the useful structure; distinguish carried-forward requirements, changed requirements, new scope, and removed/deferred items. |
| Existing app PRD | App exists but PRD is absent/stale | Reverse engineer current behavior first; separate observed current state from proposed future requirements. |

If the mode is unclear, ask one question to classify it before drafting.

## Required Workflow

1. **Build a source inventory.** List available and missing sources: initial prompt, user answers, existing PRDs, Jira work items, web pages, docs, codebase, tests, analytics, support notes, and release notes. Use Jira/web/document tools when sources are named or accessible. Do not claim information from inaccessible sources.
2. **Extract current truth.** Summarize what each source establishes. For existing applications, inspect routes, UI flows, API contracts, data models, tests, docs, and deployment/configuration to identify shipped behavior.
3. **Interview for gaps.** Ask focused questions one at a time. Prefer multiple choice for scope, release goals, personas, must/should/could priorities, constraints, and out-of-scope boundaries. Do not ask for information already available in sources.
4. **Propose PRD shape.** Before writing a large PRD, state the intended sections, source assumptions, and any unresolved questions. Get approval when the structure or scope is significant.
5. **Draft or edit.** Use the selected mode's rules below. Preserve the user's requested destination: existing file, new file, issue, or chat draft.
6. **Self-review.** Check for unsupported claims, contradictions, missing NFRs, vague requirements, absent out-of-scope items, and unresolved open questions.

## PRD Structure

Use the existing organization's PRD structure when one exists. Otherwise, use this structure, adapted to the product:

1. **Title and metadata**: status, owner, product/repository, related Jira project or issue list, source references.
2. **1. Product overview**: what the product is, who it serves, and why it exists.
3. **1.1 Strategic alignment**: business goals, market drivers, policy/regulatory drivers, and release objectives.
4. **1.2 Target users and personas**: motivations, technical depth, pain points, and success criteria.
5. **1.3 Jobs to be done / user journeys**: "When / I want / so that" statements plus variations.
6. **2. Enterprise or system context**: external systems, identity, data stores, integrations, deployment boundaries, and diagrams when useful.
7. **3. Functional requirements**: stable IDs, clear SHALL/SHOULD/MAY language, grouped by capability. DO NOT USE TABLES, USE BULLETED LISTS.
8. **4. Non-functional requirements**: compatibility, security, privacy, accessibility, observability, reliability, performance, operations, SDLC, compliance. DO NOT USE TABLES, USE BULLETED LISTS.
9. **5. System architecture**: component table, runtime/deployment targets, data ownership, integration points. USE A TABLE FOR ARCHITECTURE COMPONENTS.
10. **6. Out of scope and known limitations**: explicit exclusions, deferred work, assumptions, risks.
12. **7. Open Questions and Decision Log**: unresolved product decisions and chosen trade-offs.
11. **8. Glossary**: product-specific terms, acronyms, and overloaded words.

## Mode-Specific Rules

### New Application PRD

- Do not draft from a one-sentence idea. Ask for problem, users, alternatives, business goals, launch constraints, and success metrics.
- Define MVP and non-MVP explicitly.
- Include implementation-neutral product requirements first; add architecture implications only when they constrain product scope.
- Identify risky assumptions and validation needs.

### Editing an Existing PRD

- Read the existing PRD before proposing changes.
- Preserve valid text, section order, terminology, requirement IDs, and tone unless they are wrong or the user asks for a rewrite.
- For each change, determine whether it adds, modifies, removes, or clarifies scope.
- If adding inferred NFRs from code or docs, label them as inferred until confirmed.
- End with a concise change summary.

### Next Release PRD from a Prior PRD

- Treat the prior PRD as the baseline, not as automatically current truth.
- Add version/release metadata and describe the delta from the previous release.
- Categorize requirements as carried forward, revised, new, deprecated, or deferred.
- Keep historical context only when it still explains current product decisions.
- Avoid duplicating stale implementation details that may have changed.

### Existing Application PRD by Reverse Engineering

- Create a feature inventory from observed behavior before writing future requirements.
- Inspect docs, routes/screens, API endpoints, data models, tests, configuration, permissions, integrations, and deployment files.
- Separate **Current Functionality** from **Proposed Requirements**. Do not turn accidental implementation details into product promises.
- Cite or note the source for important claims: file, route, test, Jira issue, doc title, or URL.
- Ask the user to confirm ambiguous behavior, especially permissions, edge cases, and business rules.

## Source Handling

| Source | How to use it | Caution |
| --- | --- | --- |
| Prompt text | Treat as initial intent and constraints | Do not assume omitted details. |
| User answers | Treat as authoritative product direction | Capture decisions and open questions. |
| Jira issues | Extract acceptance criteria, scope, status, and links | Check whether issues are proposed, in progress, done, or obsolete. |
| Existing PRDs | Reuse structure and stable context | Verify whether old requirements are still true. |
| Web pages/docs | Extract externally documented promises and terminology | Record title/URL; don't invent inaccessible content. |
| Code/tests | Infer shipped behavior and technical constraints | Distinguish observed behavior from intended product requirements. |

## Requirement Quality

Good requirements are testable, scoped, and actor-aware:

- Use `SHALL` for mandatory release requirements.
- Use `SHOULD` for intended but negotiable requirements.
- Use `MAY` for optional behavior.
- Give stable IDs by capability, such as `FR-AUTH-1` or `NFR-SEC-1`.
- Prefer product behavior over implementation detail.
- Include negative and failure scenarios where product behavior matters.
- Add out-of-scope items for tempting adjacent features.

## Example Working Notes

Before drafting, create a compact working map like this:

```markdown
Mode: Next release PRD from existing PRD
Primary sources:
- Prior PRD: docs/PRD-v1.md
- Jira: AC-101, AC-102, AC-103
- Code observation: billing roles and invoice export routes

Current-state claims:
- Users can export invoices as CSV. Source: route/test names
- Admin role can manage plans. Source: existing docs and UI labels

Open product decisions:
- Should invoice exports include tax jurisdiction fields?
- Is plan deletion allowed, or only archive?
```

Then draft the PRD using the approved section structure.

## Common Mistakes

| Mistake | Why it fails | Fix |
| --- | --- | --- |
| Drafting immediately from a thin prompt | Produces plausible fiction | Inventory sources and interview first. |
| Rewriting an existing PRD wholesale | Loses history and stable decisions | Preserve what remains true; make targeted edits. |
| Treating old PRDs as current truth | Carries stale requirements forward | Verify against code, Jira status, and user confirmation. |
| Mixing current and future behavior | Confuses implementation reality with product intent | Use separate current-state and proposed-requirements sections. |
| Hiding uncertainty | Readers mistake guesses for decisions | Add open questions, assumptions, and source notes. |
| Omitting NFRs | Product may be unbuildable or unshippable | Review security, reliability, observability, compatibility, accessibility, operations, and SDLC. |
| Over-specifying file paths/code | PRD becomes brittle | Keep product requirements stable; put implementation specifics in design docs or plans. |
| Ignoring edge cases | Product may fail in uncommon scenarios | Include edge cases in requirements and test plans. |
| Excessive implementation detail | PRD becomes tightly coupled to current code | Focus on product behavior and requirements; move implementation specifics to design docs. |

## Final Self-Review

Before presenting the PRD as complete, verify:

- The mode is clear.
- Sources used and sources missing are visible.
- Personas, JTBD/user journeys, functional requirements, NFRs, architecture context, out-of-scope items, and glossary are covered or intentionally omitted.
- Requirements are testable and use consistent SHALL/SHOULD/MAY language.
- Existing PRD edits preserve still-valid content.
- Reverse-engineered claims are distinguishable from proposed future requirements.
- Open questions are explicit instead of hidden as assumptions.
