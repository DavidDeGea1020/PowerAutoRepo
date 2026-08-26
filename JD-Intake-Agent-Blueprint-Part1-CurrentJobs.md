# JD Intake Agent — Blueprint Part 1: Current Job Review Workflow
**Platform:** Copilot Studio (new agent experience, production-ready preview)
**Scope:** Ingestion of the daily job report + conversational review/update of an existing job description
**Out of scope for now:** New job creation workflow (Part 2), manager approval email (Part 3 — but the data model below is built to feed it)

---

## 0. The one architectural decision that matters most

**Do not make the daily report a Knowledge source.**

Knowledge in Copilot Studio is semantic retrieval — it chunks content, embeds it, and returns the *most relevant-looking passages*. That is exactly right for "what does our JD writing standard say about education requirements." It is exactly wrong for "give me all 14 fields for Job Code 40312, verbatim, with nothing invented and nothing dropped."

If you index the report as knowledge, you will get:
- Field bleed between similar job titles (Sales Rep II vs Sales Rep III returning blended text)
- Silently truncated Responsibilities fields
- No reliable way to write changes back
- No audit trail for your manager's approval step

**Instead:** land the report in a structured store, and expose it through a **workflow tool** with typed inputs and outputs. Deterministic lookup, deterministic write-back, full change history.

> Rule for the whole build: **Knowledge = guidance the agent reasons over. Tools = facts the agent retrieves.** Job data is facts.

---

## 1. Architecture at a glance

```
┌─────────────────────────────────────────────────────────────┐
│  LAYER 1 — INGESTION (autonomous, runs without the agent)   │
│                                                             │
│  Outlook daily report ──► Workflow: "JD-Ingest-DailyReport" │
│    trigger: When a new email arrives (V3)                   │
│    filter: from = <report sender>, subject contains <X>     │
│    steps: get attachment → parse rows → upsert SharePoint   │
└──────────────────────────┬──────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  LAYER 2 — DATA                                             │
│                                                             │
│  SP List: JobDescriptions_Current   (1 row per job code)    │
│  SP List: JD_ChangeLog              (1 row per field edit)  │
│  SP List: JD_ReviewSessions         (1 row per review)      │
└──────────────────────────┬──────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  LAYER 3 — TOOLS (agent-callable workflows)                 │
│                                                             │
│  SearchJobTitles ──► GetJobDescription ──► SaveJDDraft      │
│                                            └─► (Part 3)     │
│                                                SubmitForApproval │
└──────────────────────────┬──────────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  LAYER 4 — AGENT (new experience)                           │
│                                                             │
│  Instructions (short)                                       │
│  Skills: jd-intake-routing · review-existing-jd ·           │
│          jd-writing-standards                               │
│  Knowledge: JD standards library (SharePoint) — NOT the report │
│  Memory: ON   ·   Microsoft IQ: OFF (initially)             │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Layer 1 — Ingestion workflow

**Name:** `JD-Ingest-DailyReport`
**Type:** New-experience Workflow (or Power Automate cloud flow — either works; triggers now live in the Workflows section, not in the agent)

### Trigger
`When a new email arrives (V3)`
- **From:** the exact sender address of the report
- **Subject filter:** the exact recurring subject string
- **Include attachments:** Yes
- **Only with attachments:** Yes

Do not use a folder-based trigger unless the report reliably lands in one — sender + subject is more robust.

### Steps

1. **Guard clause.** Condition: attachment count > 0 AND attachment name matches expected pattern. If false → send yourself a failure notice and terminate. Silent ingestion failures are the #1 way this system rots.

2. **Parse the file.**
   - *If .xlsx with a named table:* `List rows present in a table` (Excel Online Business). Save the attachment to a staging SharePoint library first — the Excel connector needs a file path, not a byte array.
   - *If .csv:* save to staging, then either `Parse CSV` (if available in your tenant) or split on newline and use `split()` on the delimiter. Watch for commas inside Responsibilities — if the report has embedded commas, insist on a tab-delimited or xlsx feed instead. This will save you hours.

3. **Compute a row hash.** For each row, build a concatenated string of all JD fields and hash it (`sha256` isn't native; a simple approach is storing the concatenated string in a `SourceFingerprint` column and comparing). Purpose: skip untouched rows so you're not rewriting 800 list items daily and blowing up version history.

4. **Upsert loop.**
   - `Get items` on `JobDescriptions_Current` filtered by `JobCode eq '<code>'`
   - If found AND fingerprint unchanged → skip (just stamp `LastReportDate`)
   - If found AND fingerprint changed → `Update item`, and if `ReviewStatus` is `Draft` or `PendingApproval`, set a `SourceChangedDuringReview` flag rather than clobbering. (Edge case, but it will bite you.)
   - If not found → `Create item`

5. **Stamp the run.** Write `LastSuccessfulIngest` timestamp + row count to a small `JD_Config` list. Your `GetJobDescription` tool will surface this as data freshness to the user.

6. **Concurrency:** set the Apply to each to sequential (concurrency off) if you hit SharePoint throttling, or batch with `Send an HTTP request to SharePoint` if the volume is large.

### Test-mode gating
Same pattern you used on the escalation flow: a `TestMode` environment variable. When true, write to `JobDescriptions_Current_TEST` and email only you. Bake this in from day one — you'll want it for UAT.

---

## 3. Layer 2 — Data model

### List: `JobDescriptions_Current`

| Column | Type | Notes |
|---|---|---|
| `JobCode` | Single line (indexed) | **Primary key.** Use the requisition/job code from the report, not the title. Set as the list Title column for easy lookup. |
| `JobTitle` | Single line (indexed) | |
| `JobFunction` | Choice or Single line | |
| `IsSalesRole` | Yes/No | You called this out specifically — likely drives comp/incentive language |
| `JobFamily` | Single line | |
| `Grade` / `Level` | Single line | |
| `FLSAStatus` | Choice | Exempt/Non-Exempt |
| `ReportsTo` | Single line | |
| `JobSummary` | Multiple lines (plain) | |
| `Responsibilities` | Multiple lines (plain) | Keep plain text, not rich text — rich text HTML is painful to diff |
| `EducationRequirements` | Multiple lines (plain) | |
| `ExperienceRequirements` | Multiple lines (plain) | |
| `SkillsCompetencies` | Multiple lines (plain) | |
| `LicensesCertifications` | Multiple lines (plain) | |
| `PhysicalRequirements` | Multiple lines (plain) | |
| `LastReportDate` | Date/Time | From ingestion |
| `SourceFingerprint` | Single line | Change detection |
| `ReviewStatus` | Choice | `Current` / `InReview` / `Draft` / `PendingApproval` / `Approved` |
| `BaselineJSON` | Multiple lines (plain) | Immutable snapshot at review start — this is what your redline diffs against |

> Add the rest of your fields to this table — send me the full list and I'll finalize the schema and the tool contracts around it.

**Indexing matters.** Index `JobCode` and `JobTitle` before you exceed 5,000 items or your `Get items` filters will start failing on the list view threshold.

### List: `JD_ReviewSessions`

| Column | Type |
|---|---|
| `SessionId` | Single line (GUID) |
| `JobCode` | Single line |
| `ReviewerEmail` | Single line |
| `StartedOn` / `CompletedOn` | Date/Time |
| `Status` | Choice: `Open` / `Submitted` / `Abandoned` |
| `ChangeCount` | Number |

### List: `JD_ChangeLog`

| Column | Type |
|---|---|
| `SessionId` | Single line |
| `JobCode` | Single line |
| `FieldName` | Single line |
| `OldValue` | Multiple lines |
| `NewValue` | Multiple lines |
| `ChangedBy` | Person |
| `ChangedOn` | Date/Time |

This list is the entire payload for the Part 3 manager approval email. Build it now even though you're not emailing yet — retrofitting an audit trail is much worse than having one you don't use for a few weeks.

---

## 4. Layer 3 — Tools

Requirements for every agent-callable workflow: **`When an agent calls the flow` trigger + `Respond to the agent` action, asynchronous response OFF, published, and returns within 100 seconds.** A flow that runs fine on its own but lacks those exact node types won't even appear in the tool picker.

Keep the tool count small and the descriptions sharp — the orchestrator selects tools by description, so the description is functional code, not documentation.

### Tool 1: `SearchJobTitles`

**Description (this text does real work):**
> Searches the current job description catalog for jobs matching a user-supplied job title or keyword. Use this whenever the user names a job they want to review or update, before retrieving any job details. Returns up to 5 candidate matches with job code, title, and function. Does not return full job description content.

| Input | Type | Notes |
|---|---|---|
| `searchText` | string | Raw user phrasing |

| Output | Type |
|---|---|
| `matchCount` | number |
| `matchesJson` | string (JSON array of `{jobCode, jobTitle, jobFunction, grade}`) |

**Logic:** `Get items` with an OData `substringof` filter on `JobTitle`, top 5. If zero results, try splitting the search text and matching on the longest token. Return `matchCount: 0` cleanly rather than failing — the agent needs to offer the "is this a new job?" branch.

> If exact-match rates are poor, this is where your Levenshtein scoring work from the other agent transplants well — but start simple. Measure first.

### Tool 2: `GetJobDescription`

**Description:**
> Retrieves the complete, current job description fields for one specific job, identified by its exact job code. Call this only after the job code has been confirmed via SearchJobTitles. Returns all job description fields verbatim as stored in the HR system, plus the date the data was last refreshed.

| Input | Type |
|---|---|
| `jobCode` | string |
| `reviewerEmail` | string |

| Output | Type |
|---|---|
| `found` | boolean |
| `jobJson` | string (JSON object, all fields) |
| `lastReportDate` | string |
| `sessionId` | string |

**Logic:** Get the item → build the JSON object → **also** create the `JD_ReviewSessions` row and write `BaselineJSON` back to the job record, set `ReviewStatus = InReview`. Folding session creation into this tool saves you a round trip and guarantees a baseline exists before any edit.

**Return JSON, not prose.** Let the agent do the formatting per the skill instructions. If you return pre-formatted markdown, you lose the ability to have the agent present it section by section.

### Tool 3: `SaveJobDescriptionDraft`

**Description:**
> Saves the user's confirmed changes to a job description as a draft. Call this only after the user has explicitly confirmed a summary of all their changes. Accepts only the fields that changed. Returns a confirmation with the number of changes recorded.

| Input | Type |
|---|---|
| `sessionId` | string |
| `jobCode` | string |
| `changesJson` | string (JSON array of `{fieldName, newValue}`) |

| Output | Type |
|---|---|
| `saved` | boolean |
| `changeCount` | number |
| `summary` | string |

**Logic:** Parse `changesJson` → for each entry, read the current value from the job record → write a `JD_ChangeLog` row with old and new → update the job record → set `ReviewStatus = Draft` → update the session row.

**Do the diff in the flow, not in the agent.** The agent supplies only the new values; the flow reads old values from the record. This is the same principle as computing diffs deterministically in your JD creation agent — never trust the LLM to carry the "before" state accurately across a long conversation.

### Tool 4 (Part 3, stubbed now): `SubmitJDForApproval`
Takes `sessionId`, reads the change log, renders a word-level HTML redline, emails your manager, sets `ReviewStatus = PendingApproval`. Design the change log now so this is a pure read.

---

## 5. Layer 4 — Skills

Skills are Markdown with YAML front matter, loaded on demand by the orchestrator based on their description. Three to start.

### `jd-intake-routing.md`

```yaml
---
name: jd-intake-routing
description: Determines at the start of a conversation whether the user
  wants to create a brand-new job title or update an existing job
  description, then hands off to the correct workflow. Use at the
  beginning of every conversation and whenever the user's intent
  between new and existing is unclear.
---
```

**Instructions cover:**
- Open with a short greeting that presents exactly two paths: *create a new job title* or *update an existing job description*
- Do not ask open-ended "how can I help?" — the whole point is early disambiguation
- If the user names a job title without stating which path, call `SearchJobTitles` first; if there's a match, confirm "I found this existing job — are you updating it, or creating something new?"
- Zero matches → offer the new-job path
- Never guess the branch silently

### `review-existing-jd.md`

```yaml
---
name: review-existing-jd
description: Guides the conversational review and update of an existing
  job description. Use after the user has indicated they want to update
  a current job, from job identification through presenting each field
  for review, capturing changes, and saving the draft.
---
```

**Instructions cover:**

**Identify**
- Call `SearchJobTitles`. One match → confirm title + code. Multiple → present a numbered list with title, code, and function; ask the user to pick. Zero → offer the new-job path.
- Always display the job code alongside the title so the user knows precisely what they're editing.

**Present**
- Call `GetJobDescription`. State the data-refresh date in one short line.
- Present in sections, not as one wall of text:
  1. **Overview** — title, function, family, grade, FLSA, sales role, reports to
  2. **Summary**
  3. **Responsibilities**
  4. **Education**
  5. **Experience**
  6. **Skills & competencies**
  7. **Other requirements**
- After each section: "Does this still look right, or would you like to change anything here?"
- Let the user jump around ("actually go back to responsibilities") — the orchestrator handles this naturally, but say so explicitly in the skill so it doesn't railroad.

**Capture**
- When the user asks for a change, restate the proposed new value and get an explicit confirmation before moving on
- If the user gives a vague direction ("make it stronger", "add something about Salesforce"), draft a concrete revision and show it — never save an interpretation the user hasn't seen verbatim
- Track changes across the conversation; do not save incrementally

**Save**
- At the end, present a consolidated summary of every change: field, before, after
- Get one final explicit confirmation
- Then call `SaveJobDescriptionDraft` with only the changed fields
- Confirm the save and tell the user what happens next

**Hard guardrails (put these in the skill, verbatim in spirit):**
- Never state a job description field value that did not come from `GetJobDescription`
- If a field is empty in the source, say it's empty — do not fill it in
- If a tool call fails, tell the user plainly and stop; do not reconstruct data from memory or from earlier in the conversation
- Never save without explicit user confirmation of a change summary

### `jd-writing-standards.md`

```yaml
---
name: jd-writing-standards
description: House style rules for writing job description content —
  responsibility bullet structure, education and experience phrasing,
  inclusive language, and compliance requirements. Use whenever
  drafting or rewriting job description field content.
---
```

Your bullet formats, verb tense, banned phrasing, EEO and compliance language, how to phrase degree equivalency, sales-role specific requirements. Keeping this as a separate on-demand skill rather than in the main instruction block is a real token savings on every turn that isn't a rewrite.

---

## 6. Layer 5–7 — Knowledge, Memory, IQ, Instructions

**Knowledge:** point at a SharePoint library holding your JD standards, competency library, grade/level guidelines, comp philosophy docs. Give each source a detailed description — descriptions drive retrieval selection. **Not the daily report.**

**Memory:** turn it **on**. Useful for remembering a recruiter's department and preferred review pace. Caveats worth knowing: memories are per-user and private, makers can't see them, and they're deleted after 28 days of user inactivity. Don't build anything load-bearing on it.

**Microsoft IQ / Work IQ:** leave it **off** for v1. It pulls in org-wide email, calendar, files, and Teams context — which is precisely the ungrounded surface you're trying to avoid in a system where field accuracy is the product. Revisit later if there's a specific need.

**Instructions block:** keep it short. Role, the two-branch structure, the "facts come from tools only" rule, tone, and escalation. Everything procedural belongs in skills — that's the whole design intent of the new experience.

**Conversation starters:** "Update an existing job description" / "Create a new job title." These do a lot of the routing work for free.

---

## 7. Build order

| Phase | Deliverable | Done when |
|---|---|---|
| 0 | Finalize full field list → SharePoint schema | Lists created, `JobCode` and `JobTitle` indexed |
| 1 | `JD-Ingest-DailyReport` workflow | Tomorrow's report lands as clean rows; re-run is idempotent |
| 2 | `SearchJobTitles` + `GetJobDescription` | Both return correct JSON from the flow test harness, no agent involved |
| 3 | Agent shell + instructions + two tools | Agent retrieves and displays a real JD accurately in Preview |
| 4 | `jd-intake-routing` + `review-existing-jd` skills | Full conversational review runs end to end without saving |
| 5 | `SaveJobDescriptionDraft` + change log | Edits land in `JD_ChangeLog` with correct before/after |
| 6 | Evaluate tab test cases | Ambiguous titles, zero matches, empty fields, tool failure all handled |
| — | *Part 2: new job workflow* | |
| — | *Part 3: manager approval email* | |

**Don't skip Phase 2's standalone testing.** Debugging a tool contract through the orchestrator is dramatically harder than testing the flow directly. Get the JSON shapes right in isolation.

---

## 8. Things that will bite you

**One-way door.** New-experience agents cannot be converted to classic, and classic agents cannot be converted forward. Your existing HR agent stays where it is; this is a separate agent. Decide now whether these ever need to be one thing — if so, a **Connected agent** relationship is the path, not a merge.

**Solution placement.** In the new experience the solution-first setup step is gone — creating an agent drops it into the environment's default solution. Given your ALM history, move it into a proper unmanaged solution immediately after creation, before you build anything on top of it.

**The 100-second tool ceiling.** Return small payloads. If `SearchJobTitles` ever needs to scan a large list, filter server-side with OData; never retrieve-then-filter.

**Preview reasoning output.** The Preview window now shows the orchestrator's reasoning before the answer. That's a debugging gift — use the activity trace to see which tool fired and why when routing goes wrong.

**Preview status.** The new agent experience is a production-ready preview and skills/memory/workflows are still preview. Reasonable for an internal HR tool; worth flagging to anyone who asks about production readiness.
