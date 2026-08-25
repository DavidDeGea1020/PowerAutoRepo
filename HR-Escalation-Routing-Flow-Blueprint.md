# HR ESCALATION ROUTING TOOL
### Assembly Instructions — Model No. `HR-ESC-01`
**Estimated build time:** 90–120 minutes
**Difficulty:** ●●●○○
**Persons required:** 1 (a second person helps for UAT testing)

---

## ⚠️ BEFORE YOU BEGIN

Read all steps before assembling. Do not send live email until Step 14 (Test Mode) is complete.

**Do not:**
- Hard-code email addresses inside the flow. They belong in SharePoint (Part A).
- Let the agent free-type a recipient address. The flow resolves addresses; the agent only names a department.
- Send Employee Relations issue detail in an email body. See Step 10.

---

## 📦 PARTS LIST

| # | Part | Qty | Where it lives |
|---|---|---|---|
| **A** | SharePoint list `HR Email Routing` | 1 | HR SharePoint site |
| **B** | Power Automate flow `HR-Escalation-Route-And-Send` | 1 | Solution (same as agent) |
| **C** | Copilot Studio tool binding | 1 | HR Agent |
| **D** | Agent instruction block | 1 | HR Agent instructions |
| **E** | Shared mailbox connection reference | 1 | Solution |
| **F** | Environment variable `HR_TestMode` | 1 | Solution |
| **G** | Environment variable `HR_TestInbox` | 1 | Solution |

## 🔧 TOOLS REQUIRED

- Power Platform environments: DEV / UAT / QA
- A managed solution container (do not build outside the solution)
- Send-As permission on the HR shared mailbox for the flow's service account

---

# PART A — BUILD THE ROUTING TABLE
*(Steps 1–2)*

## STEP 1 — Create the SharePoint list

Create a list named exactly `HR Email Routing`.

| Internal name | Type | Required | Notes |
|---|---|---|---|
| `Title` | Single line | ✅ | Department name. **Must match** the agent's department vocabulary exactly. |
| `RoutingEmail` | Single line | ✅ | Destination address (distro/shared mailbox preferred over a person). |
| `CCEmail` | Single line | ❌ | Optional, semicolon-delimited. |
| `Keywords` | Multi-line (plain) | ❌ | Semicolon-delimited fallback terms. |
| `Aliases` | Multi-line (plain) | ❌ | Semicolon-delimited alternate department names. |
| `IsSensitive` | Yes/No | ✅ | `Yes` for Employee Relations. |
| `IsActive` | Yes/No | ✅ | Kill switch per row. |
| `IsDefault` | Yes/No | ✅ | Exactly **one** row = Yes (General HR). |
| `SLAHours` | Number | ❌ | Used only in email text. |

> 🔩 **Fitting note:** Keep column internal names free of spaces. Rename via list settings after creation if SharePoint appended `_x0020_`.

## STEP 2 — Populate the six rows

| Title | IsSensitive | IsDefault |
|---|---|---|
| Benefits | No | No |
| Payroll | No | No |
| Talent Acquisition | No | No |
| Development and Culture | No | No |
| Employee Relations | **Yes** | No |
| General HR | No | **Yes** |

Fill `Keywords` generously — this is your safety net when the agent's classification is blank.

Example (Payroll): `paycheck;direct deposit;W-2;W2;overtime;timesheet;garnishment;pay stub;withholding`

✅ **Checkpoint A:** Six rows, all `IsActive = Yes`, exactly one `IsDefault = Yes`.

---

# PART B — ASSEMBLE THE FLOW
*(Steps 3–13)*

Create a **solution-aware** cloud flow named `HR-Escalation-Route-And-Send`.

## STEP 3 — Fit the trigger

**Action:** `When an agent calls the flow` (Copilot Studio trigger)

Add these text inputs — spelling matters, the agent binds by name:

| Input | Type | Required |
|---|---|---|
| `EmployeeName` | Text | Yes |
| `EmployeeEmail` | Text | Yes |
| `SuggestedDepartment` | Text | No |
| `IssueSummary` | Text | Yes |
| `IssueCategoryHint` | Text | No |
| `Urgency` | Text | No |

> 🔩 `SuggestedDepartment` is *suggested*, not final. The flow has the last word.

## STEP 4 — Initialize variables

Add these in order (Power Automate requires all Initialize actions at top level, before branching):

| Name | Type | Initial value |
|---|---|---|
| `varDepartment` | String | *(empty)* |
| `varRoutingEmail` | String | *(empty)* |
| `varCCEmail` | String | *(empty)* |
| `varIsSensitive` | Boolean | `false` |
| `varSLAHours` | Integer | `24` |
| `varMatchMethod` | String | *(empty)* |
| `varStatus` | String | `Failed` |
| `varMessage` | String | *(empty)* |

> 🔩 `varStatus` starts as `Failed` on purpose. It only flips to `Success` after the send confirms. Fail closed, not open.

## STEP 5 — Compose the ticket ID

**Action:** `Compose` → rename to `Ticket ID`

```
concat('HR-', formatDateTime(utcNow(), 'yyyyMMdd'), '-', toUpper(substring(replace(guid(), '-', ''), 0, 5)))
```

Produces: `HR-20260824-A3F91`

## STEP 6 — Normalize the inputs

**Action:** `Compose` → rename to `Clean Department`

```
trim(coalesce(triggerBody()?['text'], ''))
```

> 🔩 Replace `['text']` with the actual dynamic reference for `SuggestedDepartment` from the trigger. Use the dynamic content picker rather than typing it — the trigger uses generated keys.

**Action:** `Compose` → rename to `Search Text`

```
toLower(concat(outputs('Clean_Department'), ' ', coalesce(<IssueCategoryHint>, ''), ' ', coalesce(<IssueSummary>, '')))
```

## STEP 7 — Open the Try scope

**Action:** `Scope` → rename to `TRY — Route and Send`

Everything from Step 8 through Step 12 goes **inside** this scope.

## STEP 8 — Attempt exact department match

**Action:** `Get items` (SharePoint)
- Site / List: `HR Email Routing`
- **Filter Query:**

```
IsActive eq 1 and Title eq 'REPLACE'
```

Build the filter with an expression instead of typing it, so quotes are escaped:

```
concat('IsActive eq 1 and Title eq ''', replace(outputs('Clean_Department'), '''', ''''''), '''')
```

- Top Count: `1`

Rename this action to `Get Exact Match`.

## STEP 9 — Build the three-tier fallback

**Action:** `Condition` → rename to `Exact Match Found?`

Condition: `length(body('Get_Exact_Match')?['value'])` **is greater than** `0`

### ✅ IF YES — Tier 1

Set these variables using `first(body('Get_Exact_Match')?['value'])`:

| Variable | Value expression |
|---|---|
| `varDepartment` | `first(body('Get_Exact_Match')?['value'])?['Title']` |
| `varRoutingEmail` | `first(body('Get_Exact_Match')?['value'])?['RoutingEmail']` |
| `varCCEmail` | `coalesce(first(body('Get_Exact_Match')?['value'])?['CCEmail'], '')` |
| `varIsSensitive` | `if(equals(first(body('Get_Exact_Match')?['value'])?['IsSensitive'], true), true, false)` |
| `varSLAHours` | `int(coalesce(first(body('Get_Exact_Match')?['value'])?['SLAHours'], 24))` |
| `varMatchMethod` | `ExactDepartment` |

### ❌ IF NO — Tier 2 (keyword scan), then Tier 3 (default)

**8.1** `Get items` → rename `Get All Routes`
Filter Query: `IsActive eq 1`, Top Count: `100`

**8.2** `Filter array` → rename `Keyword Matches`
- From: `body('Get_All_Routes')?['value']`
- Advanced mode expression:

```
@greater(length(intersection(
  split(toLower(coalesce(item()?['Keywords'], 'zzzznomatch')), ';'),
  split(outputs('Search_Text'), ' ')
)), 0)
```

> 🔩 **Fitting note:** This matches single-word keywords only. For multi-word phrases (`direct deposit`), add them as separate single tokens too (`deposit`). Simpler and far more reliable than phrase matching in a Filter array.

**8.3** `Condition` → rename `Keyword Match Found?`
`length(body('Keyword_Matches'))` is greater than `0`

- **Yes:** set the variables from `first(body('Keyword_Matches'))`, set `varMatchMethod` = `KeywordFallback`
- **No:** `Filter array` on `body('Get_All_Routes')?['value']` where `item()?['IsDefault']` equals `true`. Set variables from `first(...)`, set `varMatchMethod` = `DefaultRoute`

**8.4** After the condition, add `Condition` → rename `Routing Email Resolved?`
`empty(variables('varRoutingEmail'))` equals `false`
- **If No:** `Terminate` with status `Failed`, message `No active routing destination could be resolved.`

✅ **Checkpoint B:** Every path exits Step 9 with a non-empty `varRoutingEmail`, or terminates loudly.

## STEP 10 — Assemble the email body (sensitivity fork)

**Action:** `Condition` → rename `Is Sensitive Department?`
`variables('varIsSensitive')` is equal to `true`

### ✅ YES — Redacted build

`Compose` → `Email Body`:

```html
<p><strong>Confidential HR Escalation</strong></p>
<p>Ticket ID: <strong>@{outputs('Ticket_ID')}</strong></p>
<p>Department: @{variables('varDepartment')}</p>
<hr>
<p>An employee has submitted a request routed to Employee Relations.</p>
<p><em>Issue details have been intentionally withheld from this notification
due to the sensitive nature of Employee Relations matters.</em></p>
<p>Please contact the employee directly to discuss.</p>
<hr>
<p>Employee: @{<EmployeeName>}<br>
Contact: @{<EmployeeEmail>}<br>
Submitted (UTC): @{formatDateTime(utcNow(), 'yyyy-MM-dd HH:mm')}<br>
Target response: @{variables('varSLAHours')} hours</p>
```

### ❌ NO — Standard build

`Compose` → `Email Body`:

```html
<p><strong>HR Escalation</strong></p>
<p>Ticket ID: <strong>@{outputs('Ticket_ID')}</strong></p>
<p>Department: @{variables('varDepartment')}
(matched via @{variables('varMatchMethod')})</p>
<hr>
<p><strong>Request:</strong></p>
<blockquote>@{replace(coalesce(<IssueSummary>, ''), '<', '&lt;')}</blockquote>
<hr>
<p>Employee: @{<EmployeeName>}<br>
Contact: @{<EmployeeEmail>}<br>
Urgency: @{coalesce(<Urgency>, 'Normal')}<br>
Submitted (UTC): @{formatDateTime(utcNow(), 'yyyy-MM-dd HH:mm')}<br>
Target response: @{variables('varSLAHours')} hours</p>
<p style="color:#888;font-size:11px">Generated by the HR Assistant agent.
Reply directly to the employee.</p>
```

> 🔩 The `replace(..., '<', '&lt;')` prevents user text from breaking your HTML. Do not skip this piece.

> ⚠️ **Both branches must produce an action named `Email Body`.** Rename them identically so downstream references resolve on whichever branch runs. If Power Automate blocks duplicate names, name them `Email_Body_Sensitive` / `Email_Body_Standard` and reference with:
> `coalesce(outputs('Email_Body_Sensitive'), outputs('Email_Body_Standard'))`

## STEP 11 — Resolve the recipient (Test Mode gate)

`Compose` → `Final Recipient`:

```
if(equals(toLower(string(<HR_TestMode env var>)), 'true'), <HR_TestInbox env var>, variables('varRoutingEmail'))
```

> 🔩 This is the single most important safety part in the kit. With `HR_TestMode = true`, every email lands in your test inbox regardless of routing. Ship UAT with it `true`, flip to `false` only in PROD.

## STEP 12 — Send the emails

**12.1 — Departmental notification**
**Action:** `Send an email from a shared mailbox (V2)`
- Original Mailbox Address: HR shared mailbox
- To: `outputs('Final_Recipient')`
- CC: `variables('varCCEmail')`
- Subject: `concat('[', outputs('Ticket_ID'), '] ', variables('varDepartment'), ' — ', coalesce(<Urgency>, 'Normal'))`
- Body: the composed HTML
- **Is HTML:** On (Advanced options)

**12.2 — Employee confirmation**
`Condition` → `Valid Employee Email?`
`contains(coalesce(<EmployeeEmail>, ''), '@')` equals `true`

If yes, second `Send an email from a shared mailbox (V2)`:
- To: employee's email
- Subject: `concat('We received your request — ', outputs('Ticket_ID'))`
- Body:

```html
<p>Hi @{<EmployeeName>},</p>
<p>Your request has been sent to the <strong>@{variables('varDepartment')}</strong> team.</p>
<p>Your ticket ID is <strong>@{outputs('Ticket_ID')}</strong> — please reference it in any follow-up.</p>
<p>Expected response time: @{variables('varSLAHours')} hours.</p>
<p>— HR Assistant</p>
```

**12.3** `Set variable` → `varStatus` = `Success`
**12.4** `Set variable` → `varMessage` = `concat('Routed to ', variables('varDepartment'), '. Ticket ', outputs('Ticket_ID'), '.')`

## STEP 13 — Fit the Catch scope and response

**13.1** Add `Scope` → `CATCH — Handle Failure`, placed **after** the Try scope.
- Configure run after: ✅ has failed, ✅ is skipped, ✅ has timed out (uncheck *is successful*)
- Inside: `Set variable` `varStatus` = `Failed`, and `varMessage` = `Escalation could not be completed. Please contact HR directly.`

**13.2** `Respond to the agent` — configure run after Catch: ✅ is successful **and** ✅ is skipped.

Outputs:

| Output name | Type | Value |
|---|---|---|
| `Status` | Text | `variables('varStatus')` |
| `TicketID` | Text | `outputs('Ticket_ID')` |
| `RoutedDepartment` | Text | `variables('varDepartment')` |
| `MatchMethod` | Text | `variables('varMatchMethod')` |
| `ResultMessage` | Text | `variables('varMessage')` |

> ⚠️ **Do not output `varRoutingEmail`.** The agent has no reason to see or repeat internal addresses, and anything returned to the agent can surface in the chat transcript.

✅ **Checkpoint C:** Flow saves with zero errors. Run a manual test — it should complete even with a garbage department string, landing on `DefaultRoute`.

---

# PART C — MOUNT TO THE AGENT
*(Steps 14–16)*

## STEP 14 — Test Mode dry run

1. Set `HR_TestMode` = `true`, `HR_TestInbox` = your address.
2. Run the flow manually against this matrix:

| # | SuggestedDepartment | IssueSummary | Expect |
|---|---|---|---|
| 1 | `Payroll` | "My check was short" | ExactDepartment → Payroll |
| 2 | *(blank)* | "My direct deposit failed" | KeywordFallback → Payroll |
| 3 | `Benfits` *(typo)* | "dental coverage question" | KeywordFallback → Benefits |
| 4 | `Marketing` | "asdfgh" | DefaultRoute → General HR |
| 5 | `Employee Relations` | "My manager is harassing me" | Sensitive → body redacted ✅ |
| 6 | `Payroll` | `<script>alert(1)</script>` | Escapes safely, no broken HTML |

**Test 5 is mandatory.** Open the email and confirm the issue text does not appear anywhere.

## STEP 15 — Add the tool in Copilot Studio

Add the flow as a tool. Set input fill types:
- `EmployeeName`, `EmployeeEmail` → from system/user context where available, else model-filled
- `SuggestedDepartment` → **model-filled**, with description:

> *The HR department best suited to handle this request. Must be exactly one of: Benefits, Payroll, Talent Acquisition, Development and Culture, Employee Relations, General HR. If you are not confident, leave this blank.*

- `IssueSummary` → model-filled: *A concise, factual summary of the employee's request in their own words.*

## STEP 16 — Install the instruction block

Add to agent instructions:

```
ESCALATION RULES

- Attempt to answer from knowledge sources first. Only escalate when the
  employee asks to contact HR, or when no knowledge source resolves the issue.
- Offer escalation once per conversation. If the employee declines, do not
  offer again unless they raise a new issue.
- Before calling the escalation tool, confirm with the employee:
  "I can send this to the [Department] team on your behalf — shall I?"
  Do not call the tool without an affirmative reply.
- Set SuggestedDepartment only when confident. Leave it blank otherwise;
  the tool has its own fallback routing.
- Never state, guess, or ask for an HR email address. The tool resolves
  recipients internally.
- After the tool returns, give the employee the TicketID and the
  RoutedDepartment. If Status is Failed, apologize and direct them to the
  HR portal — do not retry automatically.
- For Employee Relations matters, do not repeat the details back at length.
  Acknowledge briefly and confirm the escalation.
```

Pair with your existing `Global.varEmailOfferMade` variable to hard-enforce the single-offer rule at the topic level.

---

# 🔍 TROUBLESHOOTING

| Symptom | Likely cause | Fix |
|---|---|---|
| `Get items` returns 0 for a valid department | Filter query quoting | Verify with the expression form in Step 8; check for trailing whitespace via `trim()` |
| `Column 'IsSensitive' does not exist` | SharePoint internal name mismatch | Check list settings URL for `Field=` value; may be `IsSensitive0` |
| Email arrives as raw HTML | `Is HTML` off | Advanced options on the send action |
| Flow succeeds, agent shows nothing | Response action skipped | Confirm run-after on `Respond to the agent` includes *is skipped* |
| `AIModelNotFound` on import to UAT | Missing dependency | This flow adds no AI model dependency — if it appears, it's from another component in the solution |
| Sensitive detail leaked into email | Wrong branch fired | Confirm `IsSensitive` is Yes on the SharePoint row, and that the condition compares to boolean `true`, not string `"true"` |

# 📤 DEPLOYMENT ORDER (DEV → UAT → QA)

1. Add flow + connection references to the **unmanaged** solution in DEV.
2. Confirm `HR_TestMode` and `HR_TestInbox` are **environment variables**, not literals.
3. Create the `HR Email Routing` list in the target environment **before** import, and update the SharePoint site/list references (use environment variables for site + list ID to avoid re-binding by hand each time).
4. Export managed → import to UAT → set env var values at import time.
5. Keep `HR_TestMode = true` in UAT and QA. Flip to `false` only in PROD, and only after Checkpoint C passes there.

---

*Keep these instructions for future reference. Spare parts (Keywords, Aliases) can be updated in SharePoint without redeploying the flow — that is the entire point of Part A.*
