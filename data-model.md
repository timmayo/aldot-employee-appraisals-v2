# Data model: tables, columns, data types

One business unit. Maker portal data type names are used (Text, Multiline text, Whole number, Decimal number, Yes/No, Date and time, Choice, Lookup). "Date only" means a Date and time column with the date-only format. Display names must match exactly: the Power Fx uses them.

For choices, create a **local choice** on the column. Required is "optional" unless stated.

## 1. Appraisal Setting (plural: Appraisal Settings), one row

| Column | Data type | Notes |
|---|---|---|
| Name (primary) | Text | Value "Default" |
| Min Observation Days | Whole number | 90 |
| Mid Offset Months | Whole number | 6 |
| Annual Offset Months | Whole number | 5 |
| Pre Requires Employee Signature | Yes/No | Default No |

## 2. Rating Scale (plural: Rating Scale), five rows

| Column | Data type | Notes |
|---|---|---|
| Label (rename the primary column) | Text | |
| Rating Value | Whole number | 1 to 5 |
| Responsibility Points | Whole number | 10 = best |
| Disciplinary Points | Whole number | 10 = most action |

| Label | Rating Value | Responsibility Points | Disciplinary Points |
|---|---|---|---|
| 5 - Exceeds Expectations | 5 | 10 | 0 |
| 4 - Frequently Exceeds Expectations | 4 | 8 | 2 |
| 3 - Meets Expectations | 3 | 5 | 5 |
| 2 - Partially Meets Expectations | 2 | 3 | 7 |
| 1 - Needs Improvement | 1 | 0 | 10 |

## 3. Appraisal (plural: Appraisals), one row per employee per appraisal year

| Column | Data type | Notes |
|---|---|---|
| Name (primary) | Text | For example "Jane Smith 2027" |
| Employee | Lookup to User | Required |
| Rating Supervisor | Lookup to User | Required |
| Reviewing Supervisor | Lookup to User | Required |
| Cycle Type | Choice | Standard, Probationary |
| Stage | Choice | Pre - ASA Setup, Pre - Supervisor Review, Pre - Awaiting Signatures, Pre - Complete, Mid - Open, Mid - Complete, Annual - Supervisor Rating, Annual - Awaiting Signatures, Annual - Complete. Default Pre - ASA Setup. Required |
| Appraisal Year Start | Date only | Required |
| Appraisal Year End | Date only | Required |
| Pre Signed On | Date only | Set by Flow 3 |
| Mid Due Date | Date only | Set by Flow 3 |
| Mid Session Date | Date only | Set by the app |
| Annual Due Date | Date only | Set by the app |
| Annual Signed On | Date only | Set by Flow 4 |
| Position Title | Text | Heading |
| Division | Text | Heading |
| Work Habits Discussed | Yes/No | Default No |
| Attendance | Choice | Meets, Needs Improvement |
| Punctuality | Choice | Meets, Needs Improvement |
| Cooperation | Choice | Meets, Needs Improvement |
| Compliance | Choice | Meets, Needs Improvement |
| Mid Feedback Rating | Choice | Exceptional, Competent, Poor |
| Mid Feedback Notes | Multiline text | |
| Needs Corrective Action | Yes/No | Default No |
| Disciplinary Reviewed | Yes/No | Default No |
| Disciplinary Rating | Whole number | Minimum 1, maximum 5 |
| Responsibility Score | Decimal number | 1 decimal place |
| Disciplinary Score | Decimal number | 1 decimal place |
| Performance Score | Decimal number | 1 decimal place |

Set the Yes/No defaults to No. The formulas read these columns directly, and an empty Yes/No can behave unexpectedly.

## 4. Responsibility (plural: Responsibilities)

| Column | Data type | Notes |
|---|---|---|
| Description (rename the primary column) | Text | Required |
| Appraisal | Lookup to Appraisal | Required (many-to-one) |
| Expected Results | Multiline text | |
| Date Added | Date only | Required |
| Date Ended | Date only | Blank while still performed |
| Rating | Whole number | Minimum 1, maximum 5; blank means not yet rated |

## 5. Disciplinary Action (plural: Disciplinary Actions)

| Column | Data type | Notes |
|---|---|---|
| Description (rename the primary column) | Text | |
| Appraisal | Lookup to Appraisal | Required |
| Action Date | Date only | |

## 6. Corrective Action Plan (plural: Corrective Action Plans)

| Column | Data type | Notes |
|---|---|---|
| Area of Concern (rename the primary column) | Text | |
| Appraisal | Lookup to Appraisal | Required |
| Plan | Multiline text | |
| Target Date | Date only | |

## 7. Appraisal Signature (plural: Appraisal Signatures), audit trail

| Column | Data type | Notes |
|---|---|---|
| Name (primary) | Text | For example "Pre - Rating Supervisor" |
| Appraisal | Lookup to Appraisal | Required |
| Phase | Choice | Pre, Annual |
| Role | Choice | Rating Supervisor, Reviewing Supervisor, Employee |
| Outcome | Choice | Approved, Rejected |
| Signer | Lookup to User | |
| Signed On | Date and time | |
| Comments | Multiline text | |

The app's signature Patch calls do not set `Name`. If the maker portal makes the primary column required, add a `Name:` value to those Patch calls.
