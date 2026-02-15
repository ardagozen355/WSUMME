# Course Assessment Management App (Office 365 Native)

## Recommended Platform
Because your department already has institutional access to **Microsoft 365 / SharePoint / OneDrive / Excel**, the fastest and lowest-maintenance solution is:

- **Power Apps** for user interfaces (admin + instructor)
- **SharePoint Lists** for structured master data and workflow state
- **Power Automate** for semester imports, email solicitations, reminders, and response processing
- **Excel (stored on SharePoint/OneDrive)** for analyst-friendly exports, PivotTables, and PivotCharts
- **Office 365 Outlook connector** for faculty email communication

This avoids custom hosting and gives role-based access control via Azure AD/Microsoft Entra accounts.

---

## Feature Mapping to Your Requirements

1. **Course catalog + student outcomes + performance indicators**
   - Store general Student Outcomes (`SO1`, `SO2`, ...) and Performance Indicators (`PI1.1`, `PI2.3`, ...) where each PI is tied to one SO.
   - Each course stores supported PIs directly (e.g., `ME 123` supports `PI1.1`, `PI2.3`, `PI4.6`; `ME 456` supports `PI2.3`, `PI4.3`).
   - Each course also maintains its own Course-Specific Outcomes (CSOs).
   - Build an **Admin screen in Power Apps** so admins can edit supported PIs and CSOs with clear SO -> PI labeling.

2. **Semester spreadsheet import (course → instructor assignment)**
   - Admin uploads an Excel file (template-controlled) to a SharePoint document library.
   - Power Automate parses the rows and creates semester assignment records.

3. **Email each faculty member a form**
   - After import, Power Automate sends each instructor a personalized email with a secure link to their pending course assessment form.
   - Supports scheduled reminders (e.g., 7-day and 2-day reminders) for incomplete submissions.

4. **Reconfigurable question bank and order**
   - Maintain a SharePoint-backed question model with:
     - Question text
     - Question type (`LongText` or `SingleChoice`)
     - Course-specific vs global flag
     - Display order
     - Active status
   - Admin UI lets users reorder and toggle questions without redeployment.

5. **Question types required**
   - Implement these in Power Apps form rendering logic:
     - Long text entry
     - Configurable single-select multiple choice

6. **Instructor interface to submit forms**
   - Instructor opens their link, authenticates with institutional account, sees assigned course form, and submits.
   - Submission writes normalized response rows to SharePoint, then flows to Excel output.

7. **Spreadsheet-friendly storage and analysis**
   - Responses are exported/appended to an Excel table in SharePoint/OneDrive.
   - PivotTables/PivotCharts can be built directly in Excel against this table.

---

## Data Model (Suggested)

Use SharePoint lists as the primary source of truth:

> Indicator hierarchy: each **PerformanceIndicator** must reference one **StudentOutcome**; each course stores a set of supported PIs; and each course can define additional **Course-Specific Outcomes (CSOs)**.
> For cross-tenant formula stability, prefer deriving SO label from `IndicatorCode` pattern (`PIx.y -> SOx`) or store helper `SOCode` text on PI rows.

1. **Courses**
   - `CourseId` (ID)
   - `CourseNumber` (Text, unique)
   - `CourseTitle` (Text)
   - `IsActive` (Yes/No)
   - `SupportedPIs` (Multi-lookup → PerformanceIndicators)

2. **StudentOutcomes**
   - `OutcomeId` (ID)
   - `OutcomeCode` (Text, unique)
   - `OutcomeDescription` (Text)

3. **PerformanceIndicators**
   - `IndicatorId` (ID)
   - `IndicatorCode` (Text, unique; e.g., `PI2.3`)
   - `IndicatorDescription` (Text)
   - `StudentOutcome` (Lookup → StudentOutcomes)
   - `SOCode` (Text, optional helper for app label stability if lookup parsing is inconsistent)

4. **CourseSpecificOutcomes**
   - `CSOId` (ID)
   - `Course` (Lookup → Courses)
   - `CSOCode` (Text)
   - `CSODescription` (Text)
   - `IsActive` (Yes/No)

5. **Semesters**
   - `SemesterId` (ID)
   - `TermName` (e.g., Fall 2026)
   - `StartDate`, `EndDate`
   - `Status` (Draft / Active / Closed)

6. **TeachingAssignments**
   - `AssignmentId` (ID)
   - `Semester` (Lookup)
   - `Course` (Lookup)
   - `InstructorEmail` (Text)
   - `InstructorName` (Text)
   - `FormStatus` (NotSent / Sent / InProgress / Submitted)
   - `FormToken` (GUID)

7. **Questions**
   - `QuestionId` (ID)
   - `QuestionText` (Multiple lines)
   - `QuestionType` (Choice: LongText, SingleChoice)
   - `AppliesTo` (Choice: Global, CourseSpecific)
   - `Course` (Lookup, nullable)
   - `DisplayOrder` (Number)
   - `IsActive` (Yes/No)

8. **QuestionChoices**
   - `ChoiceId` (ID)
   - `Question` (Lookup)
   - `ChoiceLabel` (Text)
   - `ChoiceValue` (Text)
   - `DisplayOrder` (Number)

9. **Responses**
   - `ResponseId` (ID)
   - `Assignment` (Lookup)
   - `Question` (Lookup)
   - `AnswerText` (Multiple lines)
   - `AnswerChoice` (Text)
   - `SubmittedAt` (DateTime)

10. **OutcomeEvaluations**
   - `EvaluationId` (ID)
   - `Assignment` (Lookup → TeachingAssignments)
   - `EvaluationType` (Choice: `PI`, `CSO`)
   - `ReferenceId` (Number)
   - `ReferenceCode` (Text)
   - `Score` (Number: 1–5)
   - `SubmittedAt` (DateTime)

---

## App Modules

### 1) Admin App (Power Apps)
- Manage courses, student outcomes, supported PIs, and course-specific outcomes
- Configure questions and order
- Upload semester assignment file
- Monitor completion status dashboard
- Trigger resend reminders manually

### 2) Instructor App (Power Apps)
- Authenticated landing page listing pending forms
- Dynamic question rendering by assignment/course
- Rate PI/CSO performance on a 1-5 scale
- Save draft + final submit

### 3) Automations (Power Automate)
- On semester file upload: parse and create assignments
- On assignment creation: send faculty email with deep link
- Reminder scheduler for pending submissions
- On submit: persist normalized responses + append Excel rowset

---

## Security & Governance

- Restrict admin app access to departmental coordinators (AAD group).
- Instructors can only read/write their own assignments/responses.
- Enable versioning and audit trail on key SharePoint lists.
- Use Data Loss Prevention (DLP) policies in Power Platform environment.
- Keep all storage within institutional tenant (SharePoint/OneDrive).

---

## Implementation Roadmap

### Phase 1 (1–2 weeks): Foundation
- Create SharePoint lists and Excel template
- Build admin CRUD screens for courses/outcomes/indicators
- Implement semester import flow

### Phase 2 (1–2 weeks): Assessment workflow
- Build question bank configuration UI
- Build instructor assessment app
- Implement email send + reminder flows

### Phase 3 (1 week): Reporting and hardening
- Build Excel PivotTables/PivotCharts workbook
- Add validation, role checks, and monitoring
- Pilot with one semester and refine

---

## Example Excel Import Template

`SemesterAssignments.xlsx` (table: `AssignmentsImport`)

- `Semester`
- `CourseNumber`
- `CourseTitle`
- `InstructorName`
- `InstructorEmail`

Power Automate validates:
- Course exists and active
- Instructor email format is valid
- No duplicate assignment rows

---

## Alternative (Full-Code) Stack if Needed

If later you need richer customization than Power Apps:
- **Frontend**: React + TypeScript
- **Backend**: .NET 8 Web API
- **Database**: Azure SQL
- **Auth**: Azure AD
- **Files/Reports**: SharePoint + Microsoft Graph API integration

Start with Power Platform first; it is the best fit for your existing Microsoft ecosystem and should deliver faster.

---

## Template Implementation (Concrete Starter)

Yes — even though Power Apps is low-code, you can still use a practical, implementation-ready template.
This repository now includes starter artifacts you can apply directly:

- SharePoint list schema template: `templates/sharepoint-lists-schema.csv`
- Power Apps formula snippets for both instructor and admin apps (dynamic forms, CRUD, ordering, dashboard actions): `templates/power-apps-formulas.md`
- Power Automate flow blueprints for import, solicitation, reminders, and Excel export: `templates/power-automate-flow-outline.md`
- Admin + Instructor screen wireframes and control naming map aligned to formulas: `templates/power-apps-ui-schematic.md`

### Quick start (first 2 hours)
1. Create SharePoint lists using `templates/sharepoint-lists-schema.csv` as your field checklist.
2. Create a Canvas app with two screens (`Admin`, `Instructor`) and paste/adapt formulas from `templates/power-apps-formulas.md`.
3. Build Flow A and Flow B from `templates/power-automate-flow-outline.md`.
4. Upload a test import file and run an end-to-end dry run with two sample instructors.

### What this gives you immediately
- A working baseline for:
  - Course + index management
  - Semester assignment ingest
  - Instructor-specific dynamic questionnaires
  - Submission persistence and export-ready response rows

If you want, the next step can be a **tenant-ready deployment checklist** (environment variables, naming conventions, security roles, and go-live validation script).

---

## FAQ: Permissions and SharePoint Lists vs Excel

### 1) Do admins and instructors both need edit permissions to all data lists?
No.

Recommended minimum permissions:

- **Admins**: Edit/Contribute on configuration + operational lists they manage
  - `Courses`, `StudentOutcomes`, `PerformanceIndicators`, `CourseSpecificOutcomes`, `Questions`, `QuestionChoices`, `Semesters`, `TeachingAssignments`, `OutcomeEvaluations`
- **Instructors**: Limited permissions
  - Read assigned `TeachingAssignments`
  - Create/Edit their own `Responses` rows only
  - No edit access to configuration lists (`Courses`, `Questions`, etc.)
- **Power Automate service account**: Contribute on lists/files touched by flows (imports, assignment updates, response export)

Implementation note:
- If you keep one SharePoint site for all data, use list-level permissions and/or item-level settings (especially for `Responses`).
- Prefer separate security groups (e.g., `DeptAssessmentAdmins`, `DeptAssessmentInstructors`) and assign app sharing + list rights to groups rather than individuals.

### 2) Can shared Excel sheets be used instead of SharePoint Lists for data?
Possible, but **not recommended** for core transactional data.

Use this pattern instead:
- **SharePoint Lists = system of record** (apps + workflows)
- **Excel = reporting/export layer** (PivotTables, charts, ad-hoc analysis)

Why Lists are better for app data:
- Better row-level CRUD behavior for multi-user apps
- More reliable for concurrent edits than a single workbook lock model
- Cleaner security model (list/item permissions)
- Better trigger semantics for Power Automate
- Better for relational structures (`Questions` ↔ `QuestionChoices`, assignments ↔ responses)

When Excel is okay:
- Semester import file (staging input)
- Final flattened response table for analysts
- Dashboard workbook for pivots/charts

If you must use Excel as primary storage, expect extra effort for:
- Concurrency conflict handling
- Validation and referential integrity checks
- More brittle flow/app behavior under high parallel usage
