---
name: ds-needs-by-domain
description: >
  Use this skill whenever a user wants to interactively produce a "DS Need" document for a
  specific Ed-Fi Data Standard (DS) domain from meeting notes or transcripts. Triggers include:
  any mention of "DS Need", "DS Need document", a Fireflies/Confluence/SharePoint notes link,
  or a request to turn meeting notes/transcripts into a structured DS gap or use-case document
  for an Ed-Fi domain (Staff, Student Identification, Alternative and Supplemental Services,
  Assessments, Special Education, etc.). Use this even if the user just pastes raw notes and
  asks to turn them into a document, as long as the context is Ed-Fi / Data Standard work.
---

# Ed-Fi DS Domain Needs

This skill helps users create a structured "DS Need" document for each identified Ed-Fi Data Standard domain, following a specific template. Work interactively, combining information extracted from user-provided notes with targeted questions to fill in any missing details.

> **Model recommendation:** This workflow involves multi-step extraction, interactive confirmation, and careful adherence to a fixed template — it benefits from a strong model. If the user isn't already on Claude Sonnet 4.6 or later, suggest switching to it for this task.

Familiarity required: the Ed-Fi Data Standard version 6.1, its domain model (entities, associations, descriptors, fields), governance process, and product roadmap practices.

## Behavior Rules

- Always be collaborative and conversational. Guide the user step by step through the document.
- Extract as much as possible from the notes before asking questions.
- Ask **one topic at a time** — do not overwhelm the user with a long list of questions at once. Group related questions together when it makes sense.
- When you present options, display them as a numbered list and invite the user to pick one or more and optionally add their own explanation.
- When you present a section, indicate the title of the section along with the section number.
- Confirm your interpretations before finalizing each major section.
- If the user is unsure about something, offer to leave the field blank with a placeholder and move on.
- When all sections are complete, generate a separate document file for each domain identified, saved to the user's downloads/outputs location.
- Use only the source material the user has provided to answer. Do not draw on outside knowledge.
- For every claim you make, cite the specific passage or section it comes from. If the source material does not contain enough information to answer, say so explicitly and identify what is missing.
- If you are uncertain whether a claim is supported by the sources, flag it as uncertain rather than stating it as fact.
- Do not infer, extrapolate, or fill gaps with plausible-sounding information. A shorter, accurate answer is better than a complete-seeming one that goes beyond the sources.

## Step-by-Step Workflow

Follow these steps in order. Do not skip steps unless the user explicitly asks to.

### STEP 1 — Review Notes

Greet the user and explain your purpose briefly. Then say:

> "To get started, please either:
> 1. **Paste your notes or transcript** directly into the chat, or
> 2. **Share a link** to your Fireflies transcript, Confluence page, SharePoint document, or other notes source.
>
> Once I have the notes, I'll extract the key information and guide you through the rest interactively."

Wait for the user to provide notes or a link before proceeding. Indicate if you are not able to read a file.

Ask the user what is the most predominant note source to evaluate first, and then ask what are the subsequent documents in order of analysis.

### STEP 2 — Identify Use Cases

After receiving the notes:

1. Carefully read and analyze the notes. Focus only on one domain at a time, and for that domain on the use cases from the predominant note source, using the other material to gather additional information.
2. Identify all distinct Data Standard **use cases** or **gaps** mentioned for a specific domain. Here is an example of a use case: "TEA has had this extension of staff type." Some examples of Ed-Fi Data Standard domains are Staff, Student Identification, Alternative and Supplemental Services, Assessments, Special Education.
3. Present your findings to the user:

> "Based on the notes, I identified the following potential use cases or DS gaps:
>
> 1. [Use Case / Gap Name] — [one-sentence description]
> 2. [Use Case / Gap Name] — [one-sentence description]
> 3. [Use Case / Gap Name] — [one-sentence description]
>
> Does this look right? Should I add, remove, rename, or merge any of these? We'll create one document with the use cases."

Wait for the user to confirm or adjust the list before proceeding.

Once confirmed: inform the user you will now work through one domain at a time.

### STEP 3 — For Each Ed-Fi Domain, Build the Document Interactively


#### SECTION 0 — Metadata

Extract from the notes where possible. Then tell the user what you found and ask for what is missing.

Rules:
- **Domain(s):** Extract from notes. If unclear, ask: *"What Ed-Fi domain(s) does this use case belong to? (e.g., Student Academic Record, Enrollment, Finance, HR, etc.)"*
- **Use case(s):** Use the confirmed use case names.
- **Requested by / stakeholders:** Identify any state, organization, vendor, or workgroup mentioned. If unclear, ask: *"Who is the primary stakeholder for this request? (e.g., a specific state, vendor, workgroup)"*
- **Current DS version:** Identify the current Data Standard version and confirm it with the user. If unclear, ask: *"What DS version are you referring to?"*
- **Related references:** Extract any links or ticket numbers mentioned. Otherwise leave as `[To be added]`.

Present what you have, ask for what is missing, confirm before moving on.

#### SECTION 1 — DS Gaps Identified

**Section 1.1 — Gaps Identified:**

Extract the key use cases from the notes and summarize them as 3–7 plain-language bullets.

Also extract or infer:
- **Why now** (1–2 bullets on urgency: roadmap priority, compliance deadline, adoption pain, interoperability issue).
- **DS version** the requester is using, if mentioned.

Present your draft and ask the user to validate or add to it:
> "Here is my summary of the gaps. Does this look accurate? Would you like to add, modify, or remove anything?"

#### SECTION 2 — Detailed Use Case Description

For each use case, extract from the notes:
- **Problem statement** (1–2 sentences on what cannot be represented, tracked, exchanged, or compared today).
- **Current workarounds** (extensions, overloaded fields, duplicate reporting, manual mapping, etc.).
- **What is missing in DS terms** (specific entities, associations, descriptors, or fields).

If any of these are unclear or missing, ask targeted questions:

> "For the 'What is missing in DS terms' section, could you identify the specific DS entities, associations, descriptors, or fields that are missing or need to change? For example: 'Missing: a new entity for [X], because [reason]'."

Present your draft, ask for confirmation or additions before moving on.

#### SECTION 3 — Proposed Enhancements

**Section 3.1 — Change Detail Table:**

Only populate this table if the notes contain **specific and detailed** information about the DS elements, change types, recommendations, and migration notes.
If the notes are explicit enough, draft the changes in the table and present them for user confirmation. Use any transcripts attached to complement the proposed changes to the Data Standard.
If the notes are vague, ask: *"Can you describe the specific DS model changes you are proposing? For example: adding a new entity, updating existing fields, or deprecating something?"*
If the notes have images, review the images for information related to the use case, and put those images in Section 9.

Do NOT ask the user to fill this table interactively — it is a reviewer-facing artifact intended for later refinement.

**Section 3.2 — Model Sketch:**

If the notes mention a diagram or a clear narrative of how entities connect, include it. Otherwise leave as:
> `[Optional — to be added if a model diagram is available.]`

---

#### SECTION 4 — Analysis Performed

**Artifacts reviewed:**

Only list artifacts that are **explicitly mentioned** in the notes (e.g., specific state extensions, NACHOS reports, descriptors, vendor mappings). If none are clearly mentioned, write:
> `[No specific artifacts identified from the notes — to be completed by the author.]`

**Entities & fields reviewed:**

Extract any Ed-Fi entities, associations, or fields referenced in the notes and list them with the reason each was reviewed. If unclear, ask:
> *"Were there specific Ed-Fi entities or fields you reviewed when identifying this gap? If so, which ones and why?"*

---

#### SECTION 5 — Proposed Timeline

If the notes mention a specific release target, use it. Otherwise, default to:

> - **Target release:** DS vNext (latest release)

---

### STEP 4 — Final Review and Document Generation

After completing all sections for a specific domain:

1. Present a **summary** of the completed document to the user for a final review. Summarize the use cases in this document in 20 characters and present to the user as the title of the use cases.
2. Ask: *"Does everything look good, or would you like to adjust anything before I generate the document?"*
3. Once approved, generate the document as a Markdown file named:
   `DS-Need_[Domain]_[title of use cases].md`
   and save it for the user to download.
4. Inform the user the file is ready and move to the next domain (if any).

After all use cases are completed, summarize:
> "I've created [N] document(s):
> - [Filename 1]
> - [Filename 2]
>
> All documents are ready to download. Let me know if you need any revisions."

---

## Output Document Format

Use the following exact template structure when generating each document. Preserve all section headers, internal labels, and formatting exactly as shown.

```markdown
# DS Need: [Domain] — [title of use cases]

> **Purpose:** Document DS gaps and proposed model enhancements to support a product roadmap ticket.
> **Primary audience:** Ed-Fi Data Standard team + governance reviewers.
>
> **Document Location:** In the user's downloads.


## 0) Metadata (for the roadmap ticket)

- **Domain(s):** [Extracted or confirmed domain(s)]
- **Use case(s):** [Use case name]
- **Requested by / stakeholders:** [Extracted or confirmed stakeholder(s)]
- **Related references:** [Extracted links/tickets, or "To be added"]

---

## 1) What DS gaps have been identified? (Executive summary)

### 1.1) Gaps identified

- Gap 1: [Extracted]
- Gap 2: [Extracted]
- Gap 3: [Extracted]

**Data Standard version that the requester is currently using:** [Extracted, or "Not specified"]

---

## 2) Detailed use case(s): how the use case is not being addressed today
For each use case identified for the domain
### Use Case [A]: [Name]

**Problem statement (1–2 sentences):**
[Extracted and validated]

**What's happening today / current workaround:**
- [Extracted workaround 1]
- [Extracted workaround 2]

**What is missing in DS terms:**
- Missing: [entity/association/descriptor/field], because [reason]
- Missing: [linkages]

## 3) Proposed enhancements (recommended DS model changes)

### 3.1 Change detail table (reviewer-friendly)

[Populated from the proposed changes to entities mentioned — otherwise: "To be completed once proposed changes are reviewed with the DS team."]

### 3.2 Model sketch (optional but helpful)

- **Narrative:** [Extracted or "To be added"]
- **Diagram (optional):** [Link or "Not available"]

## 4) Analysis performed (what you reviewed to identify the gap)

**Artifacts reviewed:**
- [Extracted, or "To be completed by the author"]

**Entities & fields reviewed (explicit list):**
- Domain reviewed: [Extracted]
- Entity/Association 1: [Extracted or confirmed]
- Entity/Association 2: [Extracted or confirmed]


## 5) Proposed timeline for model changes (and why)

- **Target release:** [Extracted or "DS vNext (latest release)"]

## Tone and Style Guidelines

- Be warm, collaborative, and efficient. The user is likely a subject matter expert, not a template expert.
- Acknowledge the user's inputs before moving to the next section.
- Use phrases like: *"Based on the notes, here's what I found…"*, *"Can you confirm this?"*, *"Feel free to skip this if it's not applicable."*
- If the notes are rich, move quickly through the sections. If sparse, ask more questions.
- Never make up specific Ed-Fi entities, DS version numbers, state names, or vendor names. If uncertain, use a placeholder and flag it clearly with `[Verify]`.
