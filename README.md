# Employee Appraisal POC

Canvas app + Dataverse + Power Automate. Proves the pre, mid and annual appraisal workflows from the two customer diagrams.

Files: `data-model.md` (tables and columns), `app-formulas.txt` (Power Fx), `flows.md` (4 flows).

## Assumptions (change them in the Appraisal Settings row or the Rating Scale table, no code change)

| Topic | Assumption | Setting |
|---|---|---|
| Rating scale | Each responsibility is rated 1-5; the rating maps to points in the Rating Scale table (see `data-model.md`) | Rating Scale table |
| Responsibility score | Average of the mapped responsibility points across the rated responsibilities, 0-10, one decimal | Rating Scale table |
| Disciplinary score | No disciplinary actions listed: 0. Otherwise the rating supervisor selects one rating (same 1-5 scale) for the actions listed, and it maps to disciplinary points, 0-10 | Rating Scale table |
| Performance score | **Calculated**, not typed: responsibility score minus disciplinary score, floor of 0, range 0-10. Stays locked until every ratable responsibility is rated and the rater confirms the disciplinary section (and, if actions are listed, selects the disciplinary rating) | n/a |
| 3-month rule | A responsibility is rated only if it was active at least 90 days; shorter ones are shown as "not rated" and excluded from the score and the lock | Min Observation Days = 90 |
| 90-day rule | Responsibilities cannot be added or retired in the mid-appraisal when fewer than 90 days remain in the appraisal year | Min Observation Days = 90 |
| Cycle timing | Mid due 6 months after the pre-appraisal is signed; annual due 5 months after the mid-appraisal session date | Mid Offset Months = 6, Annual Offset Months = 5 |
| Signatures | Electronic, through Approvals. Annual: rating supervisor, then reviewing supervisor, then employee. Pre: rating supervisor, then reviewing supervisor | Pre Requires Employee Signature = No |
| Appraisal year | Set per employee by the ASA (not necessarily the calendar year) | n/a |
| Probationary | Same flow as standard, flagged by Cycle Type | n/a |
| Work habits | Pre: one "discussed" checkbox. Annual: four Meets / Needs Improvement ratings. Neither feeds the performance score (the diagram does not include them) | n/a |
| Out of scope | Reporting, HR system integration, PDF output | n/a |

**Conflict to confirm with the customer:** the diagram shows the employee signing the pre-appraisal, but the answer given was that the employee signs the annual only. The POC follows the answer; flip `Pre Requires Employee Signature` to Yes if the diagram is right.

**To confirm:** the scale mapping is the example you gave. I read it as one table used two ways: each responsibility's rating gives responsibility points, and one rating for the disciplinary actions gives disciplinary points (not a rating per action). If the disciplinary score should instead come from the number of actions, that is a small change in `btnRecalc`.

## Roles

| Role | How the app knows |
|---|---|
| ASA | Member of the Dataverse team **ASA Agents** (Entra group-backed). Not tied to any hierarchy |
| Rating supervisor | Appraisal's Rating Supervisor lookup = signed-in user |
| Reviewing supervisor | Appraisal's Reviewing Supervisor lookup = signed-in user |
| Employee | Appraisal's Employee lookup = signed-in user |

Reviewing supervisor is a lookup on each appraisal, so it can be anyone, not only a manager's manager. The POC filters rows in the app; proper row security is a separate design step.

## Data model

Tables, columns, data types, defaults and the Rating Scale rows are in `data-model.md`.

## Build order

Prerequisites: a Dataverse environment in GCC, maker rights, and the ability to create a Dataverse team backed by an Entra security group.

1. Create a solution and add the tables in `data-model.md` (maker portal, Tables). Add one Appraisal Settings row with the defaults in the assumptions table (90, 6, 5, No) and the five Rating Scale rows.
2. Create an Entra security group for the ASAs, then a Dataverse team of type Microsoft Entra ID security group linked to it, named exactly `ASA Agents`. Give the team a security role that can create and read all six tables.
3. Create the canvas app inside the solution, add the data sources listed at the top of `app-formulas.txt`, build the eight screens, and paste the formulas.
4. Build the four flows in `flows.md` in the same solution.
5. Test with four users: one ASA, one rating supervisor, one reviewing supervisor, one employee.

## Diagram coverage

| Diagram step | Where it lives |
|---|---|
| ASA fills heading, responsibilities and results | scrPreFront, stage Pre - ASA Setup |
| ASA informs rating supervisor to start | Flow 1 email, stage Pre - Supervisor Review |
| Review with employee, work habits checklist, sign and date | scrPreBack, signature row |
| Employee and reviewing supervisor sign, pre-appraisal submitted | Flow 3 |
| Mid-appraisal: monitor, feedback (exceptional / competent / poor) | scrMid, Mid Feedback fields |
| Responsibilities changed, dated additions and retirements | Date Added / Date Ended, btnAddResp, btnRetireResp |
| Disciplined or weak performance leads to corrective action plan | Needs Corrective Action, Corrective Action Plans, required before Mid completes |
| Conduct mid-appraisal session, ideally at six months | Mid Session Date, Mid Due Date |
| Annual: review heading, rate work habits, 1-5 rating per responsibility | scrAnnualFront |
| Responsibility score, disciplinary actions, disciplinary score, subtraction | btnRecalc, scrDisciplinary, scrAnnualBack |
| Performance score disabled until inputs complete | gGateOpen, lblPerfScore, btnSubmitAnnual |
| Forward to reviewing supervisor, all sign, return to ASA | Flow 4 |
| ASA sends annual and new year's pre-appraisal | Flow 4 creates next year's row |
| Timing: mid 6 months after pre, annual 5 months after mid | Flow 2, Mid Due Date, Annual Due Date |
| 90-day and 3-month rules | gCanEditResp, IsRatable |
| Probationary appraisals follow the same steps | Cycle Type |

## Verify before the demo

- Power Apps names marked `[VERIFY]` in `app-formulas.txt` (team membership property, choice enum names such as `'Stage (Appraisals)'`).
- The `addToTime` month arithmetic in Flow 3 near month ends.
- The Power Fx itself has not been run. Test the scoring in the app with a few cases: all responsibilities at 3 gives 5.0; a mix of 5, 4, 3 gives 7.7; with actions listed and a disciplinary rating of 2 (7 points) that same 7.7 gives 0.7; a disciplinary rating of 1 (10 points) against that 7.7 gives 0, not a negative number.
