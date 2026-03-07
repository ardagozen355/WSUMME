# Power Automate Template Flows

## Flow A: Import semester assignment spreadsheet
1. **Trigger**: SharePoint — **When a file is created (properties only)**.
   - **Site Address**: the same SharePoint site that hosts your app lists and import library (example: `https://contoso.sharepoint.com/sites/Assessment`).
   - **Library Name**: the document library used for imports (example: `Documents` or `Shared Documents`).
   - **Folder**: `/SemesterImports` (or your configured import folder under the selected library).
   - **Trigger Conditions (recommended)**:
     - Only run for Excel files: `@endsWith(toLower(triggerOutputs()?['body/{FilenameWithExtension}']), '.xlsx')`
     - Ignore temporary Office lock files: `@not(startsWith(triggerOutputs()?['body/{FilenameWithExtension}'], '~$'))`
   - **Concurrency Control (recommended)**: On, Degree of Parallelism = `1` to prevent duplicate processing when multiple files arrive close together.
2. **Action**: List rows present in table (`AssignmentsImport`) from Excel Online (Business).
   - **Location**: SharePoint Site
   - **Document Library**: same library as trigger
   - **File**: use trigger identifier/path from Flow A trigger output
3. **Apply to each row**:
   - Get matching course by `CourseNumber` from `Courses`.
   - If missing or inactive -> append row to `ImportErrors` list.
   - Else create `TeachingAssignments` item with `FormStatus=NotSent`, `FormToken=guid()`.
4. **Action**: Send summary email to admin with created count/error count.

## Flow B: Send faculty solicitation email
1. **Trigger**: When `TeachingAssignments` item is created.
2. **Condition**: `FormStatus == NotSent`.
3. **Compose** instructor link:
   - `https://apps.powerapps.com/play/<APP_ID>?assignmentId=@{ID}&token=@{FormToken}`
4. **Send email** via Outlook connector.
5. **Update item**: set `FormStatus=Sent` and store `SentAt` timestamp.

## Flow C: Reminder emails
1. **Trigger**: Recurrence daily.
2. **Get items** from `TeachingAssignments` where `FormStatus != Submitted` and older than X days.
3. **Send reminder email**.
4. Optional escalation to department chair after N reminders.

## Flow D: Export submitted responses to Excel table
1. **Trigger**: When item is created in `Responses`.
2. **Action**: Get related assignment and course metadata.
3. **Action**: Add row into Excel table `AssessmentResponses` in workbook stored on SharePoint.
4. **Columns**:
   - Semester
   - CourseNumber
   - InstructorEmail
   - QuestionId
   - QuestionText
   - AnswerText
   - AnswerChoice
   - SubmittedAt
