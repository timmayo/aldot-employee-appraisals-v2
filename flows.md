# Power Automate flows

Four cloud flows in the same solution as the app. Run them under a dedicated service account (not a person's account) so reminders and stage changes keep working when people leave. Connectors: Microsoft Dataverse, Approvals, Office 365 Outlook, Office 365 Users. Confirm all four are allowed together by the GCC DLP policy for the environment.

Trigger type for flows 1, 3 and 4: Dataverse **When a row is added, modified or deleted**, change type **Modified**, table **Appraisals**, scope **Organization**, **Select columns = Stage** so the flow fires only on stage changes.

## 1. Appraisal - Stage notifications
Trigger: Stage modified. Switch on the new Stage:

| New stage | Email to | Message |
|---|---|---|
| Pre - Supervisor Review | Rating supervisor | ASA has prepared the pre-appraisal; review it with the employee (diagram: "ASA informs rating supervisor to start pre-appraisal") |
| Pre - Awaiting Signatures | Reviewing supervisor (and employee if the setting is on) | Signature request coming from Approvals |
| Pre - Complete | ASA team mailbox | Pre-appraisal submitted |
| Mid - Open | Rating supervisor | Mid-appraisal is open |
| Annual - Supervisor Rating | Rating supervisor | Annual appraisal is open for rating |
| Annual - Complete | Employee, rating supervisor, ASA team mailbox | Annual appraisal signed; next year's pre-appraisal created |

## 2. Appraisal - Daily scheduler
Trigger: Recurrence, daily.

1. **Advance to Mid:** list Appraisals where Stage = Pre - Complete and Mid Due Date <= today. Set Stage = Mid - Open.
2. **Advance to Annual:** list Appraisals where Stage = Mid - Complete and Annual Due Date <= today. Set Stage = Annual - Supervisor Rating.
3. **Reminders:** for Mid - Open or Annual - Supervisor Rating, email the rating supervisor when the due date is 14 days away, 7 days away, and every 7 days once overdue.
4. **New year (ASA):** 7 days before an Appraisal Year End, if the employee has no Appraisals row for the next year, email the ASA team mailbox (covers the "first week of the new appraisal year" rule for permanent, laterally transferred and relocated employees).

## 3. Appraisal - Pre signatures
Trigger: Stage modified; condition Stage = Pre - Awaiting Signatures. The rating supervisor already signed in the app (a row in Appraisal Signatures).

1. **Start and wait for an approval** (type Approve/Reject - First to respond), assigned to the reviewing supervisor's email. Title: "Sign pre-appraisal: [employee]". Details link to the app.
2. Add an Appraisal Signatures row: Phase = Pre, Role = Reviewing Supervisor, Signer, Signed On = utcNow(), Outcome, Comments.
3. If Appraisal Settings has **Pre Requires Employee Signature** = Yes, repeat steps 1-2 for the employee (default No, per the customer's answer; the diagram shows the employee signing the pre-appraisal, so confirm).
4. **If any outcome is Reject:** set Stage = Pre - Supervisor Review, email the rating supervisor with the comments, stop.
5. **If all approved:** update the appraisal: Stage = Pre - Complete, Pre Signed On = utcNow(), Mid Due Date = addToTime(utcNow(), Mid Offset Months, 'Month').

## 4. Appraisal - Annual signatures
Trigger: Stage modified; condition Stage = Annual - Awaiting Signatures. The rating supervisor signed in the app when submitting.

1. Approval to the reviewing supervisor, then an approval to the employee (sequential). Each result adds an Appraisal Signatures row with Phase = Annual.
2. **If any outcome is Reject:** set Stage = Annual - Supervisor Rating, email the rating supervisor with the comments, stop. (Scores are recalculated when the rater reopens the appraisal.)
3. **If all approved:** set Stage = Annual - Complete and Annual Signed On = utcNow().
4. **Create next year's row** (replaces "ASA emails out the new year's pre-appraisal"): add an Appraisals row with the same Employee, Rating Supervisor, Reviewing Supervisor, Cycle Type = Standard, Appraisal Year Start = this Year End + 1 day, Appraisal Year End = start + 1 year - 1 day, Stage = Pre - ASA Setup. The ASA fills in the heading and responsibilities.

## Known limits of the POC
- Email only; no Teams notifications.
- The annual's two paper signing passes are collapsed into one sequence: rating supervisor, reviewing supervisor, employee.
- No PDF output; add later if the customer needs a printable copy.
