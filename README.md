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

1. **Course catalog + performance indices**
   - Store courses in a SharePoint list with fields such as:
     - Course Number
     - Course Title
     - Active/Inactive
     - Associated General Performance Indices (multi-select lookup)
   - Build an **Admin screen in Power Apps** to add/remove/edit courses and indices.

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

1. **Courses**
   - `CourseId` (ID)
   - `CourseNumber` (Text, unique)
   - `CourseTitle` (Text)
   - `IsActive` (Yes/No)

2. **PerformanceIndices**
   - `IndexId` (ID)
   - `IndexCode` (Text)
   - `IndexDescription` (Text)

3. **CoursePerformanceIndices** (junction list)
   - `Course` (Lookup → Courses)
   - `PerformanceIndex` (Lookup → PerformanceIndices)

4. **Semesters**
   - `SemesterId` (ID)
   - `TermName` (e.g., Fall 2026)
   - `StartDate`, `EndDate`
   - `Status` (Draft / Active / Closed)

5. **TeachingAssignments**
   - `AssignmentId` (ID)
   - `Semester` (Lookup)
   - `Course` (Lookup)
   - `InstructorEmail` (Text)
   - `InstructorName` (Text)
   - `FormStatus` (NotSent / Sent / InProgress / Submitted)
   - `FormToken` (GUID)

6. **Questions**
   - `QuestionId` (ID)
   - `QuestionText` (Multiple lines)
   - `QuestionType` (Choice: LongText, SingleChoice)
   - `AppliesTo` (Choice: Global, CourseSpecific)
   - `Course` (Lookup, nullable)
   - `DisplayOrder` (Number)
   - `IsActive` (Yes/No)

7. **QuestionChoices**
   - `ChoiceId` (ID)
   - `Question` (Lookup)
   - `ChoiceLabel` (Text)
   - `ChoiceValue` (Text)
   - `DisplayOrder` (Number)

8. **Responses**
   - `ResponseId` (ID)
   - `Assignment` (Lookup)
   - `Question` (Lookup)
   - `AnswerText` (Multiple lines)
   - `AnswerChoice` (Text)
   - `SubmittedAt` (DateTime)

---

## App Modules

### 1) Admin App (Power Apps)
- Manage courses and performance indices
- Configure questions and order
- Upload semester assignment file
- Monitor completion status dashboard
- Trigger resend reminders manually

### 2) Instructor App (Power Apps)
- Authenticated landing page listing pending forms
- Dynamic question rendering by assignment/course
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
- Build admin CRUD screens for courses/indices
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
