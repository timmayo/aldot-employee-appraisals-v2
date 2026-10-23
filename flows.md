# Power Automate flows: build guide

Four cloud flows in the same solution as the app, as built in the maker portal (GCC). Run them under a dedicated service account (not a person's account) so they keep working when people leave.

Connectors: Microsoft Dataverse, Approvals, Office 365 Outlook. Confirm all three are allowed together by the GCC DLP policy for the environment.

| Flow | Trigger | Purpose |
|---|---|---|
| 1. Appraisal - Stage notifications | Stage changes | Emails the next person at each stage |
| 2. Appraisal - Daily scheduler | Daily recurrence | Advances stages by due date, sends reminders, prompts the ASA for next year |
| 3. Appraisal - Pre signatures | Stage = Pre - Awaiting Signatures | Reviewing supervisor (and optionally employee) approval, then completes the pre-appraisal |
| 4. Appraisal - Annual signatures | Stage = Annual - Awaiting Signatures | Reviewing supervisor then employee approval, completes the annual, creates next year's row |

## Before you build

1. **Service account.** Give it the **Appraisal ASA** security role plus create/read/write on all seven appraisal tables (see README, Security roles) and read on the Users table. It also needs a mailbox, or send-as rights on a shared mailbox, for Send an email (V2).
2. **Environment variables** (solution, New, More, Environment variable):

| Display name | Data type | Value |
|---|---|---|
| Appraisal App URL | Text | The play link of the canvas app |
| ASA Team Mailbox | Text | A shared mailbox the ASAs read |

   Environment variables do not appear in Dynamic content. Use an expression. The name is the display name followed by the schema name in parentheses; read the schema name on the variable:

```
parameters('Appraisal App URL (aldot2_AppraisalAppURL)')
parameters('ASA Team Mailbox (aldot2_ASATeamMailbox)')
```

   The schema names above are examples; use yours.

3. **Logical names are lowercase.** The maker portal's column list shows the schema name (for example `aldot2_Stage`). Flows need the logical name: the same text in lowercase (`aldot2_stage`). Lookup columns are `_aldot2_employee_value`, `_aldot2_ratingsupervisor_value` and `_aldot2_reviewingsupervisor_value`. Confirm each name under Tables, the table, Columns, the column, Advanced options, Logical name. Names used in this guide:

| Display name | Logical name |
|---|---|
| Stage | `aldot2_stage` |
| Appraisal (primary key) | `aldot2_appraisalid` |
| Employee, Rating Supervisor, Reviewing Supervisor | `_aldot2_employee_value`, `_aldot2_ratingsupervisor_value`, `_aldot2_reviewingsupervisor_value` |
| Name (primary column) | `aldot2_name` |
| Appraisal Year Start / End | `aldot2_appraisalyearstart`, `aldot2_appraisalyearend` |
| Mid Due Date, Annual Due Date | `aldot2_midduedate`, `aldot2_annualduedate` |
| Position Title, Division | `aldot2_positiontitle`, `aldot2_division` |
| Performance Score | `aldot2_performancescore` |
| Pre Requires Employee Signature, Mid Offset Months (Appraisal Settings) | `aldot2_prerequiresemployeesignature`, `aldot2_midoffsetmonths` |

4. **Stage choice values.** These are the values in this environment (Appraisals, Columns, Stage). Flows compare the number, not the label.

| Stage | Value |
|---|---|
| Pre - ASA Setup | 867720000 |
| Pre - Supervisor Review | 867720001 |
| Pre - Awaiting Signatures | 867720002 |
| Pre - Complete | 867720003 |
| Mid - Open | 867720004 |
| Mid - Complete | 867720005 |
| Annual - Supervisor Rating | 867720006 |
| Annual - Awaiting Signatures | 867720007 |
| Annual - Complete | 867720008 |

5. **Stage trigger (flows 1, 3 and 4).** Trigger **When a row is added, modified or deleted** (Microsoft Dataverse):

| Parameter | Value |
|---|---|
| Change type | Modified |
| Table name | Appraisals |
| Scope | Organization |
| Select columns | `aldot2_stage` |

   Select columns makes the flow run only on requests that include Stage. It fires even when the value did not change, so flows 3 and 4 also have a trigger condition. Lookup columns cannot be used in Select columns. The trigger returns the choice as a number only; the formatted label is not in its outputs. Read it with `triggerOutputs()?['body/aldot2_stage']`.

6. **The three people (flows 1, 3 and 4).** Start each with three **Get a row by ID** actions (Dataverse), Table name **Users**:

| Action name | Row ID (Expression tab) |
|---|---|
| Get employee | `triggerOutputs()?['body/_aldot2_employee_value']` |
| Get rater | `triggerOutputs()?['body/_aldot2_ratingsupervisor_value']` |
| Get reviewer | `triggerOutputs()?['body/_aldot2_reviewingsupervisor_value']` |

   Email: `outputs('Get_rater')?['body/internalemailaddress']` (Dynamic content name: Primary Email). Name: `outputs('Get_employee')?['body/fullname']`.

## Rules learned while building

- **Action names in expressions.** Spaces become underscores and every other character stays. `Get rater - annual` is `Get_rater_-_annual`. Avoid hyphens in the name of any action that an expression refers to (Get, Compose, Approve actions). Names that nothing refers to, such as `Update a row - mid open`, can contain hyphens.
- **Unique names.** A flow cannot have two actions with the same name. A second `Days to due` or `Get rater` gets a `_2` suffix, so give each its own name (`Get rater mid`, `Get rater annual`).
- **Rename with the three dots, Rename.** If a save fails with "action(s) X referenced by inputs in action Y are not defined", the expression in Y uses a name that does not match the action's real name.
- **Condition with an expression.** The condition card has no expression mode. Put the expression in the left box (Expression tab), leave the operator as is equal to, and put `true` in the right box (Expression tab).
- **Lookup fields in Add a new row** take a path expression: `concat('aldot2_appraisals(', triggerOutputs()?['body/aldot2_appraisalid'], ')')` and `concat('systemusers(', outputs('Get_reviewer')?['body/systemuserid'], ')')`. If a lookup fails, check the table's plural logical name. Choice fields (Phase, Role, Outcome, Stage, Cycle Type) are set from their dropdowns.
- **Filter rows.** Date-only columns are compared without quotes (`aldot2_midduedate le 2026-10-23`). A filter that mixes text and expressions is more reliable built as one expression with `concat(...)`. If a filter shows a syntax error, rebuild it that way.
- **Dates.** Date-only columns return `2027-01-03` style text. Write them with `formatDateTime(..., 'yyyy-MM-dd')`.
- **Approvals.** Outcome values are `Approve` and `Reject`. Approver comments are `first(outputs('<action>')?['body/responses'])?['comments']`.
- **Status changes by a flow trigger flow 1 again.** That is intended: it sends the next notification. Flows 3 and 4 ignore other stages through their trigger conditions.

---

## 1. Appraisal - Stage notifications

Trigger as in step 5. No trigger condition.

```
When a row is added, modified or deleted
├─ Get employee
├─ Get rater
├─ Get reviewer
└─ Switch   (On: triggerOutputs()?['body/aldot2_stage'])
   ├─ Case 867720001   Send an email (V2) - to rater
   ├─ Case 867720003   Send an email (V2) - to ASA mailbox
   ├─ Case 867720004   Send an email (V2) - to rater
   ├─ Case 867720006   Send an email (V2) - to rater
   └─ Case 867720008   Send an email (V2) - to employee, rater, ASA mailbox
```

| Case | Stage | To | Subject | Body |
|---|---|---|---|---|
| 867720001 | Pre - Supervisor Review | rater email | Pre-appraisal ready: [employee name] | The ASA has prepared the pre-appraisal. Review it with the employee and sign in the app. |
| 867720003 | Pre - Complete | ASA Team Mailbox | Pre-appraisal submitted: [employee name] | Signed by the reviewing supervisor. The mid-appraisal due date is set. |
| 867720004 | Mid - Open | rater email | Mid-appraisal open: [employee name] | The mid-appraisal is open. Hold the session and complete it in the app. |
| 867720006 | Annual - Supervisor Rating | rater email | Annual appraisal open: [employee name] | Rate the responsibilities, complete the disciplinary section and submit. |
| 867720008 | Annual - Complete | employee, rater (semicolon separated) and ASA Team Mailbox | Annual appraisal signed: [employee name] | All signatures are in. Next year's pre-appraisal has been created for the ASA to fill in. |

Put the app link at the end of every body. There is no email for Pre - Awaiting Signatures because Approvals emails the approver itself.

---

## 2. Appraisal - Daily scheduler

Trigger: **Scheduled cloud flow**, every 1 day, at an off-hours time in your time zone.

```
Recurrence
├─ Init varToday
├─ Init varIn7
├─ List rows - mid due
│   └─ Each mid due
│       └─ Update a row - mid open
├─ List rows - annual due
│   └─ Each annual due
│       └─ Update a row - annual open
├─ List rows - mid open
│   └─ Each mid open
│       ├─ Days to due mid
│       └─ Reminder due - mid
│           └─ If yes: Get rater mid, Send an email (V2) - mid reminder
├─ List rows - annual open
│   └─ Each annual open
│       ├─ Days to due annual
│       └─ Reminder due - annual
│           └─ If yes: Get rater annual, Send an email (V2) - annual reminder
└─ List rows - year ending in 7 days
    └─ Each year ending
        ├─ List next year row
        └─ No next year row?
            └─ If yes: Get employee, Send an email (V2) - create next year
```

**Variables.** `varToday` (String) = `formatDateTime(utcNow(), 'yyyy-MM-dd')`. `varIn7` (String) = `formatDateTime(addDays(utcNow(), 7), 'yyyy-MM-dd')`.

**Advance to Mid.** List rows (Appraisals), Filter rows:
```
aldot2_stage eq 867720003 and aldot2_midduedate le @{variables('varToday')}
```
Apply to each over `value`. Update a row: Table Appraisals, Row ID `items('Each_mid_due')?['aldot2_appraisalid']`, Stage = Mid - Open. Flow 1 emails the rater.

**Advance to Annual.** Filter rows `aldot2_stage eq 867720005 and aldot2_annualduedate le @{variables('varToday')}`. Update a row: Row ID `items('Each_annual_due')?['aldot2_appraisalid']`, Stage = Annual - Supervisor Rating.

**Reminders (mid and annual are the same shape).**

| | Mid | Annual |
|---|---|---|
| List rows filter | `aldot2_stage eq 867720004` | `aldot2_stage eq 867720006` |
| Loop name | Each mid open | Each annual open |
| Compose name | Days to due mid | Days to due annual |
| Due date column | `aldot2_midduedate` | `aldot2_annualduedate` |
| Lookup name | Get rater mid | Get rater annual |

Compose (mid shown; use `items('Each_annual_open')` and the annual column for annual):
```
div(sub(ticks(items('Each_mid_open')?['aldot2_midduedate']), ticks(utcNow('yyyy-MM-dd'))), 864000000000)
```
Condition, expression in the left box, operator is equal to, right box `true`. A reminder is sent 14 days out, 7 days out, then every 7 days once overdue:
```
or(equals(outputs('Days_to_due_mid'), 14), equals(outputs('Days_to_due_mid'), 7), and(less(outputs('Days_to_due_mid'), 0), equals(mod(outputs('Days_to_due_mid'), 7), 0)))
```
If yes: **Get a row by ID** (Users), Row ID `items('Each_mid_open')?['_aldot2_ratingsupervisor_value']`. Then **Send an email (V2)** to `outputs('Get_rater_mid')?['body/internalemailaddress']`, subject `Reminder: mid-appraisal due ` + the due date, body with the appraisal name (`items('Each_mid_open')?['aldot2_name']`), the due date and the app link.

A row opened by this same run shows 0 days to due, so no reminder is sent until a later run; the Flow 1 "Mid - Open" email covers day one.

**New appraisal year (ASA).** List rows (Appraisals), Filter rows `aldot2_appraisalyearend eq @{variables('varIn7')}`. For each row, **List next year row** (Appraisals, Row count 1), Filter rows set as an expression:
```
concat('_aldot2_employee_value eq ', items('Each_year_ending')?['_aldot2_employee_value'], ' and aldot2_appraisalyearstart eq ', formatDateTime(addDays(items('Each_year_ending')?['aldot2_appraisalyearend'], 1), 'yyyy-MM-dd'))
```
Condition `No next year row?`: `length(outputs('List_next_year_row')?['body/value'])` is equal to `0`. If yes: Get employee (Users, Row ID `items('Each_year_ending')?['_aldot2_employee_value']`), then Send an email (V2) to the ASA Team Mailbox: "Create next year's appraisal: [employee name]".

---

## 3. Appraisal - Pre signatures

Trigger as in step 5, plus a **trigger condition** (trigger, three dots, Settings, Trigger conditions):
```
@equals(triggerOutputs()?['body/aldot2_stage'], 867720002)
```
The rating supervisor already signed in the app (a row in Appraisal Signatures).

```
When a row is added, modified or deleted
├─ Get employee
├─ Get rater
├─ Get reviewer
├─ Get settings                      (List rows, Appraisal Settings, Row count 1)
├─ Approve reviewer
└─ Reviewer approved
   ├─ If no
   │   ├─ Add a new row - reviewer rejected
   │   ├─ Update a row - reviewer rejected      (Stage = Pre - Supervisor Review)
   │   ├─ Send an email (V2) - reviewer rejected
   │   └─ Terminate - reviewer rejected         (Cancelled)
   └─ If yes
       ├─ Add a new row - approved
       ├─ Employee signature                    (setting is true?)
       │   ├─ If yes
       │   │   ├─ Approve employee
       │   │   └─ Employee approved
       │   │       ├─ If yes: Add a new row - employee approved
       │   │       └─ If no:  Add a new row - employee rejected
       │   │                  Update a row - employee rejected
       │   │                  Send an email (V2) - employee rejected
       │   │                  Terminate - employee rejected
       │   └─ If no: (empty)
       └─ Update a row - pre-appraisal complete   (after the Employee signature condition)
```

**Get settings:** `first(outputs('Get_settings')?['body/value'])` is the settings row.

**Approve reviewer** (Start and wait for an approval):

| Field | Value |
|---|---|
| Approval type | Approve/Reject - First to respond |
| Title | `Sign pre-appraisal: ` + employee name |
| Assigned to | `outputs('Get_reviewer')?['body/internalemailaddress']` |
| Details | Review the pre-appraisal in the app, then approve or reject. |
| Item link | `parameters('Appraisal App URL (your schema name)')` |
| Item link description | Open the appraisal app |

**Reviewer approved** condition: the Outcome token from Approve reviewer is equal to `Approve`.

**Appraisal Signatures row** (Add a new row, Table Appraisal Signatures), reviewer shown:

| Field | Value |
|---|---|
| Name | `Pre - Reviewing Supervisor` |
| Appraisal (Appraisals) | `concat('aldot2_appraisals(', triggerOutputs()?['body/aldot2_appraisalid'], ')')` |
| Phase | Pre |
| Role | Reviewing Supervisor |
| Outcome | Approved or Rejected (by branch) |
| Signer (Users) | `concat('systemusers(', outputs('Get_reviewer')?['body/systemuserid'], ')')` |
| Signed On | `utcNow()` |
| Comments | `first(outputs('Approve_reviewer')?['body/responses'])?['comments']` |

For the employee rows use Name `Pre - Employee`, Role Employee, Signer from `Get_employee`, Comments from `Approve_employee`. Leave Return Full Metadata, Appraisal Signature, Import Sequence Number, Owner, Record Created On and the time-zone fields blank.

**Rejected handling (both branches):** Update a row (Row ID `triggerOutputs()?['body/aldot2_appraisalid']`, Stage = Pre - Supervisor Review), Send an email (V2) to the rater with the comments and app link, then Terminate with status Cancelled.

**Employee signature** condition: left box expression `first(outputs('Get_settings')?['body/value'])?['aldot2_prerequiresemployeesignature']`, is equal to, right box `true`. Default is No, per the customer's answer; the diagram shows the employee signing the pre-appraisal, so confirm. **Approve employee** is the same as Approve reviewer, assigned to the employee, title `Sign your pre-appraisal: ` + name.

**Update a row - pre-appraisal complete** (below the Employee signature condition, inside the reviewer's If yes branch):

| Field | Value |
|---|---|
| Table name | Appraisals |
| Row ID | `triggerOutputs()?['body/aldot2_appraisalid']` |
| Stage | Pre - Complete |
| Pre Signed On | `formatDateTime(utcNow(), 'yyyy-MM-dd')` |
| Mid Due Date | `formatDateTime(addToTime(utcNow(), first(outputs('Get_settings')?['body/value'])?['aldot2_midoffsetmonths'], 'Month'), 'yyyy-MM-dd')` |

---

## 4. Appraisal - Annual signatures

Built as a copy of Flow 3 (Save As). Trigger as in step 5, trigger condition:
```
@equals(triggerOutputs()?['body/aldot2_stage'], 867720007)
```
The rating supervisor signed in the app when submitting.

```
When a row is added, modified or deleted
├─ Get employee
├─ Get rater
├─ Get reviewer
├─ Approve reviewer
└─ Reviewer approved
   ├─ If no
   │   ├─ Add a new row - reviewer rejected
   │   ├─ Update a row - reviewer rejected      (Stage = Annual - Supervisor Rating)
   │   ├─ Send an email (V2) - reviewer rejected
   │   └─ Terminate - reviewer rejected
   └─ If yes
       ├─ Add a new row - approved
       ├─ Approve employee
       ├─ Employee approved
       │   ├─ If yes: Add a new row - employee approved
       │   └─ If no:  Add a new row - employee rejected
       │              Update a row - employee rejected   (Stage = Annual - Supervisor Rating)
       │              Send an email (V2) - employee rejected
       │              Terminate - employee rejected
       ├─ Update a row - annual complete        (after the Employee approved condition)
       ├─ List next year row
       └─ No next year row?
           ├─ If yes: Add a new row - next year appraisal
           └─ If no: (empty)
```

**Differences from Flow 3 when copying:**
- Trigger condition uses 867720007.
- Delete Get settings and the Employee signature condition. Drag Approve employee out of its If yes branch to just below the reviewer's approved signature row, drag the Employee approved condition below it, then delete the empty Employee signature condition. The employee approval is always required in the annual.
- Every signature row: Phase = **Annual**, Name `Annual - Reviewing Supervisor` or `Annual - Employee`.
- Both rejected Update a row actions: Stage = **Annual - Supervisor Rating**.
- Titles and email text say annual. Approve reviewer Details: `Performance score: ` + `triggerOutputs()?['body/aldot2_performancescore']` + ` / 10. Review the annual appraisal in the app, then approve or reject.` Rejected emails say scores are recalculated when the rater reopens the appraisal.

**Update a row - annual complete:** Row ID `triggerOutputs()?['body/aldot2_appraisalid']`, Stage = Annual - Complete, Annual Signed On = `formatDateTime(utcNow(), 'yyyy-MM-dd')`. Flow 1 then emails the employee, rater and ASA mailbox.

**List next year row** (Appraisals, Row count 1), Filter rows as an expression (guards against a duplicate created by the ASA or flow 2):
```
concat('_aldot2_employee_value eq ', triggerOutputs()?['body/_aldot2_employee_value'], ' and aldot2_appraisalyearstart eq ', formatDateTime(addDays(triggerOutputs()?['body/aldot2_appraisalyearend'], 1), 'yyyy-MM-dd'))
```
**No next year row?** condition: `length(outputs('List_next_year_row')?['body/value'])` is equal to `0`.

**Add a new row - next year appraisal** (Appraisals):

| Field | Value |
|---|---|
| Name | `concat(outputs('Get_employee')?['body/fullname'], ' ', year(addDays(triggerOutputs()?['body/aldot2_appraisalyearend'], 1)))` |
| Employee | `concat('systemusers(', outputs('Get_employee')?['body/systemuserid'], ')')` |
| Rating Supervisor | `concat('systemusers(', outputs('Get_rater')?['body/systemuserid'], ')')` |
| Reviewing Supervisor | `concat('systemusers(', outputs('Get_reviewer')?['body/systemuserid'], ')')` |
| Cycle Type | Standard |
| Appraisal Year Start | `formatDateTime(addDays(triggerOutputs()?['body/aldot2_appraisalyearend'], 1), 'yyyy-MM-dd')` |
| Appraisal Year End | `formatDateTime(addDays(addToTime(addDays(triggerOutputs()?['body/aldot2_appraisalyearend'], 1), 1, 'Year'), -1), 'yyyy-MM-dd')` |
| Position Title | `triggerOutputs()?['body/aldot2_positiontitle']` |
| Division | `triggerOutputs()?['body/aldot2_division']` |
| Stage | Pre - ASA Setup |

---

## Testing

See `test-plan.md`.

## Known limits of the POC
- Email only; no Teams notifications.
- The annual's two paper signing passes are collapsed into one sequence: rating supervisor (in the app), reviewing supervisor, employee.
- No PDF output; add later if the customer needs a printable copy.
- Security is still screen-level (the home gallery filter). Row-level security (User-scope roles, access teams or sharing, parental relationships on the child tables) is a separate step and will add steps to these flows.
