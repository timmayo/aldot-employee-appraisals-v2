# Basic test plan

Confirms the app and the four flows work together from creating an appraisal to the next year's row.

## Setup

- Four test accounts if possible: ASA, rating supervisor, reviewing supervisor, employee.
- Each has a Primary Email on the Users table, the Appraisal User role (the ASA also has Appraisal ASA, through the AL-DOT ASA Agents team), and the app shared with them.
- All four flows are turned on. Keep the flow run history open in a second tab.
- Flow runs are in make.powerautomate.com, My flows or the solution, then the flow, then the run list.

## A. Create and pre-appraisal

| # | Do | Expect |
|---|---|---|
| 1 | As ASA, create a new appraisal (employee, rater, reviewer, dates, position, division) | The row appears on the home screen at stage Pre - ASA Setup. |
| 2 | Open it, add 2 responsibilities with expected results, choose Send to rating supervisor | Stage is Pre - Supervisor Review. Flow 1 ran and the rater got the email. |
| 3 | As rater, open it, tick the discussed box, sign | Stage is Pre - Awaiting Signatures. Flow 3 is waiting on an approval. A signature row exists for the rater. |
| 4 | As reviewer, approve it in Approvals | Stage is Pre - Complete. Pre Signed On and Mid Due Date are set. A reviewer signature row exists. Flow 1 emailed the ASA mailbox. |
| 5 | Repeat steps 1 to 3 on a second appraisal and reject as reviewer | Stage returns to Pre - Supervisor Review. The rater got an email with the comments. A Rejected signature row exists. |

## B. Mid-appraisal

| # | Do | Expect |
|---|---|---|
| 6 | In the table, set the Pre - Complete row's Mid Due Date to today, then run Flow 2 manually | Stage is Mid - Open. Flow 1 emailed the rater. |
| 7 | As rater, open it, add a feedback rating, notes and a session date, then complete | Stage is Mid - Complete. Annual Due Date is the session date plus 5 months. |

## C. Annual appraisal

| # | Do | Expect |
|---|---|---|
| 8 | Set the Annual Due Date to today and run Flow 2 | Stage is Annual - Supervisor Rating. The rater got the email. |
| 9 | As rater, rate every responsibility, set the work habits, confirm the disciplinary section, and check the Performance score | The score unlocks only after all inputs are done. Submit is enabled. |
| 10 | Submit | Stage is Annual - Awaiting Signatures. Flow 4 is waiting on the reviewer's approval. |
| 11 | As reviewer, approve. Then as employee, approve | Stage is Annual - Complete. Annual Signed On is set. Three Annual signature rows exist. Flow 1 emailed employee, rater and ASA mailbox. |
| 12 | Check the Appraisals table | A new row exists for next year, stage Pre - ASA Setup, with the right start and end dates. |
| 13 | On another appraisal, reject as employee | Stage returns to Annual - Supervisor Rating and the rater got the comments. |

## D. Reminders (run Flow 2 manually each time)

| # | Do | Expect |
|---|---|---|
| 14 | Set a Mid - Open row's Mid Due Date to 7 days from today | The rater gets a reminder email. |
| 15 | Set it to 10 days from today | No email. |
| 16 | Set an Annual - Supervisor Rating row's Annual Due Date to 7 days ago | The rater gets a reminder. Set it to 3 days ago and nothing is sent. |
| 17 | Set an appraisal's Appraisal Year End to 7 days from today, with no row for the next year | The ASA mailbox gets a "create next year's appraisal" email. |

## E. Rules

| # | Do | Expect |
|---|---|---|
| 18 | As employee, open another person's appraisal | It is not in your gallery. This is screen-level only for now; row-level security is a later step. |
| 19 | In mid-appraisal with under 90 days left in the year, try to add a responsibility | The window warning shows and Add is disabled. |
| 20 | Add a responsibility that is under 90 days old before rating | It shows "Not rated" and does not block the Performance score. |

## If a flow fails

Open its run history, find the red action, and read the error. Most are:
- a wrong logical name (see `flows.md`, Before you build),
- a misspelled action name in an expression (spaces become underscores, hyphens stay),
- a missing security role or mailbox on the service account.

## Result

| Section | Pass | Notes |
|---|---|---|
| A. Create and pre-appraisal | | |
| B. Mid-appraisal | | |
| C. Annual appraisal | | |
| D. Reminders | | |
| E. Rules | | |
