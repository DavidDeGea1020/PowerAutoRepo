# JD Intake Agent — Build Manual, Part 1: Current Job Review
**Platform:** Copilot Studio (new agent experience) + Dataverse + Office Scripts
**Read time:** ~25 min · **Build time:** ~6–9 hours across 5 sessions

---

## How to use this manual

- Steps are numbered by part: **A1, A2…** in Part A, **B1, B2…** in Part B, and so on.
- **✓ VERIFY** blocks are checkpoints. Do not proceed past a failed verify — every downstream step assumes it passed.
- **⚠️** marks a step where a wrong value causes a failure that surfaces much later and is hard to trace.
- Anything in `code font` is a literal value to type or paste.
- `{{PREFIX}}` is your Dataverse publisher prefix. You'll fix its value in step A3 and substitute it everywhere after.

**Navigation caveat:** the new agent experience is a production-ready preview and workflows/skills are public preview. Menu labels move between releases. Where I write "look for," the label may read slightly differently — the action or setting itself is what matters.

**Parts list (what you're building):**

| # | Component | Part |
|---|---|---|
| 1 | Solution + publisher | A |
| 2 | `Job Description` Dataverse table | A |
| 3 | `JD Change Log` Dataverse table | A |
| 4 | `JD Review Session` Dataverse table | A |
| 5 | `JD Config` Dataverse table | A |
| 6 | CSV parser Office Script + host workbook | B |
| 7 | `JD-Ingest-DailyReport` flow | C |
| 8 | `SearchJobTitles` agent tool | D |
| 9 | `GetJobDescription` agent tool | D |
| 10 | `SaveJobDescriptionDraft` agent tool | D |
| 11 | The agent + instructions | E |
| 12 | Three skills | F |
| 13 | Test suite | G |

---

# PART A — Data foundation

*Session 1. Nothing here is conversational; get it exactly right and the rest is downhill.*

## A0. Map your CSV headers before you build anything

⚠️ **Do this first.** Open tomorrow's report (or today's, in Excel) and write down **the exact header text of every column, in order**. Spelling, spacing, capitalization, trailing spaces.

Create a scratch file with two columns:

| CSV header (exact) | Dataverse column (you'll fill in) |
|---|---|
| `Job Code` | `{{PREFIX}}_jobcode` |
| `Job Title` | `{{PREFIX}}_jobtitle` |
| `Sales Role (Y/N)` | `{{PREFIX}}_issalesrole` |
| … | … |

You told me the report includes sales-role flag, job function, responsibilities, education requirements, and experience requirements. The schema below covers those plus the fields these reports usually carry. **Send me your real header list and I'll finalize the mapping** — for now, add rows to the table in A4 for anything I've missed and remove what you don't have.

## A1. Choose your environment

Build in **DEV**. Everything in this manual is solution-aware and will export cleanly.

## A2. Create (or confirm) a publisher

1. Go to `make.powerapps.com`, confirm you're in your DEV environment (top right).
2. Left nav → **Solutions** → **Publishers** (may be under a **⋯ More** menu).
3. If your team already has a publisher for HR tooling, use it and skip to A3.
4. Otherwise **+ New publisher**:
   - Display name: `City National HR Tooling`
   - Name: `cnbhrtooling`
   - Prefix: `hrjd`
   - Choice value prefix: leave the auto-generated number
5. **Save**.

## A3. Fix your prefix value

From here on, **`{{PREFIX}}` = the prefix from A2** (e.g. `hrjd`).

Write it at the top of your scratch file. Every logical name below uses it. Dataverse will build logical names as `{{PREFIX}}_columnname` automatically when you type the display name — but **verify each one**, because Dataverse derives the logical name from the display name and strips characters unpredictably.

## A4. Create the solution

1. **Solutions** → **+ New solution**
2. Display name: `HR Job Description Intake`
3. Name: `HRJobDescriptionIntake`
4. Publisher: the one from A2
5. Version: `1.0.0.0`
6. **Create**

⚠️ Build **everything** from inside this solution. In the new agent experience the pre-creation solution picker is gone and agents drop into the environment's default solution — Part E has a step to move it. Tables and flows you create from inside the solution are captured automatically.

**✓ VERIFY A4:** Open the solution. It should be empty. The publisher shown in solution details matches A2.

---

## A5. Create table 1 of 4: `Job Description`

Inside the solution: **+ New** → **Table** → **Table (blank)**.

**Table properties:**

| Setting | Value |
|---|---|
| Display name | `Job Description` |
| Plural display name | `Job Descriptions` |
| Name | `{{PREFIX}}_jobdescription` |
| Primary column display name | `Job Title` |
| Primary column name | `{{PREFIX}}_jobtitle` |

Expand **Advanced options** and confirm:
- Ownership: **User or team**
- ☑ Create a new activity (leave default)
- ☑ Enable auditing — **turn this ON**

**Save**.

⚠️ The primary column is **Job Title**, not Job Code. Job Code gets its own column plus an alternate key in A7 — that's what makes upsert work. Do not swap these.

## A6. Add columns to `Job Description`

For each row: **+ New column**, set display name, data type, and the settings in the Notes column, then **Save**.

**Identity & classification**

| Display name | Logical name | Type | Notes |
|---|---|---|---|
| Job Code | `{{PREFIX}}_jobcode` | Single line of text | Max length `50`, **Required** |
| Job Function | `{{PREFIX}}_jobfunction` | Single line of text | Max length `100` |
| Job Family | `{{PREFIX}}_jobfamily` | Single line of text | Max length `100` |
| Grade / Level | `{{PREFIX}}_grade` | Single line of text | Max length `50` |
| FLSA Status | `{{PREFIX}}_flsastatus` | Single line of text | Max length `50` |
| Is Sales Role | `{{PREFIX}}_issalesrole` | Yes/No | Default **No** |
| Reports To | `{{PREFIX}}_reportsto` | Single line of text | Max length `150` |
| Department | `{{PREFIX}}_department` | Single line of text | Max length `150` |
| Location | `{{PREFIX}}_location` | Single line of text | Max length `150` |

**Job description content** — ⚠️ all of these are **Multiple lines of text**, and you must raise the max length. The default is 2,000 characters and a long Responsibilities field *will* exceed it. Set **Max length = `10000`** on each, and Format = **Text** (not Rich text — rich text HTML is miserable to diff).

| Display name | Logical name |
|---|---|
| Job Summary | `{{PREFIX}}_jobsummary` |
| Responsibilities | `{{PREFIX}}_responsibilities` |
| Education Requirements | `{{PREFIX}}_educationrequirements` |
| Experience Requirements | `{{PREFIX}}_experiencerequirements` |
| Skills And Competencies | `{{PREFIX}}_skillscompetencies` |
| Licenses And Certifications | `{{PREFIX}}_licensescertifications` |
| Physical Requirements | `{{PREFIX}}_physicalrequirements` |
| Working Conditions | `{{PREFIX}}_workingconditions` |

*(Add your remaining fields here following the same pattern.)*

**Control columns** — these aren't in your CSV; the system maintains them.

| Display name | Logical name | Type | Notes |
|---|---|---|---|
| Last Report Date | `{{PREFIX}}_lastreportdate` | Date and Time | Behavior: **User local** |
| Source Fingerprint | `{{PREFIX}}_sourcefingerprint` | Single line of text | Max length `500` |
| Review Status | `{{PREFIX}}_reviewstatus` | Choice (see below) | |
| Baseline JSON | `{{PREFIX}}_baselinejson` | Multiple lines of text | Max length `50000`, Format Text |
| Source Changed During Review | `{{PREFIX}}_sourcechangedduringreview` | Yes/No | Default **No** |

**Review Status choice values** — when creating the column choose **Choice** → **New choice**, name it `{{PREFIX}}_reviewstatus_options`, and add exactly:

| Label | Value |
|---|---|
| `Current` | `100000000` |
| `In Review` | `100000001` |
| `Draft` | `100000002` |
| `Pending Approval` | `100000003` |
| `Approved` | `100000004` |

Set the column's **Default choice** to `Current`.

⚠️ Write these integer values into your scratch file. Flows reference the integer, not the label, and guessing them later wastes an afternoon.

## A7. Add the alternate key on Job Code

This is the step that makes single-action upsert possible.

1. In the table editor, left panel → **Keys**
2. **+ New key**
3. Display name: `Job Code Key`
4. Name: `{{PREFIX}}_jobcodekey`
5. Columns: select **Job Code** only
6. **Save**

Wait for the key status to become **Active** (refresh after ~30 seconds). A key stuck in "Pending" means duplicate or null Job Code values exist — on an empty table it should activate immediately.

**✓ VERIFY A7:** Keys list shows `Job Code Key`, status **Active**.

## A8. Create table 2 of 4: `JD Change Log`

**+ New** → **Table** → **Table (blank)**

| Setting | Value |
|---|---|
| Display name | `JD Change Log` |
| Plural | `JD Change Logs` |
| Name | `{{PREFIX}}_jdchangelog` |
| Primary column display name | `Change Reference` |
| Primary column name | `{{PREFIX}}_changereference` |

Columns:

| Display name | Logical name | Type | Notes |
|---|---|---|---|
| Session Id | `{{PREFIX}}_sessionid` | Single line of text | Max `50` |
| Job Code | `{{PREFIX}}_jobcode` | Single line of text | Max `50` |
| Field Name | `{{PREFIX}}_fieldname` | Single line of text | Max `100` |
| Field Label | `{{PREFIX}}_fieldlabel` | Single line of text | Max `100` |
| Old Value | `{{PREFIX}}_oldvalue` | Multiple lines of text | Max `10000`, Text |
| New Value | `{{PREFIX}}_newvalue` | Multiple lines of text | Max `10000`, Text |
| Changed By | `{{PREFIX}}_changedby` | Single line of text | Max `150` — store the UPN as text, not a lookup |
| Changed On | `{{PREFIX}}_changedon` | Date and Time | User local |
| Submitted | `{{PREFIX}}_submitted` | Yes/No | Default **No** — flips true when Part 3 emails your manager |

⚠️ `Changed By` is deliberately plain text. A Dataverse user lookup requires the reviewer to exist as a system user record and adds a resolution step in every flow. Text UPN is enough for an approval email.

## A9. Create table 3 of 4: `JD Review Session`

| Setting | Value |
|---|---|
| Display name | `JD Review Session` |
| Plural | `JD Review Sessions` |
| Name | `{{PREFIX}}_jdreviewsession` |
| Primary column display name | `Session Id` |
| Primary column name | `{{PREFIX}}_sessionid` |

Columns:

| Display name | Logical name | Type | Notes |
|---|---|---|---|
| Job Code | `{{PREFIX}}_jobcode` | Single line of text | Max `50` |
| Reviewer Email | `{{PREFIX}}_revieweremail` | Single line of text | Max `150` |
| Started On | `{{PREFIX}}_startedon` | Date and Time | User local |
| Completed On | `{{PREFIX}}_completedon` | Date and Time | User local |
| Change Count | `{{PREFIX}}_changecount` | Whole number | Min `0` |
| Status | `{{PREFIX}}_status` | Choice `{{PREFIX}}_sessionstatus_options` | `Open`=`100000000`, `Submitted`=`100000001`, `Abandoned`=`100000002`. Default `Open` |

## A10. Create table 4 of 4: `JD Config`

Single-row table holding ingestion health.

| Setting | Value |
|---|---|
| Display name | `JD Config` |
| Plural | `JD Configs` |
| Name | `{{PREFIX}}_jdconfig` |
| Primary column display name | `Config Key` |
| Primary column name | `{{PREFIX}}_configkey` |

Columns:

| Display name | Logical name | Type |
|---|---|---|
| Last Successful Ingest | `{{PREFIX}}_lastsuccessfulingest` | Date and Time (User local) |
| Last Ingest Row Count | `{{PREFIX}}_lastingestrowcount` | Whole number |
| Last Ingest Status | `{{PREFIX}}_lastingeststatus` | Single line of text (max `500`) |

Then create one row manually: **Data** → **+ New row** → Config Key = `PRIMARY`. Save. Note the record GUID from the URL — you'll paste it in C14.

## A11. Add environment variables

Inside the solution: **+ New** → **More** → **Environment variable**.

**Variable 1 — Test Mode**

| Setting | Value |
|---|---|
| Display name | `JD Test Mode` |
| Name | `{{PREFIX}}_JDTestMode` |
| Data type | Yes/No |
| Default value | `Yes` |

**Variable 2 — Report sender**

| Setting | Value |
|---|---|
| Display name | `JD Report Sender` |
| Name | `{{PREFIX}}_JDReportSender` |
| Data type | Text |
| Default value | the exact sending address of the daily report |

**Variable 3 — Report subject filter**

| Setting | Value |
|---|---|
| Display name | `JD Report Subject Filter` |
| Name | `{{PREFIX}}_JDReportSubjectFilter` |
| Data type | Text |
| Default value | a distinctive substring of the subject line |

**Variable 4 — Admin notification address**

| Setting | Value |
|---|---|
| Display name | `JD Admin Email` |
| Name | `{{PREFIX}}_JDAdminEmail` |
| Data type | Text |
| Default value | your email |

⚠️ Set **default values only**, not current values. Current values don't export cleanly and cause the classic "environment variable has no value" deployment stall you've hit before.

## A12. Publish

Solution → **Publish all customizations**. Wait for completion.

**✓ VERIFY PART A:**
1. Solution contains 4 tables and 4 environment variables.
2. `Job Description` → Keys → `Job Code Key` is **Active**.
3. Open `Job Description` → **Data** → **+ New row**. Paste 3,000 characters of lorem ipsum into Responsibilities. It saves without truncation. **Delete the test row.**
4. Your scratch file has: the prefix value, all choice integer values, and the JD Config record GUID.

---

# PART B — CSV parser

*Session 2. ~45 minutes.*

Your report is a CSV attachment, and CSV plus free-text Responsibilities is the highest-risk part of this build. Splitting on commas in Power Automate expressions will shred any field containing a comma, a quote, or an embedded newline — and it fails **silently**, producing plausible-looking rows with data in the wrong columns.

You'll parse it properly with an Office Script instead.

> **Before building this:** it is worth one email asking the report sender to switch the attachment to `.xlsx` or tab-delimited. Ten minutes of asking beats maintaining a parser. If they say no, or you can't wait, continue.

## B1. Create the host workbook

Office Scripts need a workbook to run against, even when the script never touches it.

1. Go to your HR SharePoint site → Documents → create a folder `JD Automation`.
2. **New** → **Excel workbook**. Rename it `JD-CSV-Parser-Host.xlsx`.
3. Leave it empty. Close it.

⚠️ Store it in SharePoint or OneDrive for Business, not on your desktop. The Power Automate connector needs a cloud file path.

## B2. Create the script

1. Open `JD-CSV-Parser-Host.xlsx` in Excel for the web.
2. Ribbon → **Automate** → **New Script**.
3. Delete all boilerplate in the editor.
4. Paste the code in B3 in full.
5. Rename the script (top of the editor pane): `ParseJobDescriptionCsv`
6. **Save script**.

## B3. The script

```typescript
/**
 * ParseJobDescriptionCsv
 * RFC 4180-compliant CSV parser for the daily job description report.
 * Handles quoted fields, escaped quotes, embedded commas and newlines.
 *
 * @param workbook  Required by the Office Scripts runtime. Unused.
 * @param csvText   Raw CSV file contents as a string.
 * @param delimiter Field delimiter. Default ",". Pass "\t" for TSV.
 * @returns JSON string: { ok, rowCount, headers, rows, error }
 */
function main(
  workbook: ExcelScript.Workbook,
  csvText: string,
  delimiter: string = ","
): string {

  if (!csvText || csvText.length === 0) {
    return JSON.stringify({
      ok: false, error: "Empty CSV payload", rowCount: 0, headers: [], rows: []
    });
  }

  let grid: string[][];
  try {
    grid = parseCsv(csvText, delimiter);
  } catch (e) {
    return JSON.stringify({
      ok: false, error: "Parse failure: " + e, rowCount: 0, headers: [], rows: []
    });
  }

  if (grid.length < 2) {
    return JSON.stringify({
      ok: false, error: "Header row present but no data rows", rowCount: 0,
      headers: grid.length ? grid[0] : [], rows: []
    });
  }

  const rawHeaders: string[] = grid[0];
  const headers: string[] = rawHeaders.map(h => normalizeHeader(h));

  // Reject a file whose headers changed shape upstream.
  if (headers.filter(h => h.length > 0).length === 0) {
    return JSON.stringify({
      ok: false, error: "No usable headers", rowCount: 0, headers: [], rows: []
    });
  }

  const records: { [key: string]: string }[] = [];

  for (let r = 1; r < grid.length; r++) {
    const cells: string[] = grid[r];

    // Skip fully blank lines
    if (cells.length === 1 && cells[0].trim() === "") { continue; }

    const record: { [key: string]: string } = {};
    for (let c = 0; c < headers.length; c++) {
      if (headers[c].length === 0) { continue; }
      const value: string = c < cells.length ? cells[c] : "";
      record[headers[c]] = value.trim();
    }
    records.push(record);
  }

  return JSON.stringify({
    ok: true,
    error: "",
    rowCount: records.length,
    headers: headers,
    rows: records
  });
}

/**
 * State-machine CSV parser. Correctly handles:
 *   "field, with comma"
 *   "field with ""escaped"" quotes"
 *   "field with
 *    embedded newline"
 */
function parseCsv(text: string, delimiter: string): string[][] {

  // Strip UTF-8 BOM if present — otherwise the first header is corrupted.
  if (text.charCodeAt(0) === 0xFEFF) {
    text = text.substring(1);
  }

  const rows: string[][] = [];
  let row: string[] = [];
  let field: string = "";
  let inQuotes: boolean = false;

  for (let i = 0; i < text.length; i++) {
    const ch: string = text.charAt(i);

    if (inQuotes) {
      if (ch === '"') {
        if (i + 1 < text.length && text.charAt(i + 1) === '"') {
          field += '"';   // escaped quote
          i++;
        } else {
          inQuotes = false;
        }
      } else {
        field += ch;
      }
      continue;
    }

    if (ch === '"') {
      inQuotes = true;
    } else if (ch === delimiter) {
      row.push(field);
      field = "";
    } else if (ch === "\r") {
      // ignore; handled by \n
    } else if (ch === "\n") {
      row.push(field);
      rows.push(row);
      row = [];
      field = "";
    } else {
      field += ch;
    }
  }

  // Final field / row if file does not end with a newline
  if (field.length > 0 || row.length > 0) {
    row.push(field);
    rows.push(row);
  }

  return rows;
}

/**
 * Converts a CSV header into a stable key.
 * "Education Requirements " -> "educationrequirements"
 * "Sales Role (Y/N)"        -> "salesroleyn"
 */
function normalizeHeader(header: string): string {
  let out: string = "";
  const lower: string = header.toLowerCase();
  for (let i = 0; i < lower.length; i++) {
    const ch: string = lower.charAt(i);
    const isLetter: boolean = ch >= "a" && ch <= "z";
    const isDigit: boolean = ch >= "0" && ch <= "9";
    if (isLetter || isDigit) { out += ch; }
  }
  return out;
}
```

## B4. Test the script standalone

1. In the script editor, the **main** signature now takes parameters — the editor shows input boxes when you press **Run**.
2. In the `csvText` box paste this deliberately nasty sample:

```
Job Code,Job Title,Sales Role (Y/N),Responsibilities
40312,"Sales Representative, Senior",Y,"Manage accounts; prospect new business, upsell existing clients"
40313,Analyst II,N,"Prepare reports.
Reconcile ledgers. Handle ""special"" requests."
```

3. Leave `delimiter` as `,`.
4. **Run**.

**✓ VERIFY B4:** The output log shows `"ok":true` and `"rowCount":2`. Row 1's Job Title is `Sales Representative, Senior` **with the comma intact**. Row 2's Responsibilities contains both sentences and the word `"special"` in quotes. If any field is split across keys, the paste was mangled — re-copy and retry.

## B5. Record your normalized header keys

Run the script once against a **real** report file and copy the `headers` array from the output. These normalized keys (`jobcode`, `jobtitle`, `salesroleyn`, …) are the exact strings your flow will use in step C11.

Paste them into your scratch file next to the Dataverse column mapping from A0.

⚠️ If the report's headers ever change upstream, these keys change and the flow silently writes nulls. C13 adds a guard for exactly this.

---

# PART C — Ingestion flow

*Session 3. ~2 hours.*

## C1. Create the flow

From inside your solution: **+ New** → **Automation** → **Cloud flow** → **Automated**.

- Flow name: `JD-Ingest-DailyReport`
- Trigger: search `When a new email arrives (V3)` (Office 365 Outlook)
- **Create**

## C2. Configure the trigger

| Setting | Value |
|---|---|
| Folder | `Inbox` |
| To | leave blank |
| From | leave blank — you'll filter in C4 using the env variable |
| Include Attachments | **Yes** |
| Only with Attachments | **Yes** |
| Subject Filter | leave blank — filtered in C4 |

⚠️ Leaving From and Subject blank on the trigger and filtering inside the flow costs a few extra runs but keeps the filter values in environment variables, so DEV/UAT/PROD can point at different senders without editing the flow.

## C3. Read the environment variables

Add four **Initialize variable** actions (or one **Compose** each — variables are clearer here).

| # | Action name | Name | Type | Value |
|---|---|---|---|---|
| 1 | `Init varTestMode` | `varTestMode` | Boolean | env var `{{PREFIX}}_JDTestMode` |
| 2 | `Init varReportSender` | `varReportSender` | String | env var `{{PREFIX}}_JDReportSender` |
| 3 | `Init varSubjectFilter` | `varSubjectFilter` | String | env var `{{PREFIX}}_JDReportSubjectFilter` |
| 4 | `Init varAdminEmail` | `varAdminEmail` | String | env var `{{PREFIX}}_JDAdminEmail` |

To reference an environment variable, use the dynamic content picker's **Environment variable** section, or the expression:

```
parameters('{{PREFIX}}_JDReportSender')
```

## C4. Gate on sender and subject

Add a **Condition** named `Check Is Report Email`.

Left value (expression):
```
and(
  contains(toLower(triggerOutputs()?['body/from']), toLower(variables('varReportSender'))),
  contains(toLower(triggerOutputs()?['body/subject']), toLower(variables('varSubjectFilter')))
)
```
Operator: **is equal to** · Right value: expression `true`

In the **If no** branch: add **Terminate** with status `Succeeded`. Not Failed — a non-matching email isn't an error and you don't want failure noise in your run history.

Everything from C5 on goes in the **If yes** branch.

## C5. Isolate the CSV attachment

Add **Filter array**.

- From: `triggerOutputs()?['body/attachments']`
- Left: expression `toLower(item()?['name'])`
- Condition: **ends with**
- Right: `.csv`

Rename the action `Filter CSV Attachments`.

## C6. Guard: exactly one CSV

Add a **Condition** named `Check One CSV Present`.

Left (expression): `length(body('Filter_CSV_Attachments'))`
Operator: **is equal to** · Right: `1`

**If no** branch:
1. **Send an email (V2)** — To: `variables('varAdminEmail')`, Subject: `JD Ingest FAILED — attachment count`, Body: include `length(body('Filter_CSV_Attachments'))` and the trigger subject.
2. **Terminate**, status `Failed`, message `Expected exactly one CSV attachment`.

⚠️ Do not skip this. Silent ingestion failure is how this system rots without anyone noticing for weeks.

## C7. Decode the CSV to text

In the **If yes** branch, add **Compose** named `Compose CSV Text`:

```
base64ToString(first(body('Filter_CSV_Attachments'))?['contentBytes'])
```

## C8. Run the parser

Add **Run script** (Excel Online (Business)).

| Setting | Value |
|---|---|
| Location | your SharePoint site |
| Document Library | `Documents` |
| File | `/JD Automation/JD-CSV-Parser-Host.xlsx` |
| Script | `ParseJobDescriptionCsv` |
| csvText | `outputs('Compose_CSV_Text')` |
| delimiter | `,` |

Rename the action `Run CSV Parser`.

⚠️ Office Script parameter and return payloads have size ceilings. If your report is very large and this action fails on payload size, split `Compose CSV Text` into chunks by line count and call the script per chunk, or push the sender for xlsx. Test with a real file early.

## C9. Parse the script result

Add **Parse JSON** named `Parse Script Result`.

Content: `body('Run_CSV_Parser')?['result']`

Schema:
```json
{
  "type": "object",
  "properties": {
    "ok": { "type": "boolean" },
    "error": { "type": "string" },
    "rowCount": { "type": "integer" },
    "headers": { "type": "array", "items": { "type": "string" } },
    "rows": { "type": "array", "items": { "type": "object" } }
  }
}
```

⚠️ The script returns a JSON **string**. If Parse JSON errors with "expected object, got string," wrap the content in `json(...)`:
```
json(body('Run_CSV_Parser')?['result'])
```

## C10. Guard: parser succeeded

**Condition** `Check Parse OK`: `body('Parse_Script_Result')?['ok']` **is equal to** `true`.

**If no**: email `varAdminEmail` with `body('Parse_Script_Result')?['error']`, then **Terminate** / `Failed`.

## C11. Loop the rows

**Apply to each** named `For Each Job Row`.

Select an output: `body('Parse_Script_Result')?['rows']`

⚠️ Open the action's **Settings** → **Concurrency Control** → **On** → **Degree of Parallelism: 1**. Parallel upserts against the same table produce throttling and out-of-order writes. Slower and correct beats faster and wrong.

Everything from C12 to C16 lives **inside** this loop.

## C12. Build the fingerprint

**Compose** named `Compose Fingerprint`:

```
concat(
  item()?['jobtitle'], '|',
  item()?['jobfunction'], '|',
  item()?['responsibilities'], '|',
  item()?['educationrequirements'], '|',
  item()?['experiencerequirements'], '|',
  item()?['skillscompetencies'], '|',
  item()?['salesroleyn']
)
```

Replace the keys with **your** normalized headers from B5.

Then **Compose** named `Compose Fingerprint Hash` — Dataverse's 500-char limit means you shouldn't store the raw concatenation:

```
string(length(outputs('Compose_Fingerprint')))
```

⚠️ Length alone is a weak fingerprint — it misses same-length edits. Better: if your tenant allows it, use the `Data Operations` → `Compose` with
```
base64(outputs('Compose_Fingerprint'))
```
and store the **last 400 characters** via `substring()`. Or accept the extra writes and skip fingerprinting entirely in v1, adding it only if daily run duration becomes a problem. **Recommended for v1: skip the fingerprint optimization.** Get correctness first; optimize when you have a measured problem.

*(If skipping: omit C12 and the fingerprint condition, and always update.)*

## C13. Guard: required keys present

**Condition** `Check Row Has Job Code`:

Left: `coalesce(item()?['jobcode'], '')`
Operator: **is not equal to** · Right: leave empty

**If no**: **Append to array variable** (initialize `varSkippedRows` as an Array back in C3) with the row index, then do nothing else for this row. This catches upstream header renames — a run where every row is skipped tells you immediately that the report format changed.

## C14. Upsert the row

Inside **If yes**, add **Upsert a row** (Microsoft Dataverse).

| Setting | Value |
|---|---|
| Table name | `Job Descriptions` |
| Row ID | `{{PREFIX}}_jobcode='@{item()?['jobcode']}'` |

Then map each column. Use the **Show all** / advanced parameters toggle to reveal every field.

| Dataverse field | Value |
|---|---|
| Job Title (primary) | `item()?['jobtitle']` |
| Job Code | `item()?['jobcode']` |
| Job Function | `item()?['jobfunction']` |
| Job Family | `item()?['jobfamily']` |
| Grade / Level | `item()?['grade']` |
| FLSA Status | `item()?['flsastatus']` |
| Is Sales Role | `if(or(equals(toLower(coalesce(item()?['salesroleyn'],'')),'y'), equals(toLower(coalesce(item()?['salesroleyn'],'')),'yes')), true, false)` |
| Reports To | `item()?['reportsto']` |
| Job Summary | `item()?['jobsummary']` |
| Responsibilities | `item()?['responsibilities']` |
| Education Requirements | `item()?['educationrequirements']` |
| Experience Requirements | `item()?['experiencerequirements']` |
| Skills And Competencies | `item()?['skillscompetencies']` |
| Licenses And Certifications | `item()?['licensescertifications']` |
| Physical Requirements | `item()?['physicalrequirements']` |
| Last Report Date | `utcNow()` |

⚠️ **Do not map Review Status or Baseline JSON here.** Ingestion must never overwrite an in-progress review. Those columns are owned entirely by the agent tools in Part D.

**If you don't see an "Upsert a row" action** in your connector version, use this fallback instead:
1. **List rows** — Table `Job Descriptions`, Filter rows: `{{PREFIX}}_jobcode eq '@{item()?['jobcode']}'`, Row count `1`
2. **Condition**: `length(outputs('List_rows')?['body/value'])` **is greater than** `0`
3. **If yes** → **Update a row**, Row ID = `first(outputs('List_rows')?['body/value'])?['{{PREFIX}}_jobdescriptionid']`
4. **If no** → **Add a new row**
5. Map the same columns in both branches.

## C15. Flag source changes during review

Still inside the loop, after the upsert: add a **Condition** checking whether this record was mid-review.

Simplest reliable approach: before the upsert, add **List rows** filtered to this job code returning `{{PREFIX}}_reviewstatus`. After the upsert, if the prior status was `In Review` (`100000001`) or `Draft` (`100000002`), **Update a row** setting `Source Changed During Review` = `Yes`.

*(If this feels like scope creep for v1, skip it — but add a note in your backlog. It's a real edge case that will eventually confuse someone.)*

## C16. Stamp the config row

**Outside** the loop, after `For Each Job Row`:

**Update a row** (Dataverse)
| Setting | Value |
|---|---|
| Table name | `JD Configs` |
| Row ID | the GUID you saved in A10 |
| Last Successful Ingest | `utcNow()` |
| Last Ingest Row Count | `body('Parse_Script_Result')?['rowCount']` |
| Last Ingest Status | `concat('OK. Skipped rows: ', string(length(variables('varSkippedRows'))))` |

## C17. Configure run-after error handling

1. Add a **Send an email (V2)** action at the very end named `Notify Ingest Failure`.
2. Click its **⋯** → **Configure run after** → check **has failed**, **is skipped**, **has timed out**; uncheck **is successful**.
3. To: `variables('varAdminEmail')` · Subject: `JD Ingest FAILED — run @{workflow()['run']['name']}`

## C18. Save and test

1. **Save**.
2. Forward yourself a copy of a real report email so it matches the sender/subject filters — or temporarily set the `JD Report Sender` env var current value to your own address.
3. Watch the run.

**✓ VERIFY PART C:**
1. Run history shows **Succeeded**.
2. `Job Description` table row count matches the report's data row count.
3. Open a job whose Responsibilities field contains a comma. **The text is complete and in the right column.** This is the single most important check in the entire manual.
4. Re-run the same email. Row count does **not** double — upsert matched on Job Code.
5. `JD Config` row `PRIMARY` shows today's timestamp.

---

# PART D — Agent tools

*Session 4. ~2 hours.*

Every agent-callable workflow must have the **`When an agent calls the flow`** trigger, a **`Respond to the agent`** action, **asynchronous response OFF**, and be **published**. A flow missing those exact node types won't appear in the tool picker at all — it won't error, it just won't be listed.

Also: each must return within **100 seconds**.

## D1. Tool 1 — `SearchJobTitles`

### D1.1 Create
From the solution: **+ New** → **Automation** → **Cloud flow** → **Instant**, or from Copilot Studio's Tools panel choose **+ Add a tool** → **Workflow** → create new. Either way you need the agent trigger.

Name: `SearchJobTitles`

### D1.2 Trigger inputs

Trigger: `When an agent calls the flow`. Add these inputs:

| Name | Type | Description (this text is read by the orchestrator) |
|---|---|---|
| `searchText` | Text | The job title, keyword, or job code the user provided, in their own words. |

### D1.3 Detect a direct code lookup

**Compose** `Compose Clean Search`:
```
trim(coalesce(triggerBody()?['text'], ''))
```
*(Field reference name may differ — use the dynamic content picker for the `searchText` input.)*

**List rows** named `List By Code`:
- Table: `Job Descriptions`
- Filter rows: `{{PREFIX}}_jobcode eq '@{outputs('Compose_Clean_Search')}'`
- Row count: `1`

### D1.4 Branch on code hit

**Condition** `Check Code Match`: `length(outputs('List_By_Code')?['body/value'])` **is greater than** `0`

**If yes** → skip to the response, using `List By Code` results.

**If no** → **List rows** named `List By Title`:
- Table: `Job Descriptions`
- Filter rows:
```
contains(@{concat('''', replace(outputs('Compose_Clean_Search'), '''', ''''''), '''')}, '')
```
Simpler and safer — use this literal filter expression instead:
```
contains({{PREFIX}}_jobtitle,'@{replace(outputs('Compose_Clean_Search'), '''', '''''')}')
```
- Select columns: `{{PREFIX}}_jobcode,{{PREFIX}}_jobtitle,{{PREFIX}}_jobfunction,{{PREFIX}}_grade,{{PREFIX}}_issalesrole`
- Row count: `5`

⚠️ The `replace()` doubles single quotes. Without it, a search for `O'Brien Analyst` breaks the OData filter. Filter injection via a job title is unlikely but the flow failure is not.

### D1.5 Build the response payload

**Select** action named `Select Matches`:
- From: the results array from whichever branch ran (use `union()` or a variable set in both branches)
- Map:
  | Key | Value |
  |---|---|
  | `jobCode` | `item()?['{{PREFIX}}_jobcode']` |
  | `jobTitle` | `item()?['{{PREFIX}}_jobtitle']` |
  | `jobFunction` | `item()?['{{PREFIX}}_jobfunction']` |
  | `grade` | `item()?['{{PREFIX}}_grade']` |
  | `isSalesRole` | `item()?['{{PREFIX}}_issalesrole']` |

### D1.6 Respond to the agent

**Respond to the agent** — add outputs:

| Name | Type | Value |
|---|---|---|
| `matchCount` | Number | `length(body('Select_Matches'))` |
| `matchesJson` | Text | `string(body('Select_Matches'))` |
| `searchInterpretedAs` | Text | `outputs('Compose_Clean_Search')` |

⚠️ **Settings** → **Networking** → **Asynchronous Response: Off**.

### D1.7 Publish
**Save**, then **Publish**.

**✓ VERIFY D1:** Use the flow's **Test** panel. Input `Analyst`. Output returns `matchCount` > 0 and `matchesJson` containing valid JSON. Input a nonsense string — returns `matchCount: 0` and **succeeds** (does not fail).

---

## D2. Tool 2 — `GetJobDescription`

Name: `GetJobDescription`

### D2.1 Trigger inputs

| Name | Type | Description |
|---|---|---|
| `jobCode` | Text | The exact job code of the job to retrieve. Must come from a confirmed SearchJobTitles result. |
| `reviewerEmail` | Text | Email address of the person conducting the review. |

### D2.2 Retrieve

**Get a row by ID** (Dataverse):
- Table: `Job Descriptions`
- Row ID: `{{PREFIX}}_jobcode='@{triggerBody()?['jobCode']}'`

Configure run-after handling: add a parallel **Respond to the agent** with `found: false` on the failure path, so a bad code returns cleanly rather than erroring.

### D2.3 Build the JD object

**Compose** named `Compose Job JSON`:

```json
{
  "jobCode": "@{outputs('Get_a_row_by_ID')?['body/{{PREFIX}}_jobcode']}",
  "jobTitle": "@{outputs('Get_a_row_by_ID')?['body/{{PREFIX}}_jobtitle']}",
  "jobFunction": "@{outputs('Get_a_row_by_ID')?['body/{{PREFIX}}_jobfunction']}",
  "jobFamily": "@{outputs('Get_a_row_by_ID')?['body/{{PREFIX}}_jobfamily']}",
  "grade": "@{outputs('Get_a_row_by_ID')?['body/{{PREFIX}}_grade']}",
  "flsaStatus": "@{outputs('Get_a_row_by_ID')?['body/{{PREFIX}}_flsastatus']}",
  "isSalesRole": @{if(equals(outputs('Get_a_row_by_ID')?['body/{{PREFIX}}_issalesrole'], true), 'true', 'false')},
  "reportsTo": "@{outputs('Get_a_row_by_ID')?['body/{{PREFIX}}_reportsto']}",
  "jobSummary": @{json(concat('"', replace(replace(coalesce(outputs('Get_a_row_by_ID')?['body/{{PREFIX}}_jobsummary'],''), '\', '\\'), '"', '\"'), '"'))},
  "responsibilities": @{json(concat('"', replace(replace(coalesce(outputs('Get_a_row_by_ID')?['body/{{PREFIX}}_responsibilities'],''), '\', '\\'), '"', '\"'), '"'))},
  "educationRequirements": @{json(concat('"', replace(replace(coalesce(outputs('Get_a_row_by_ID')?['body/{{PREFIX}}_educationrequirements'],''), '\', '\\'), '"', '\"'), '"'))},
  "experienceRequirements": @{json(concat('"', replace(replace(coalesce(outputs('Get_a_row_by_ID')?['body/{{PREFIX}}_experiencerequirements'],''), '\', '\\'), '"', '\"'), '"'))},
  "skillsCompetencies": @{json(concat('"', replace(replace(coalesce(outputs('Get_a_row_by_ID')?['body/{{PREFIX}}_skillscompetencies'],''), '\', '\\'), '"', '\"'), '"'))},
  "licensesCertifications": @{json(concat('"', replace(replace(coalesce(outputs('Get_a_row_by_ID')?['body/{{PREFIX}}_licensescertifications'],''), '\', '\\'), '"', '\"'), '"'))},
  "physicalRequirements": @{json(concat('"', replace(replace(coalesce(outputs('Get_a_row_by_ID')?['body/{{PREFIX}}_physicalrequirements'],''), '\', '\\'), '"', '\"'), '"'))}
}
```

⚠️ The nested `replace()` calls escape backslashes and quotes so that free-text JD content doesn't break the JSON. **This is the most error-prone expression in the build.** If you'd rather avoid it, build the object with a **Select** action or a small Office Script instead — both are less fragile than hand-escaping.

### D2.4 Create the review session

**Add a new row** → `JD Review Sessions`:

| Field | Value |
|---|---|
| Session Id (primary) | `guid()` — but capture it first in a **Compose** named `Compose Session Id` so you can return the same value |
| Job Code | `triggerBody()?['jobCode']` |
| Reviewer Email | `triggerBody()?['reviewerEmail']` |
| Started On | `utcNow()` |
| Status | `Open` |
| Change Count | `0` |

⚠️ Call `guid()` **once** in a Compose at the top and reference `outputs('Compose_Session_Id')` everywhere. Calling `guid()` twice returns two different values and your change log will orphan.

### D2.5 Snapshot the baseline

**Update a row** → `Job Descriptions`, Row ID `{{PREFIX}}_jobcode='@{triggerBody()?['jobCode']}'`:

| Field | Value |
|---|---|
| Baseline JSON | `outputs('Compose_Job_JSON')` |
| Review Status | `In Review` |

This immutable snapshot is what Part 3's redline diffs against.

### D2.6 Read data freshness

**Get a row by ID** → `JD Configs`, the GUID from A10. Capture `Last Successful Ingest`.

### D2.7 Respond

| Output name | Type | Value |
|---|---|---|
| `found` | Boolean | `true` |
| `jobJson` | Text | `string(outputs('Compose_Job_JSON'))` |
| `lastReportDate` | Text | `formatDateTime(outputs('Get_Config_Row')?['body/{{PREFIX}}_lastsuccessfulingest'], 'MMMM d, yyyy')` |
| `sessionId` | Text | `outputs('Compose_Session_Id')` |
| `sourceChangedDuringReview` | Boolean | the column value |

Asynchronous Response **Off**. **Save** → **Publish**.

**✓ VERIFY D2:** Test with a real job code. `jobJson` output parses as valid JSON (paste it into a JSON validator). A new row exists in `JD Review Sessions`. The job's Review Status is now `In Review` and Baseline JSON is populated.

---

## D3. Tool 3 — `SaveJobDescriptionDraft`

Name: `SaveJobDescriptionDraft`

### D3.1 Trigger inputs

| Name | Type | Description |
|---|---|---|
| `sessionId` | Text | The session ID returned by GetJobDescription for this review. |
| `jobCode` | Text | The job code being updated. |
| `reviewerEmail` | Text | Email of the person making the changes. |
| `changesJson` | Text | JSON array of objects, each with fieldName and newValue, containing only fields the user changed and confirmed. |

### D3.2 Parse the changes

**Parse JSON** named `Parse Changes`:
- Content: `json(triggerBody()?['changesJson'])`
- Schema:
```json
{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "fieldName": { "type": "string" },
      "newValue": { "type": "string" }
    },
    "required": ["fieldName", "newValue"]
  }
}
```

### D3.3 Fetch the current record

**Get a row by ID** → `Job Descriptions`, Row ID `{{PREFIX}}_jobcode='@{triggerBody()?['jobCode']}'`.

⚠️ **The old value comes from this record, never from the agent.** This is the core rule of the tool: the agent supplies only what changed *to*, the flow determines what it changed *from*. Never trust an LLM to carry "before" state accurately across a long conversation.

### D3.4 Log each change

**Apply to each** over `body('Parse_Changes')`, concurrency **1**.

Inside, add a **Switch** on `item()?['fieldName']` with a case per editable field. In each case, an **Add a new row** to `JD Change Logs`:

| Field | Value (example: Responsibilities case) |
|---|---|
| Change Reference (primary) | `concat(triggerBody()?['jobCode'], '-', item()?['fieldName'], '-', utcNow('yyyyMMddHHmmss'))` |
| Session Id | `triggerBody()?['sessionId']` |
| Job Code | `triggerBody()?['jobCode']` |
| Field Name | `item()?['fieldName']` |
| Field Label | `Responsibilities` |
| Old Value | `outputs('Get_a_row_by_ID')?['body/{{PREFIX}}_responsibilities']` |
| New Value | `item()?['newValue']` |
| Changed By | `triggerBody()?['reviewerEmail']` |
| Changed On | `utcNow()` |
| Submitted | `No` |

**Default case:** add a **Compose** recording the unrecognized field name and append it to an array variable `varUnknownFields`. Return it in the response so you can see when the agent invents a field name.

### D3.5 Apply the changes

Still inside the Switch cases, add an **Update a row** on `Job Descriptions` setting the matching column to `item()?['newValue']`.

*(Alternative that avoids a giant Switch: build a single Update a row after the loop, with each column set via a lookup expression against the changes array. More elegant, harder to debug. Use the Switch for v1.)*

### D3.6 Finalize

After the loop:

1. **Update a row** → `Job Descriptions`: Review Status = `Draft`
2. **List rows** → `JD Review Sessions` filtered `{{PREFIX}}_sessionid eq '@{triggerBody()?['sessionId']}'`
3. **Update a row** → that session: Change Count = `length(body('Parse_Changes'))`

### D3.7 Respond

| Output | Type | Value |
|---|---|---|
| `saved` | Boolean | `true` |
| `changeCount` | Number | `length(body('Parse_Changes'))` |
| `unknownFields` | Text | `string(variables('varUnknownFields'))` |
| `summary` | Text | `concat('Saved ', string(length(body('Parse_Changes'))), ' change(s) to job ', triggerBody()?['jobCode'], ' as a draft.')` |

Asynchronous Response **Off**. **Save** → **Publish**.

**✓ VERIFY D3:** Test with `changesJson` = `[{"fieldName":"responsibilities","newValue":"Test responsibility text."}]`. A `JD Change Logs` row appears with the **correct old value** populated from the record. The job's Responsibilities field is updated. Review Status is `Draft`.

**Then restore the job record manually** so your test data isn't polluted.

---

# PART E — The agent

*Session 5. ~1 hour.*

## E1. Create the agent

1. Go to `copilotstudio.microsoft.com`, confirm DEV environment.
2. **Create** → choose the **new agent experience** (not classic).
3. Name: `Job Description Intake Agent`
4. Description: `Helps HR team members create new job descriptions and review and update existing ones.`

⚠️ **One-way door:** new-experience agents cannot be converted to classic, and classic agents cannot be converted forward. Your existing HR assistant stays where it is. If they ever need to work together, that's a **Connected agent** relationship, not a merge.

## E2. Move the agent into your solution

⚠️ Do this **immediately**, before adding anything. In the new experience the pre-creation solution picker is gone and the agent lands in the environment's default solution.

1. Go to `make.powerapps.com` → **Solutions** → `HR Job Description Intake`
2. **+ Add existing** → **Chatbot** (or **Agent**)
3. Select `Job Description Intake Agent` → **Add**

**✓ VERIFY E2:** The agent appears in your solution's component list. If it doesn't, everything you build from here will fail to export.

## E3. Instructions

Build tab → instructions editor (the large center pane). Paste:

```
You are the Job Description Intake Agent for City National's HR team.

You help HR team members with exactly two tasks:
1. Creating a brand-new job description for a job title that does not exist yet.
2. Reviewing and updating an existing job description.

Determine which of these two the user needs before doing anything else.

## Where facts come from

All job description content comes from your tools. Never state a job
description field value that did not come from a tool result in this
conversation. If a tool returns an empty field, say the field is empty.
Do not fill it in, infer it, or carry it over from a similar job.

If a tool call fails, tell the user plainly that you could not retrieve
the information and stop. Do not reconstruct data from memory or from
earlier in the conversation.

## Boundaries

- You do not discuss compensation ranges, individual employee performance,
  or hiring decisions. Refer those to the HR Business Partner.
- You do not make changes to job descriptions without explicit user
  confirmation of a summary of those changes.
- If the user asks about anything outside job description intake, say so
  briefly and redirect.

## Tone

Professional and efficient. HR team members use this repeatedly and value
speed. Keep responses tight. Use formatting to make job description fields
scannable. Do not pad with pleasantries.
```

⚠️ Keep this block short. Everything procedural belongs in Skills — that's the design intent of the new experience, and it saves tokens on every turn.

## E4. Add the tools

Components panel (right side) → **Tools** → **+ Add a tool** → **Workflow**.

Add all three: `SearchJobTitles`, `GetJobDescription`, `SaveJobDescriptionDraft`.

If any doesn't appear in the picker, re-check: agent trigger node present, `Respond to the agent` present, async off, published.

For each tool, confirm the **description** field carries the text from Part D. The orchestrator selects tools by description — treat it as functional code.

## E5. Add knowledge

**Knowledge** → **+ Add knowledge** → **SharePoint**.

Point at your JD standards library — writing guidelines, competency dictionary, grade/level definitions.

Give the source a detailed description, e.g.:
```
City National's job description writing standards, competency library,
and job grade and level definitions. Use for guidance on how job
description content should be written and structured. Does not contain
data about specific existing jobs.
```

⚠️ **Do not add the daily report or the Dataverse job table as knowledge.** Job facts come through tools only. That last sentence in the description exists specifically to stop the orchestrator reaching here for job data.

## E6. Memory and Microsoft IQ

- **Memory:** toggle **On**. Useful for remembering a recruiter's department and preferred pace. Note the governance: memories are per-user and private, makers cannot see them, and they're deleted after 28 days of user inactivity. Don't build anything load-bearing on it.
- **Microsoft IQ:** leave **Off** for v1. It pulls org-wide email, calendar, files, and Teams context — exactly the ungrounded surface you're avoiding in a system where field accuracy is the product.

## E7. Conversation starters

Look for the starters/suggested prompts setting and add exactly two:
1. `Update an existing job description`
2. `Create a new job title`

These do a large share of your routing work for free.

---

# PART F — Skills

Skills are Markdown with YAML front matter, loaded on demand based on their description. Author them in a text editor, save as `.md`, and upload — this also means they version cleanly in your GitHub repo.

## F1. `jd-intake-routing.md`

```markdown
---
name: jd-intake-routing
description: Determines whether the user wants to create a brand-new job
  title or update an existing job description, then routes to the correct
  workflow. Use at the start of every conversation and any time the user's
  intent between creating new and updating existing is unclear.
---

# Job description intake routing

## Opening

Open with a brief greeting that presents exactly two paths:

- Creating a new job title
- Updating an existing job description

Do not open with an open-ended "how can I help you?" Early disambiguation
is the entire purpose of this step.

## When the user names a job title without stating the path

Call `SearchJobTitles` with what they said.

- If `matchCount` is 1: tell them you found an existing job with that title
  and job code, and ask whether they are updating it or creating something
  new that happens to have a similar name.
- If `matchCount` is more than 1: list the matches with job code, title,
  and function, then ask which one they mean — or whether none of these
  match and they are creating a new job.
- If `matchCount` is 0: tell them you did not find an existing job matching
  that, and ask whether they would like to create it as a new job title.

## When the user gives a job code directly

Job codes are unique. Call `SearchJobTitles` with the code, confirm the
title that comes back, and proceed to the review workflow.

## Rules

- Never guess the branch silently. Ambiguity gets one clarifying question.
- Once the path is established, hand off and do not re-ask.
- If the user switches paths mid-conversation, follow them, but confirm
  the switch first if there is unsaved review work in progress.
```

## F2. `review-existing-jd.md`

```markdown
---
name: review-existing-jd
description: Guides the conversational review and update of an existing job
  description, from identifying the job through presenting each field for
  review, capturing confirmed changes, and saving the draft. Use whenever
  the user is updating a current job description.
---

# Reviewing an existing job description

## Step 1 — Identify the job

Call `SearchJobTitles` with the user's phrasing.

- One match: confirm the job title and job code in one short line, then
  proceed.
- Multiple matches: present a numbered list showing job code, title, and
  function. Ask the user to pick a number.
- Zero matches: offer the new job creation path.

Always show the job code alongside the title so the user knows precisely
what they are editing.

## Step 2 — Retrieve

Call `GetJobDescription` with the confirmed job code and the user's email.

State the data refresh date in one short line, for example: "This is from
the report refreshed on March 4, 2026."

If `sourceChangedDuringReview` is true, tell the user the source data
changed since a previous review was started and they may be looking at
updated content.

## Step 3 — Present for review, section by section

Do not dump the whole job description as one block. Walk through these
sections in order:

1. **Overview** — job title, function, family, grade, FLSA status, sales
   role flag, reports to
2. **Job summary**
3. **Responsibilities**
4. **Education requirements**
5. **Experience requirements**
6. **Skills and competencies**
7. **Other requirements** — licenses, certifications, physical requirements

After each section ask whether it still looks right or whether they want
to change anything in it.

If a field is empty, say it is empty. Do not skip it silently and do not
fill it in.

The user may jump between sections at any time. Follow them. Do not force
them back into sequence.

## Step 4 — Capture changes

When the user requests a change:

- Restate the proposed new value in full
- Get explicit confirmation before moving on
- Hold the change in the conversation; do not save yet

When the user gives a vague direction such as "make it stronger" or "add
something about Salesforce experience," draft a concrete revision and show
it to them in full. Never save an interpretation the user has not seen
word for word.

If they ask you to rewrite content, follow the job description writing
standards.

## Step 5 — Confirm and save

When the user indicates they are done, present a consolidated summary of
every change as a list showing, for each field: the field name, what it
said before, and what it will say now.

Ask for one final explicit confirmation.

Only after they confirm, call `SaveJobDescriptionDraft` with the session
ID from step 2 and only the fields that changed.

Use these exact field names in `changesJson`:

`jobTitle`, `jobFunction`, `jobFamily`, `grade`, `flsaStatus`,
`isSalesRole`, `reportsTo`, `jobSummary`, `responsibilities`,
`educationRequirements`, `experienceRequirements`, `skillsCompetencies`,
`licensesCertifications`, `physicalRequirements`

Then confirm the save, state the number of changes recorded, and tell them
the draft is saved and awaiting the approval step.

## Hard rules

- Never state a field value that did not come from `GetJobDescription`.
- Never save without explicit confirmation of a change summary.
- Never call `SaveJobDescriptionDraft` more than once per session unless
  the user makes additional changes afterward.
- If any tool fails, say so plainly and stop. Do not proceed on assumed data.
- If the user makes no changes, say so and do not call the save tool.
```

## F3. `jd-writing-standards.md`

```markdown
---
name: jd-writing-standards
description: House style rules for writing job description content —
  responsibility bullet structure, education and experience phrasing,
  inclusive language, and compliance requirements. Use whenever drafting
  or rewriting any job description field content.
---

# Job description writing standards

[Replace this section with City National's actual standards. Suggested
structure below.]

## Responsibilities

- Begin each bullet with a present-tense action verb
- One responsibility per bullet
- Target 6 to 10 bullets
- Avoid internal jargon and system names unless the role requires them

## Education requirements

- State the minimum, not the ideal
- Include an equivalency clause where the role permits it

## Experience requirements

- Express as a range in years plus the domain
- Separate required from preferred

## Sales roles

[Any specific requirements that apply when the sales role flag is set.]

## Language to avoid

[Your banned phrasing list, gendered language, age-coded terms.]

## Required compliance language

[EEO statement and any other mandated text.]
```

## F4. Upload the skills

Components panel → **Skills** → add each `.md` file.

**✓ VERIFY F4:** All three skills are listed and enabled.

## F5. Commit to GitHub

Push all three `.md` files to your repo alongside this manual. Skills being plain Markdown is the reason they version properly — take advantage of it.

---

# PART G — Test

## G1. Preview tab cases

Run each. The Preview window now shows the orchestrator's reasoning before the answer — read it when routing goes wrong.

| # | Input | Expected |
|---|---|---|
| 1 | *(open the conversation)* | Two clear paths offered |
| 2 | `Update an existing job description` | Asks which job |
| 3 | An exact job title | Confirms title + code, retrieves, presents Overview first |
| 4 | A partial title matching several jobs | Numbered disambiguation list |
| 5 | Complete nonsense as a title | Zero matches handled gracefully, offers new-job path |
| 6 | A raw job code | Resolves directly to that job |
| 7 | A job with an empty field | States the field is empty; does not invent content |
| 8 | `change the education requirement to a bachelor's degree` | Restates new value, asks confirmation |
| 9 | `make the responsibilities stronger` | Drafts concrete text and shows it before saving |
| 10 | `actually go back to the summary` | Follows without railroading |
| 11 | Confirm and finish | Consolidated before/after summary, then one confirmation |
| 12 | Confirm the save | Save tool fires once; change log rows correct |
| 13 | Make no changes and end | Does **not** call the save tool |
| 14 | `what's the salary range for this role?` | Declines and redirects per boundaries |

## G2. Grounding test

Ask: `what are the responsibilities for a Chief Unicorn Wrangler?`

**Pass:** the agent reports no match. **Fail:** it invents a plausible job description. If it fails, strengthen the "facts come from tools" language in both the instructions and the review skill.

## G3. Evaluate tab

Convert your best cases from G1 into saved test methods on the Evaluate tab so you can regression-test after every skill edit.

## G4. Activity trace

For any case that misroutes, open the trace and check which tool fired. Wrong tool selection is nearly always a tool **description** problem, not an instruction problem. Fix the description first.

---

# Deployment notes

**Export order.** Publish the agent, then export the solution as **managed** for UAT. Verify the solution contains: 4 tables, 4 environment variables, 4 flows (ingest + 3 tools), 1 agent, and all connection references.

**Environment variables.** Set current values in the target environment during import — sender address, subject filter, admin email. Set `JD Test Mode` to `Yes` in UAT.

**Connection references.** Office 365 Outlook, Excel Online (Business), and Dataverse. Confirm each resolves in the target before the first ingest run.

**Office Script.** The script lives in the host workbook, **not** in the solution. Copy `JD-CSV-Parser-Host.xlsx` to the target environment's SharePoint site manually and re-point the Run script action's file path. This is the one manual step in the pipeline — document it in your runbook.

**Preview status.** The new agent experience is a production-ready preview; skills, memory, and workflows are public preview. Reasonable for an internal HR tool, worth stating plainly to anyone who asks about production readiness.

---

# Backlog for Parts 2 and 3

**Part 2 — New job workflow:** `create-new-jd.md` skill, a `CreateJobDescription` tool writing to the same table with `ReviewStatus = Draft`, and a job code assignment strategy (does the agent generate a provisional code, or does HRIS assign it?).

**Part 3 — Manager approval:** `SubmitJDForApproval` tool that reads `JD_ChangeLog` by session, renders a word-level HTML redline against `BaselineJSON`, emails your manager, sets `ReviewStatus = PendingApproval`, and flips `Submitted` to Yes. Your redline generation work from the other JD agent transplants directly here.
