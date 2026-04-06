# Power Automate Template Flows

## Flow A: Admin-initiated import from Power Apps (Semester Dashboard)
1. **Trigger**: **Power Apps (V2)**.
   - Inputs:
     - `semesterId` (Number)
     - `fileName` (Text)
     - `fileContent` (File)
2. **Action**: Create file in a temp/import library (optional but recommended for traceability).
   - Folder example: `/SemesterImports`
   - File name: include timestamp + semester (`{semesterId}-{utcNow()}.xlsx`)
3. **Action**: Excel Online (Business) — **List rows present in a table**.
   - File: from created file (or directly from trigger file content, if your connector pattern supports it)
   - Table: `AssignmentsImport`
   - Pagination: On, threshold sized to expected import volume (example `5000`).
3. **Action**: Initialize variables (before Apply to each).
   - `varCreatedCount` (Integer) = `0`
   - `varErrorCount` (Integer) = `0`
4. **Action**: **Apply to each** row from `value`.
   1. **Normalize row values** (Compose actions recommended): trim `InstructorName`, `CourseNumber`, `Semester`, `Section`.
   2. **SharePoint — Get items** from `Courses` with OData filter:
      - `CourseNumber eq '<row.CourseNumber>' and IsActive eq 1`
      - Top Count = `1`
   3. **SharePoint — Get items** from `Faculty` (directory list).
      - Pull candidate faculty rows and match normalized full name (`FirstName & " " & LastName`) to row `InstructorName`
      - Expected result count: exactly `1` match
   4. **Compose / Switch** campus code mapping from matched faculty campus:
      - `Pullman -> PUL`
      - `Everett -> EVE`
      - `Bremerton -> BRE`
   5. **Condition**: course found and exactly one faculty match?
      - **No**:
        - Create `ImportErrors` row with file name, row number/key, and error message (missing/ambiguous faculty or missing course).
        - Increment `varErrorCount`.
      - **Yes**:
        - **SharePoint — Create item** in `TeachingAssignments`:
          - `Semester` = row semester
          - `Campus` = matched Faculty.Campus
          - `CampusCode` = mapped code (`PUL`/`EVE`/`BRE`) from Faculty.Campus
          - `InstructorName` = matched Faculty full name
          - `InstructorEmail` = matched Faculty.Email
          - `Course` lookup = returned `Courses` item ID
          - `Section` = row section
          - `FormStatus` = `NotSent`
          - `FormToken` = `guid()`
        - Increment `varCreatedCount`.
5. **Action**: Respond to Power Apps with summary payload (created count, error count, error log link).
6. **Optional**: Outlook — send import summary email to admin group.

---

## Flow B: Send faculty solicitation email
1. **Trigger**: SharePoint — **When an item is created** on `TeachingAssignments`.
   - **Site Address**: same solution site
   - **List Name**: `TeachingAssignments`
   - **Settings (recommended)**: Concurrency Control On, Degree = `1`
2. **Condition**: run only when `FormStatus` is `NotSent`.
   - Expression example: `@equals(triggerBody()?['FormStatus'], 'NotSent')`
3. **Compose** instructor link:
   - `https://apps.powerapps.com/play/<APP_ID>?assignmentId=@{triggerBody()?['ID']}&token=@{triggerBody()?['FormToken']}`
4. **Outlook — Send an email (V2)** to `InstructorEmail`.
   - Subject includes campus code + course + semester.
   - Body includes due date, instructions, and the composed link.
5. **SharePoint — Update item** (`TeachingAssignments`).
   - Set `FormStatus = Sent`
   - Set `SentAt = utcNow()`

---

## Flow C: Reminder emails
1. **Trigger**: **Recurrence**.
   - Frequency: Daily (recommended)
   - Time zone: local campus timezone
   - Suggested run time: early morning local time
2. **SharePoint — Get items** from `Semesters` where reminders are enabled.
   - Filter example: `Status eq 'Active' and RemindersEnabled eq 1`
3. **Apply to each active semester**.
   1. **SharePoint — Get items** from `TeachingAssignments`.
      - Filter Query example:
        - `SemesterId eq <current semester ID> and FormStatus ne 'Submitted' and FormStatus ne 'Closed'`
   2. Use semester cadence:
      - `ReminderCadenceDays` from current semester (fallback to `7` if blank).
   3. **Apply to each** returned assignment.
      - Calculate age since `SentAt` or `LastReminderSentAt`.
      - Condition: if age >= cadence, send reminder.
   4. **Outlook — Send an email (V2)** reminder.
      - Include secure form link, campus, and current status.
   5. **SharePoint — Update item** (`TeachingAssignments`):
      - `LastReminderSentAt = utcNow()`
      - `ReminderCount = add(int(coalesce(ReminderCount, 0)), 1)`
4. **Optional escalation**:
   - If reminder count >= N (example 3), email department chair and/or set `FormStatus = Escalated`.

---

## Flow D: Export submitted responses to Excel table
1. **Trigger**: SharePoint — **When an item is created** on `Responses`.
   - **Site Address**: same solution site
   - **List Name**: `Responses`
2. **SharePoint — Get item / Get items** for related records.
   - Resolve assignment metadata from `TeachingAssignments`
   - Resolve course metadata from `Courses` (if needed)
3. **Excel Online (Business) — Add a row into a table**.
   - **Location**: SharePoint Site
   - **Document Library**: reporting library
   - **File**: reporting workbook path
   - **Table**: `AssessmentResponses`
4. **Map output columns**:
   - Semester
   - Campus
   - CampusCode
   - CourseNumber
   - InstructorEmail
   - QuestionId
   - QuestionText
   - AnswerText
   - AnswerChoice
   - SubmittedAt

---

## Recommended Excel format for `AssignmentsImport`
Create one table in the workbook named exactly **`AssignmentsImport`** with the following columns:

| Column name | Required | Type / format | Example |
|---|---|---|---|
| `Semester` | Yes | Text (`YYYY-Term`) | `2026-Spring` |
| `CourseNumber` | Yes | Text (must match `Courses.CourseNumber`) | `MATH-2413` |
| `Section` | Yes | Text | `001` |
| `InstructorName` | Yes | Text (`FirstName LastName`) matching Faculty directory | `Jane Doe` |

Additional recommendations:
- Keep one assignment per row (no merged cells).
- Keep header names exactly as above.
- Avoid formulas in key fields (`Semester`, `CourseNumber`, `Section`, `InstructorName`); paste values.
- Save as `.xlsx` before uploading to `/SemesterImports`.
