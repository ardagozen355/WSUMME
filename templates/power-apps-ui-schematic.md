# Power Apps UI Schematic (Admin + Instructor)

This schematic aligns directly with control/variable names used in `templates/power-apps-formulas.md`.

Implementation note: if you build a **single combined app**, use the merged `App.OnStart` from `templates/power-apps-formulas.md` and route to `scrAdminHome` or `scrMyAssignments` by `varIsAdmin`. If you build **two separate apps**, each app uses only its own `OnStart` section.

Startup behavior note:
- Set **App.StartScreen** for routing (recommended):
```powerfx
If(varIsAdmin, scrAdminHome, scrMyAssignments)
```
- In Studio preview, use **App -> Run OnStart** to refresh startup variables.


---

## 1) Instructor App UI (Canvas)

## Screen IA-1: `scrMyAssignments`
Purpose: Instructor sees pending forms.

### Layout (wireframe)
```text
+--------------------------------------------------------------------------------+
| Header: "Course Assessment Forms"                                             |
| User: <User().FullName>                                                        |
+--------------------------------------------------------------------------------+
| Search (optional) [txtAssignmentSearch]                                        |
|--------------------------------------------------------------------------------|
| Gallery [galAssignments] (blank vertical; embedded controls)                   |
|  - [lblAssignCourseTitle]                                                      |
|  - [lblAssignSemester] [lblAssignStatus] [btnOpenFormRow]                      |
+--------------------------------------------------------------------------------+
```

### Controls
- `galAssignments.Items` -> `colMyAssignments`
- `lblAssignCourseTitle.Text` -> `Coalesce(ThisItem.Course.Value, "") & " - " & Coalesce(LookUp(Courses, ID = ThisItem.Course.Id, CourseTitle), "")`
- `lblAssignSemester.Text` -> `Coalesce(ThisItem.Semester.Value, "")`
- `lblAssignStatus.Text` -> `Coalesce(ThisItem.FormStatus.Value, "")`
- `btnOpenFormRow.OnSelect`:
  - set `varAssignmentId` and `varCourseId`
  - build `colQuestions` and `colResponses`
  - `Navigate(scrAssessmentForm, ScreenTransition.Fade)`

(Uses formulas from section **Instructor 1 & 2**.)

---

## Screen IA-2: `scrAssessmentForm`
Purpose: Instructor answers global questions and completes PI/CSO evaluations with PI-specific option ratings (configured by admins) plus assessment-tools rationale.

### Layout (wireframe)
```text
+--------------------------------------------------------------------------------+
| Back | Course: <selected course> | Semester: <selected semester>              |
+--------------------------------------------------------------------------------+
| Scrollable gallery [galQuestions] (blank vertical; embedded controls)          |
|  - [lblQuestionText]                                                           |
|  - [txtLongAnswer]  (Visible when LongText)                                    |
|  - [drpSingleChoice] (Visible when SingleChoice)                               |
|--------------------------------------------------------------------------------|
|  ...                                                                           |
+--------------------------------------------------------------------------------+
| Ratings [galEvalItems] (blank vertical; embedded controls):                    |
|  - [lblEvalCode] [drpScore (PI options / CSO scale)] [txtAssessmentTools]      |
| [btnSubmitAssessment]                                                          |
+--------------------------------------------------------------------------------+
```

### Controls
- `lblAssessmentCourse.Text` -> `"Course: " & Coalesce(LookUp(Courses, ID = varCourseId, CourseNumber & " - " & CourseTitle), "(No course selected)")`
- `lblAssessmentSemester.Text` -> `"Semester: " & Coalesce(LookUp(TeachingAssignments, ID = varAssignmentId, Semester.Value), "(No semester selected)")`
- `btnBackToAssignments.OnSelect` -> `Navigate(scrMyAssignments, ScreenTransition.Fade)` (optional)
- `galQuestions.Items` -> `colResponses`
- `lblQuestionText.Text` -> `ThisItem.QuestionText`
- `txtLongAnswer.Visible` -> `ThisItem.QuestionType.Value = "LongText"`
- `drpSingleChoice.Visible` -> `ThisItem.QuestionType.Value = "SingleChoice"`
- `drpSingleChoice.Items` -> `Filter(QuestionChoices, Question.Id = ThisItem.ID)` sorted by `DisplayOrder`
- `txtLongAnswer.OnChange` and `drpSingleChoice.OnChange` patch `colResponses`
- `galEvalItems.Items` -> `colEvalItems` (PI + CSO items)
- `drpScore.OnChange` -> updates `ScoreLocal`
- `txtAssessmentTools.OnChange` -> updates `AssessmentToolsLocal`
- `drpScore.Items` -> formula **7** (`ThisItem.OptionItems`, PI-specific when `EvalType="PI"`)
- `btnSubmitAssessment.OnSelect` uses submit formula + saves `OutcomeEvaluations`.

(Uses formulas from section **Instructor 3–7**.)

---

## 2) Admin App UI (Canvas)

## Screen AD-1: `scrAdminHome`
Purpose: Navigation hub and admin guard.

### Layout (wireframe)
```text
+--------------------------------------------------------------------------------+
| Header: "Assessment Admin"                                                    |
| Admin: <varDisplayName>                                                         |
+--------------------------------------------------------------------------------+
| [btnCourses] [btnFaculty] [btnOutcomesPIs] [btnQuestions] [btnSemesterDashboard] [btnImports] |
+--------------------------------------------------------------------------------+
| Info card: role status (varIsAdmin)                                            |
+--------------------------------------------------------------------------------+
```

### Controls (exact types + variable bindings)
- **Header container**: Insert > **Horizontal container** (`conHeaderAdmin`)
  - Child label `lblHeaderTitle.Text`:
  ```powerfx
  "Assessment Admin"
  ```
  - Child label `lblAdminName.Text`:
  ```powerfx
  "Admin: " & varDisplayName
  ```
- **Navigation controls**: Insert > **Button**
  - `btnCourses`, `btnFaculty`, `btnOutcomesPIs`, `btnQuestions`, `btnSemesterDashboard`, `btnImports`
- **Info card**: easiest approach is Insert > **Container** (`conRoleCard`) with two labels inside:
  - `lblRoleTitle.Text`:
  ```powerfx
  "Role Status"
  ```
  - `lblRoleValue.Text`:
  ```powerfx
  If(varIsAdmin, "Admin access granted", "No admin access")
  ```
  - Optional color cue (`lblRoleValue.Color`):
  ```powerfx
  If(varIsAdmin, Color.DarkGreen, Color.DarkRed)
  ```

### How `AdminUsers` drives access on this screen
- Add `AdminUsers` as a data source to the app (SharePoint connector).
- In `App.OnStart`, the app evaluates whether signed-in email exists in that list and sets `varIsAdmin`.
- UI behavior depends on `varIsAdmin`:
  - Admin buttons enabled when `true`
  - Admin buttons disabled when `false`

### Calling variables on this screen
- `App.OnStart` (or combined OnStart) initializes:
  - `varUserEmail`
  - `varDisplayName`
  - `varIsAdmin`
- Any label/button can reference these directly, e.g.:
  - `lblWhoAmI.Text`:
  ```powerfx
  "Signed in as: " & varUserEmail
  ```

### Navigation formulas for Admin Home buttons (`OnSelect`)
```powerfx
// btnCourses
Navigate(scrCourses, ScreenTransition.Fade)
```

```powerfx
// btnQuestions
Navigate(scrQuestions, ScreenTransition.Fade)
```

```powerfx
// btnFaculty
Navigate(scrFaculty, ScreenTransition.Fade)
```

```powerfx
// btnSemesterDashboard
Navigate(scrSemesterDashboard, ScreenTransition.Fade)
```

```powerfx
// btnImports (if you create this screen)
Navigate(scrImports, ScreenTransition.Fade)
```

### Guarding buttons for non-admin users
Set each admin button `DisplayMode` to:
```powerfx
If(varIsAdmin, DisplayMode.Edit, DisplayMode.Disabled)
```

(Uses formula **A1**.)

---

## Screen AD-2: `scrCourses`
Purpose: Course CRUD + supported PI mapping + course-specific outcomes (CSOs).

### Layout (wireframe)
```text
+--------------------------------------------------------------------------------+
| Courses                                                                         |
+------------------------------+-------------------------------------------------+
| Left pane                    | Right pane                                      |
| [tglShowActiveOnly]          | Course editor                                   |
| [galCourses]                 | Course Number [txtCourseNumber]                 |
|  - CourseNumber              | Course Title  [txtCourseTitle]                  |
|  - CourseTitle               | Active       [tglCourseActive]                  |
|                              | [btnNewCourse] [btnSaveCourse] [btnDeleteCourse] |
|                              |--------------------------------------------------|
|                              | Supported PIs [galSupportedPIs]                   |
|                              |  - [lblSupportedPI] [btnRemovePI]                 |
|                              |--------------------------------------------------|
|                              | Available PIs [galAvailablePIs]                   |
|                              |  - [lblAvailablePI] [btnAddPI]                    |
|                              |--------------------------------------------------|
|                              | Course-specific outcomes [galCSOs]                |
|                              |  - [lblCSOCode] [lblCSODescription] [btnRemoveCSO]|
|                              | [txtCSOCode] [txtCSODescription] [btnNewCSO] [btnAddCSO] |
+------------------------------+-------------------------------------------------+
```

### Controls & bindings
When a course is selected in `galCourses`, the right panel immediately shows that course's number, title, active status, the supported PIs list, the available (not-yet-supported) PIs list, and existing CSOs.

- `galCourses.Items` -> formula **A2**
- `galCourses.OnSelect` -> formula **A2b** (sets `varSelectedCourse`, preloads fields, caches `colAllPIs`, rebuilds `colSupportedPIs`/`colAvailablePIs`, and loads `colCSOs`)
- `txtCourseNumber.Default` -> formula **A2b**
- `txtCourseTitle.Default` -> formula **A2b**
- `tglCourseActive.Default` -> formula **A2b**
- `galSupportedPIs` control type: Vertical gallery (blank)
- `galSupportedPIs` data source selection in designer: **None/blank** (do not pre-bind to `PerformanceIndicators`)
- `galSupportedPIs.Items` -> formula **A5** (shows `colSupportedPIs` and returns empty when no valid course is selected)
- `lblSupportedPI.Text` -> formula **A5** (shows `IndicatorCode` + `IndicatorDescription` on separate lines)
- `galAvailablePIs` control type: Vertical gallery (blank)
- `galAvailablePIs.Items` -> formula **A5** (`colAvailablePIs`, all other PIs not yet supported)
- `lblAvailablePI.Text` -> formula **A5** (shows `IndicatorCode` + `IndicatorDescription` on separate lines)
- `btnAddPI.OnSelect` -> formula **A5** (add PI to selected course)
- `btnRemovePI.OnSelect` -> formula **A5** (remove PI from selected course)
- `galCSOs.Items` -> formula **A5b** (`colCSOs`, all active CSOs for selected course)
- `galCSOs.OnSelect` -> formula **A5b** (load selected CSO into text inputs for editing)
- `txtCSOCode.Default` -> formula **A5b**
- `txtCSODescription.Default` -> formula **A5b**
- `btnNewCSO.OnSelect` -> formula **A5b** (clear selected CSO + clear text inputs for new entry)
- `btnNewCourse.OnSelect` -> formula **A3a** (clear selected course + clear right-pane inputs for new entry)
- `btnSaveCourse.OnSelect` -> formula **A3**
- `btnDeleteCourse.OnSelect` -> formula **A3b** (permanently delete selected course + related CSOs)
- `btnAddCSO.OnSelect` -> formula **A5b** (save selected CSO edits or add new CSO, then refresh `colCSOs`)
- `btnRemoveCSO.OnSelect` -> formula **A5b** (hard delete selected CSO row and refresh `colCSOs`)

---

## Screen AD-2b: `scrFaculty`
Purpose: Manage the Faculty directory used by assignment imports.

### Layout (wireframe)
```text
+--------------------------------------------------------------------------------+
| Faculty Directory                                                               |
+------------------------------+-------------------------------------------------+
| Left pane                    | Right pane                                      |
| [galFaculty]                 | First Name [txtFacultyFirstName]                |
|  - [lblFacultyName]          | Last Name  [txtFacultyLastName]                 |
|  - [lblFacultyCampus]        | Email      [txtFacultyEmail]                    |
|                              | Campus     [drpFacultyCampus]                   |
|                              | [btnNewFaculty] [btnSaveFaculty] [btnDeleteFaculty] |
+------------------------------+-------------------------------------------------+
```

### Controls & bindings
- `galFaculty.Items` -> formula **A5d**
- `lblFacultyName.Text` -> `ThisItem.LastName & ", " & ThisItem.FirstName`
- `lblFacultyCampus.Text` -> `Coalesce(ThisItem.Campus.Value, "")`
- `galFaculty.OnSelect` -> formula **A5d**
- `txtFacultyFirstName.Default` -> formula **A5d**
- `txtFacultyLastName.Default` -> formula **A5d**
- `txtFacultyEmail.Default` -> formula **A5d**
- `drpFacultyCampus.Items` -> formula **A5d**
- `drpFacultyCampus.Default` -> formula **A5d**
- `btnNewFaculty.OnSelect` -> formula **A5d**
- `btnSaveFaculty.OnSelect` -> formula **A5d**
- `btnDeleteFaculty.OnSelect` -> formula **A5d**


## Screen AD-3: `scrQuestions`
Purpose: Question bank management and ordering.

### Layout (wireframe)
```text
+--------------------------------------------------------------------------------+
| Questions                                                                       |
+------------------------------+-------------------------------------------------+
| Left pane                    | Right pane                                      |
| [galQuestionsAdmin]          | Question editor                                 |
|  - [lblOrderAndText]         | Text [txtQuestionText]                          |
|  - [lblType]                 | Type [drpQuestionType]                          |
|  - [lblRequired]             | Required [tglQuestionRequired]                  |
|  - [btnMoveUp]               | Order [txtDisplayOrder]                         |
|                              | [btnNewQuestion] [btnSaveQuestion] [btnDeleteQuestion] |
|                              |--------------------------------------------------|
|                              | Single-choice options                            |
|                              | [btnNewChoice]                                   |
|                              | [galChoices]                                     |
|                              | [txtChoiceOrderRow] [txtChoiceLabelRow]         |
|                              | [btnSaveChoiceRow] [btnDeleteChoiceRow]         |
+------------------------------+-------------------------------------------------+
```

### Controls & bindings
When a course is selected in `galCourses`, the right panel immediately shows that course's number, title, active status, the supported PIs list, the available (not-yet-supported) PIs list, and existing CSOs.

- `galQuestionsAdmin.Items` -> formula **A6**
- `lblOrderAndText.Text` -> `Text(Coalesce(ThisItem.DisplayOrder, ThisItem.ID)) & " - " & Left(ThisItem.QuestionText, 120)`
- `lblType.Text` -> `ThisItem.QuestionType.Value`
- `lblRequired.Text` -> `If(Coalesce(ThisItem.IsRequired, true), "Required", "Optional")`
- `lblRequired.Color` -> `If(Coalesce(ThisItem.IsRequired, true), Color.Red, Color.Gray)` (optional visual cue)
- `galQuestionsAdmin.OnSelect`:
```powerfx
Set(varSelectedQuestion, ThisItem);
Set(varQuestionTextLocal, Coalesce(ThisItem.QuestionText, ""));
Set(varQuestionTypeLocal, Coalesce(ThisItem.QuestionType.Value, "LongText"));
Set(varQuestionRequiredLocal, Coalesce(ThisItem.IsRequired, true));
Set(varQuestionOrderLocal, Text(Coalesce(ThisItem.DisplayOrder, ThisItem.ID)));
Reset(txtQuestionText);
Reset(drpQuestionType);
Reset(tglQuestionRequired);
Reset(txtDisplayOrder)
```
- `txtQuestionText.Default` -> `varQuestionTextLocal`
- `drpQuestionType.Items` -> `Choices(Questions.QuestionType)`
- `drpQuestionType.Default` -> `Coalesce(LookUp(Choices(Questions.QuestionType), Value = varQuestionTypeLocal).Value, "LongText")`
- `tglQuestionRequired.Default` -> `varQuestionRequiredLocal`
- `txtDisplayOrder.Default` -> `varQuestionOrderLocal`
- `btnNewQuestion.OnSelect` -> formula **A7** (clears editor for a new question)
- `btnSaveQuestion.OnSelect` -> formula **A7** (saves text/type/required/order)
- `btnDeleteQuestion.OnSelect` -> formula **A7a** (permanently deletes selected question)
- `btnNewChoice.OnSelect` -> formula **A8** (adds a blank choice row for selected question)
- `galChoices.Visible` -> `Coalesce(varQuestionTypeLocal, "LongText") = "SingleChoice"`
- `galChoices.Items` -> formula **A8** (items)
- `txtChoiceOrderRow.Default` -> `Text(ThisItem.DisplayOrder)`
- `txtChoiceLabelRow.Default` -> `ThisItem.ChoiceLabel`
- `btnSaveChoiceRow.OnSelect` -> formula **A8** (update row)
- `btnDeleteChoiceRow.OnSelect` -> formula **A8** (delete row)
- `btnMoveUp.OnSelect` -> formula **A9**

---


## Screen AD-3b: `scrOutcomesAndPIs`
Purpose: Manage Student Outcomes and Performance Indicators with cascading delete.

### Layout (wireframe)
```text
+--------------------------------------------------------------------------------+
| Outcomes & Performance Indicators                                              |
+------------------------------+-------------------------------------------------+
| Left pane                    | Right pane                                      |
| Student Outcomes             | Performance Indicators (for selected outcome)    |
| [galStudentOutcomesAdmin]    | [galPIsByOutcome]                                |
|  - [lblOutcomeAdmin] [btnMoveUpOutcome] [btnDeleteOutcomeRow] | - [lblPIAdmin] [btnMoveUpPI] [btnDeletePIFromOutcome] |
| [txtOutcomeCode]             | [txtNewPIIndicatorCode]                          |
| [txtOutcomeDescription]      | [txtNewPIIndicatorDescription]                   |
| [btnNewOutcome]              | [btnNewPIForOutcome]                             |
|                              | PI grading options [galPIGradeOptions]           |
|                              | [txtPIOptionOrderRow] [txtPIOptionLabelRow]      |
|                              | [btnSavePIOptionRow] [btnDeletePIOptionRow] [btnNewPIOption] |
+------------------------------+-------------------------------------------------+
```

### Controls & bindings
- `galStudentOutcomesAdmin` control type: Vertical gallery (blank)
- `galStudentOutcomesAdmin.Items` -> formula **A5c**
- `lblOutcomeAdmin.Text` -> formula **A5c**
- `btnMoveUpOutcome.OnSelect` -> formula **A5c** (swap outcome display order upward)
- `btnDeleteOutcomeRow.OnSelect` -> formula **A5c** (row-level cascade delete: selected outcome + related PIs)
- `galStudentOutcomesAdmin.OnSelect` -> formula **A5c** (loads selected outcome into `txtOutcomeCode`/`txtOutcomeDescription`)
- `txtOutcomeCode.Default` -> formula **A5c**
- `txtOutcomeDescription.Default` -> formula **A5c**
- `txtOutcomeCode` + `txtOutcomeDescription` + `btnNewOutcome.OnSelect` -> formula **A5c** (save selected outcome or add SO when none selected)

- `galPIsByOutcome` control type: Vertical gallery (blank)
- `galPIsByOutcome.Items` -> formula **A5c** (typed filter on `PerformanceIndicators`; avoids empty-table schema loss)
- `lblPIAdmin.Text` -> formula **A5c**
- If `ThisItem` shows only `IsSelected`, ensure `lblPIAdmin` is inside `galPIsByOutcome` row template and reselect an outcome.
- `btnMoveUpPI.OnSelect` -> formula **A5c** (swap PI display order upward within selected outcome)
- `btnDeletePIFromOutcome.OnSelect` -> formula **A5c** (row-level PI delete)
- `galPIsByOutcome.OnSelect` -> formula **A5c** (loads selected PI into `txtNewPIIndicatorCode`/`txtNewPIIndicatorDescription` and sets options editor PI)
- `txtNewPIIndicatorCode.Default` -> formula **A5c**
- `txtNewPIIndicatorDescription.Default` -> formula **A5c**
- `txtNewPIIndicatorCode` + `txtNewPIIndicatorDescription` + `btnNewPIForOutcome.OnSelect` -> formula **A5c** (save selected PI or add PI when none selected)
- `galPIGradeOptions.Items` -> formula **A5c** (PI-specific options)
- `btnNewPIOption.OnSelect` -> formula **A5c** (add option for selected PI)

---

## Screen AD-4: `scrSemesterDashboard`
Purpose: Import/manage `TeachingAssignments`, track completion, and control reminders.

### Layout (wireframe)
```text
+--------------------------------------------------------------------------------+
| Semester Dashboard                                                              |
+--------------------------------------------------------------------------------+
| Semester [drpSemester] [tglSemesterReminders] [txtReminderCadenceDays] [btnSaveReminderCadence] [btnSendReminderNow] |
| New Semester: [txtNewSemesterTermName] [dtNewSemesterStart] [dtNewSemesterEnd] [drpNewSemesterStatus] [btnCreateSemester] |
| [attAssignmentsImport] [btnImportAssignments]                                  |
|--------------------------------------------------------------------------------|
| Card: Pending Count                                                            |
| Card: Submitted Count                                                          |
|--------------------------------------------------------------------------------|
| Gallery [galAssignmentsBySemester] (left; blank vertical gallery like Faculty) |
|  - [lblAssignInstructor] [lblAssignCourse] [lblAssignSection] [lblAssignStatus] |
| Assignment editor (right):                                                     |
|  [drpAssignCourse] [txtAssignSection] [drpAssignInstructor]                   |
|  [drpAssignCampus] [drpAssignStatus]                                           |
|  [btnNewAssignmentAdmin] [btnSaveAssignmentAdmin] [btnDeleteAssignmentAdmin]   |
+--------------------------------------------------------------------------------+
```

### Controls & bindings
- `drpSemester.Items`:
```powerfx
SortByColumns(Semesters, "StartDate", Descending)
```

- `drpSemester.DefaultSelectedItems` (recommended):
```powerfx
If(
    CountRows(Filter(Semesters, Status.Value = "Active")) > 0,
    [First(SortByColumns(Filter(Semesters, Status.Value = "Active"), "StartDate", Descending))],
    [First(SortByColumns(Semesters, "StartDate", Descending))]
)
```

- `tglSemesterReminders.Default` -> formula **A12**
- `tglSemesterReminders.OnChange` -> formula **A12**
- `btnSaveReminderCadence.OnSelect` -> formula **A12**
- `btnImportAssignments.OnSelect` -> formula **A15**
- `btnCreateSemester.OnSelect` -> formula **A16**
- Pending card text -> formula **A10** (pending)
- Submitted card text -> formula **A10** (submitted)
- `btnSendReminderNow.OnSelect` -> formula **A11**
- `galAssignmentsBySemester.Items` -> formula **A13**
- `lblAssignInstructor.Text` -> `ThisItem.InstructorDisplayName`
- `lblAssignCourse.Text` -> `Coalesce(ThisItem.Course.Value, "")`
- `lblAssignSection.Text` -> `Coalesce(ThisItem.Section, "")`
- `lblAssignStatus.Text` -> `Coalesce(ThisItem.FormStatus.Value, "")`
- `galAssignmentsBySemester.OnSelect` -> formula **A13**
- `txtAssignSection.Default` -> `Coalesce(varSelectedAssignmentAdmin.Section, "")`
- `drpAssignCourse.Default` -> `Coalesce(varSelectedAssignmentAdmin.Course.Value, "")`
- `drpAssignInstructor.Default` -> `Coalesce(varSelectedAssignmentAdmin.Instructor.Value, Coalesce(varSelectedAssignmentAdmin.Instructor.Email, ""))`
- `drpAssignCampus.Default` -> `Coalesce(varSelectedAssignmentAdmin.Campus.Value, "")`
- `drpAssignStatus.Default` -> `Coalesce(varSelectedAssignmentAdmin.FormStatus.Value, "")`
- `btnSaveAssignmentAdmin.OnSelect` -> formula **A14**
- `btnNewAssignmentAdmin.OnSelect` -> formula **A14**
- `btnDeleteAssignmentAdmin.OnSelect` -> formula **A14**
- `galAssignmentsBySemester.Items` example:
```powerfx
AddColumns(
    Filter(TeachingAssignments, Semester.Id = drpSemester.Selected.ID),
    InstructorDisplayName,
    Coalesce(
        LookUp(Faculty, Lower(Email) = Lower(Instructor.Email), LastName & ", " & FirstName),
        Coalesce(Instructor.Email, "(No instructor email)")
    )
)
```

---

## 3) Naming Consistency Checklist

To avoid broken formulas, keep these names exactly:

- Toggles: `tglShowActiveOnly`, `tglCourseActive`, `tglQuestionRequired`, `tglSemesterReminders`
- Text inputs: `txtCourseNumber`, `txtCourseTitle`, `txtFacultyFirstName`, `txtFacultyLastName`, `txtFacultyEmail`, `txtQuestionText`, `txtDisplayOrder`, `txtChoiceOrderRow`, `txtChoiceLabelRow`, `txtOutcomeCode`, `txtOutcomeDescription`, `txtNewPIIndicatorCode`, `txtNewPIIndicatorDescription`, `txtReminderCadenceDays`, `txtAssignSection`, `txtNewSemesterTermName`
- Dropdowns: `drpFacultyCampus`, `drpQuestionType`, `drpSemester`, `drpAssignCourse`, `drpAssignInstructor`, `drpAssignCampus`, `drpAssignStatus`, `drpNewSemesterStatus`
- Instructor controls: `galAssignments`, `lblAssignCourseTitle`, `lblAssignSemester`, `lblAssignStatus`, `btnOpenFormRow`, `lblAssessmentCourse`, `lblAssessmentSemester`, `btnBackToAssignments`, `galQuestions`, `lblQuestionText`
- PI/CSO controls: `galSupportedPIs`, `galAvailablePIs`, `btnAddPI`, `btnRemovePI`, `galCSOs`, `btnNewCSO`, `btnAddCSO`, `btnRemoveCSO`, `galEvalItems`, `drpScore`, `galPIGradeOptions`, `btnNewPIOption`, `btnSavePIOptionRow`, `btnDeletePIOptionRow`
- Semester dashboard controls: `galAssignmentsBySemester`, `btnImportAssignments`, `attAssignmentsImport`, `btnNewAssignmentAdmin`, `btnSaveAssignmentAdmin`, `btnDeleteAssignmentAdmin`, `btnSaveReminderCadence`, `btnSendReminderNow`, `btnCreateSemester`, `dtNewSemesterStart`, `dtNewSemesterEnd`
- Course controls: `btnNewCourse`, `btnSaveCourse`, `btnDeleteCourse`
- Faculty controls: `galFaculty`, `btnNewFaculty`, `btnSaveFaculty`, `btnDeleteFaculty`
- Question controls: `galQuestionsAdmin`, `btnNewQuestion`, `btnSaveQuestion`, `btnDeleteQuestion`, `galChoices`, `btnNewChoice`, `btnSaveChoiceRow`, `btnDeleteChoiceRow`
- Outcome/PI controls: `galStudentOutcomesAdmin`, `galPIsByOutcome`, `btnNewOutcome`, `btnMoveUpOutcome`, `btnDeleteOutcomeRow`, `btnNewPIForOutcome`, `btnMoveUpPI`, `btnDeletePIFromOutcome`
- Variables: `varIsAdmin`, `varUserEmail`, `varSelectedCourse`, `varSelectedFaculty`, `varSelectedQuestion`, `varSelectedOutcome`, `varSelectedPIAdmin`, `varAssignmentId`, `varCourseId`, `varQuestionTextLocal`, `varQuestionTypeLocal`, `varQuestionRequiredLocal`, `varQuestionOrderLocal`, `varFacultyFirstNameLocal`, `varFacultyLastNameLocal`, `varFacultyEmailLocal`, `varFacultyCampusLocal`, `varSelectedCSO`, `varCSOCodeLocal`, `varCSODescriptionLocal`, `varOutcomeCodeLocal`, `varOutcomeDescriptionLocal`, `varPIIndicatorCodeLocal`, `varPIIndicatorDescriptionLocal`, `varSelectedAssignmentAdmin`
- Collections: `colMyAssignments`, `colQuestions`, `colResponses`

If you prefer different control names, update the formula references consistently.
