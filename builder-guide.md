# HR Assistant — Email Flow

Assembly instructions.

Estimated time: 60–90 minutes.
Do not skip Part A. The flow will not work without it.

---

## Parts included

| # | Part | Qty |
|---|------|-----|
| A | SharePoint list — `HR Email Routing` | 1 |
| B | Cloud flow — `HR Assistant - Send Routed Email` | 1 |
| C | Copilot Studio tool registration | 1 |
| D | Confirmation topic | 1 |
| E | Solution packaging check | 1 |

## Tools required

- Access to your **Dev** environment (Power Apps maker portal)
- Your existing HR agent solution
- Your existing Office 365 Outlook **connection reference**
- The SharePoint site hosting your HR knowledge sources

## Do not

- Do not build any part of this in the **QA** environment.
- Do not create the flow from the Power Automate home screen.
- Do not proceed past a **CHECKPOINT** if it fails.

---

# PART A — SharePoint routing list

### A1

Go to your HR SharePoint site.
**New → List → Blank list.**
Name it `HR Email Routing`.

### A2

Add four columns. Use exactly these display names.

| Display name | Type | Settings |
|---|---|---|
| EmailAddress | Single line of text | Required: Yes |
| DisplayName | Single line of text | Required: Yes |
| RedactQuestion | Yes/No | Default: No |
| Active | Yes/No | Default: Yes |

The built-in **Title** column becomes your department key. Leave it alone.

### A3 — CHECKPOINT: internal names

**List settings → Columns →** click `EmailAddress`.
Look at the browser address bar. Find `Field=`.

It must read `Field=EmailAddress`.

If it reads `Field=EmailAddress0` or `Field=_x0045_mail`, delete the column
and recreate it. A mismatched internal name makes your filter return nothing,
silently, forever.

Repeat for `RedactQuestion` and `Active`.

### A4

Add one row per department. Six rows.

| Title | EmailAddress | DisplayName | RedactQuestion | Active |
|---|---|---|---|---|
| Benefits | benefits@bank.com | Benefits Team | No | Yes |
| Payroll | payroll@bank.com | Payroll Team | No | Yes |
| TalentAcquisition | talent@bank.com | Talent Acquisition | No | Yes |
| DevelopmentCulture | devculture@bank.com | Development and Culture | No | Yes |
| EmployeeRelations | er@bank.com | Employee Relations | **Yes** | Yes |
| GeneralHR | hr@bank.com | HR Service Desk | No | Yes |

Replace the addresses with your real ones.
`Title` values must match your classifier's output strings **character for
character**, including case.

---

# PART B — The flow

## B1 — Create it in the right place

**make.powerapps.com** → **Solutions** → open your HR agent solution.

**+ New → Automation → Cloud flow → Instant.**

Name: `HR Assistant - Send Routed Email`

> If you started anywhere else, stop and start over here. This step is the
> entire reason the flow will deploy cleanly.

## B2 — Trigger

Search the trigger list for **When Copilot Studio calls a flow**.
(Older tenants label this *When Power Virtual Agents calls a flow*.)

Add eight inputs. All type **Text**. Order matters for your own sanity only.

| # | Input name |
|---|---|
| 1 | DepartmentKey |
| 2 | RecipientEmail |
| 3 | EmployeeQuestion |
| 4 | AgentAnswer |
| 5 | MessageBody |
| 6 | Subject |
| 7 | EmployeeEmail |
| 8 | EmployeeName |

## B3 — Variables

Add four **Initialize variable** actions, in order.

| Name | Type | Initial value |
|---|---|---|
| varRecipient | String | *(leave empty)* |
| varDisplayName | String | *(leave empty)* |
| varRedact | Boolean | false |
| varStatus | String | Rejected |

`varStatus` starts at `Rejected` on purpose. It only becomes `Sent` if the
flow actually sends. Fail closed.

## B4 — Normalize the address

Add a **Compose**. Rename it `normalizedEmail`.

```
toLower(trim(coalesce(triggerBody()?['text_1'], '')))
```

> The `text_1` suffix depends on input order. Click the dynamic content
> picker and select **RecipientEmail** rather than typing this by hand —
> the designer inserts the correct reference.

## B5 — Lookup 1: exact address match

Add **Get items** (SharePoint).

- Site Address: your HR site
- List Name: `HR Email Routing`
- Filter Query:

```
EmailAddress eq '@{outputs('normalizedEmail')}' and Active eq 1
```

- Top Count: `1`

Rename this action `Get items - by address`.

## B6 — Branch on the result

Add a **Condition**. Rename it `Address matched?`

Left side:
```
length(body('Get_items_-_by_address')?['value'])
```
Operator: **is greater than**
Right side: `0`

### B6-YES — address is on the allowlist

Inside the **If yes** branch, add three **Set variable** actions:

| Variable | Value |
|---|---|
| varRecipient | `first(body('Get_items_-_by_address')?['value'])?['EmailAddress']` |
| varDisplayName | `first(body('Get_items_-_by_address')?['value'])?['DisplayName']` |
| varRedact | `first(body('Get_items_-_by_address')?['value'])?['RedactQuestion']` |

### B6-NO — fall back to the department key

Inside the **If no** branch, add a second **Get items**.
Rename it `Get items - by department`.

Filter Query:
```
Title eq '@{triggerBody()?['text']}' and Active eq 1
```
*(Use dynamic content to select **DepartmentKey**.)*

Top Count: `1`

Then add a nested **Condition**, `Department matched?`:

```
length(body('Get_items_-_by_department')?['value'])
```
is greater than `0`

- **If yes** → same three Set variable actions as B6-YES, but reading from
  `Get_items_-_by_department`.
- **If no** → leave empty. `varStatus` stays `Rejected` and nothing sends.

## B7 — CHECKPOINT: the safety property

Read B5 and B6 back to yourself and confirm:

> There is no path through this flow where an email address supplied by the
> agent reaches the Send action without first appearing as a row in
> `HR Email Routing`.

If you can find one, you have wired something to the trigger input instead of
`varRecipient`. Fix it now.

## B8 — Build the question block

Add a **Condition** below the whole B6 structure. Rename it `Redact?`

Left: `varRedact` — is equal to — `true`

Add a **Compose** in *each* branch, both named `questionBlock`
(one per branch, so only one runs).

**If yes:**
```
<p><em>An employee has requested contact regarding a confidential
matter. Details withheld.</em></p>
```

**If no:**
```html
<p><strong>Employee's question:</strong><br>
@{triggerBody()?['text_2']}</p>
<p><strong>Answer provided by the HR Assistant:</strong><br>
@{triggerBody()?['text_3']}</p>
```
*(Select **EmployeeQuestion** and **AgentAnswer** from dynamic content.)*

## B9 — Assemble the body

Add a **Compose**. Rename it `emailBody`.

```html
<p>Submitted by: @{triggerBody()?['text_7']}
(@{triggerBody()?['text_6']})</p>

<p>@{triggerBody()?['text_4']}</p>

@{outputs('questionBlock')}

<hr>
<p style="font-size:11px;color:#666666">
Sent via the HR Assistant on behalf of the employee named above.
Reply to this message to reach them directly.</p>
```

*(Dynamic content: EmployeeName, EmployeeEmail, MessageBody.)*

Note the shape: the AI's `MessageBody` occupies exactly one paragraph.
The identity line, the question block, and the footer are all yours.

## B10 — Send

Add **Send an email (V2)** (Office 365 Outlook).

When prompted for a connection, select your **existing connection
reference**. If the picker only shows a personal connection with no
reference option, you built this flow outside the solution. Go back to B1.

| Field | Value |
|---|---|
| To | `varRecipient` |
| Subject | `[HR Assistant] @{triggerBody()?['text_5']}` |
| Body | `outputs('emailBody')` |
| Reply To | `triggerBody()?['text_6']` *(EmployeeEmail)* |
| Is HTML | Yes *(Advanced options)* |

**To must be `varRecipient`.** Not the trigger input. This is the one field
where a wrong drag defeats the entire design.

Directly after the Send, add **Set variable** → `varStatus` → `Sent`.

## B11 — Respond

Add **Respond to Copilot Studio** as the final action, outside all branches.

| Output | Type | Value |
|---|---|---|
| Status | Text | `varStatus` |
| ResolvedRecipient | Text | `varDisplayName` |
| Message | Text | *(see below)* |

For `Message`:
```
if(equals(variables('varStatus'), 'Sent'),
   concat('Your message was sent to ', variables('varDisplayName'), '.'),
   'I could not verify that recipient as an official HR address, so nothing was sent.')
```

Save.

## B12 — CHECKPOINT: test both paths

**Test → Manually.**

**Run 1 — valid.** Enter a real address from your list. Confirm:
`Status = Sent`, and the mail arrives with a working Reply-To.

**Run 2 — invalid.** Enter `notreal@bank.com`, DepartmentKey blank.
Confirm: `Status = Rejected`, Send action **skipped**, no mail.

**Run 3 — redaction.** DepartmentKey `EmployeeRelations`, RecipientEmail
blank. Confirm the question text does not appear in the received mail.

All three must pass before Part C.

---

# PART C — Register as a tool

## C1

Copilot Studio → your agent → **Tools → + Add a tool → Flow**.
Select `HR Assistant - Send Routed Email`.

## C2 — Tool description

This is what the orchestrator reads when deciding to fire. Paste:

> Sends an email to an HR department on the employee's behalf. Use this when
> your answer tells the employee to contact a specific HR department or
> address, and the employee has confirmed they want you to reach out for
> them. Only official HR addresses are permitted; unrecognized addresses are
> rejected and no email is sent.

## C3 — Input configuration

For each input, set the fill method. **This is the step that reproduces the
Outlook tool's generative behavior.**

| Input | Fill method | Description to enter |
|---|---|---|
| DepartmentKey | Dynamically fill with AI | One of exactly: Benefits, Payroll, TalentAcquisition, DevelopmentCulture, EmployeeRelations, GeneralHR. |
| RecipientEmail | Dynamically fill with AI | The HR email address referenced in the answer you just gave. Must be an official HR department address. |
| EmployeeQuestion | Dynamically fill with AI | The employee's original question, verbatim. |
| AgentAnswer | Dynamically fill with AI | The answer you just gave the employee, verbatim. |
| MessageBody | Dynamically fill with AI | A short, professional message explaining what the employee needs. Do not include email addresses, phone numbers, or contact details. |
| Subject | Dynamically fill with AI | A brief subject line, under 10 words, describing the request. |
| EmployeeEmail | **Custom value** | `System.User.Email` |
| EmployeeName | **Custom value** | `System.User.DisplayName` |

The last two must be **Custom value**, not AI-filled. Identity comes from the
authenticated session, never from the model.

---

# PART D — Confirmation topic

The tool sends. The topic asks permission first.

## D1

Copilot Studio → **Topics → + Add a topic → From blank.**
Name: `Confirm HR Contact`.

## D2

Trigger: **When an event occurs** — or leave it callable and invoke it from
your instructions. Do not give it phrase triggers; the agent decides when to
offer, not the employee.

## D3

Add a **Question** node:

> Would you like me to email @{DisplayName} on your behalf? I'll include your
> question and a short summary of what you need.

Type: **Multiple choice** → `Yes, send it` / `No thanks`

## D4

On `Yes` → call the tool.
On `No` → Message node: *"No problem. Let me know if you need anything else."*

## D5

After the tool call, add a **Condition** on the returned `Status`:

- `Sent` → *"Done — I've sent that to @{ResolvedRecipient}."*
- anything else → return the flow's `Message`, then route into your existing
  manual department selection fallback.

## D6 — Agent instructions

Add to your agent-level instructions:

> When your answer directs the employee to contact a specific HR department,
> offer to send that message for them using the Confirm HR Contact topic. Do
> not send anything without explicit confirmation. Never invent an email
> address — only offer contact when the address came from a knowledge source.

---

# PART E — Package for deployment

Do these in order. This is the part that failed last time.

### E1
Publish the agent in Dev. Wait for it to complete.

### E2
Power Apps → Solutions → your solution → find the agent → **⋯ → Advanced →
+ Add required objects.**

### E3 — CHECKPOINT: component list

Open the solution's component list. Confirm all four are present:

- [ ] The agent
- [ ] `HR Assistant - Send Routed Email` (cloud flow)
- [ ] The Office 365 Outlook connection reference
- [ ] The `Confirm HR Contact` topic

If the flow or the connection reference is missing, the export will not
contain it. Re-run E2.

### E4
Publish the agent again. Export.

### E5
Import to QA. Bind the Outlook connection when prompted. Publish the agent
in QA.

### E6 — CHECKPOINT: QA

In QA, open the flow and confirm the SharePoint site address points at a
**QA-reachable** site. If Dev and QA share one SharePoint site, you're fine.
If not, this is the one hardcoded value that needs an environment variable —
convert the Site Address and List Name to environment variables in Dev and
redeploy.

### E7
Run test 2 from B12 in QA. Confirm an unrecognized address is still rejected
there. Allowlists that only work in Dev are worse than no allowlist.

---

## Troubleshooting

**Filter returns nothing, no error.**
Internal column name mismatch. Go back to A3.

**`Active eq 1` throws a filter error.**
Some tenants want `Active eq true`. Try both.

**Tool never fires.**
Tool description too vague, or the agent isn't reaching a "contact someone"
answer. Test by asking a question whose knowledge-source answer names a
department.

**Emails arrive from a service account, replies go nowhere.**
Reply To wasn't set. See B10.

**Dependency error on import.**
Something isn't in the solution. E3.
