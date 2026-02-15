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
| Gallery [galAssignments]                                                       |
|  - CourseNumber - CourseTitle                                                  |
|  - Semester                                                                     |
|  - Status badge (Sent/InProgress)                                              |
|  - Button: "Open Form"                                                        |
+--------------------------------------------------------------------------------+
```

### Controls
- `galAssignments.Items` -> `colMyAssignments`
- Open button `OnSelect`:
  - set `varAssignmentId` and `varCourseId`
  - build `colQuestions` and `colResponses`
  - `Navigate(scrAssessmentForm, ScreenTransition.Fade)`

(Uses formulas from section **Instructor 1 & 2**.)

---

## Screen IA-2: `scrAssessmentForm`
Purpose: Instructor answers dynamic questions and rates PI/CSO performance on a 1-5 scale.

### Layout (wireframe)
```text
+--------------------------------------------------------------------------------+
| Back | Course: <selected course> | Semester: <selected semester>              |
+--------------------------------------------------------------------------------+
| Scrollable gallery [galQuestions]                                              |
|  Q1. <QuestionText>                                                            |
|      [txtLongAnswer]  (Visible when LongText)                                  |
|      [drpSingleChoice] (Visible when SingleChoice)                             |
|--------------------------------------------------------------------------------|
|  Q2. <QuestionText> ...                                                        |
+--------------------------------------------------------------------------------+
| Ratings [galEvalItems]: [lblEvalCode] [drpScore (1-5)]                         |
| [btnSubmitAssessment]                                                          |
+--------------------------------------------------------------------------------+
```

### Controls
- `galQuestions.Items` -> `colResponses`
- `txtLongAnswer.Visible` -> `ThisItem.QuestionType.Value = "LongText"`
- `drpSingleChoice.Visible` -> `ThisItem.QuestionType.Value = "SingleChoice"`
- `drpSingleChoice.Items` -> `Filter(QuestionChoices, Question.Id = ThisItem.ID)` sorted by `DisplayOrder`
- `txtLongAnswer.OnChange` and `drpSingleChoice.OnChange` patch `colResponses`
- `galEvalItems.Items` -> `colEvalItems` (PI + CSO items)
- `drpScore.Items` -> `[1,2,3,4,5]`
- `drpScore.OnChange` patches `colEvalItems.ScoreLocal`
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
| [btnCourses] [btnQuestions] [btnSemesterDashboard] [btnImports]               |
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
  - `btnCourses`, `btnQuestions`, `btnSemesterDashboard`, `btnImports`
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
|                              | [btnSaveCourse]                                  |
|                              |--------------------------------------------------|
|                              | Supported PIs [galSupportedPIs]                   |
|                              |  - [lblSupportedPI] [btnRemovePI]                 |
|                              |--------------------------------------------------|
|                              | Available PIs [galAvailablePIs]                   |
|                              |  - [lblAvailablePI] [btnAddPI]                    |
|                              |--------------------------------------------------|
|                              | Course-specific outcomes [galCSOs]                |
|                              | [txtCSOCode] [txtCSODescription] [btnAddCSO]      |
+------------------------------+-------------------------------------------------+
```

### Controls & bindings
When a course is selected in `galCourses`, the right panel immediately shows that course's number, title, active status, the supported PIs list, the available (not-yet-supported) PIs list, and existing CSOs.

- `galCourses.Items` -> formula **A2**
- `galCourses.OnSelect` -> formula **A2b** (sets `varSelectedCourse` and preloads right-panel fields)
- `txtCourseNumber.Default` -> formula **A2b**
- `txtCourseTitle.Default` -> formula **A2b**
- `tglCourseActive.Default` -> formula **A2b**
- `galSupportedPIs` control type: Vertical gallery (blank)
- `galSupportedPIs` data source selection in designer: **None/blank** (do not pre-bind to `PerformanceIndicators`)
- `galSupportedPIs.Items` -> formula **A5** (currently supported PIs for selected course)
- `lblSupportedPI.Text` -> formula **A5** (uses typed `PerformanceIndicators` fields such as `IndicatorCode`/`Title`)
- `galAvailablePIs` control type: Vertical gallery (blank)
- `galAvailablePIs.Items` -> formula **A5** (all other PIs not yet supported)
- `lblAvailablePI.Text` -> formula **A5**
- `btnAddPI.OnSelect` -> formula **A5** (add PI to selected course)
- `btnRemovePI.OnSelect` -> formula **A5** (remove PI from selected course)
- `galCSOs.Items` -> formula **A2b/A5b** (shows CSOs for selected course)
- `btnSaveCourse.OnSelect` -> formula **A3**
- `btnAddCSO.OnSelect` -> formula **A5b** (add CSO)
- `btnRemoveCSO.OnSelect` -> formula **A5b** (soft remove CSO)

---

## Screen AD-3: `scrQuestions`
Purpose: Question bank management and ordering.

### Layout (wireframe)
```text
+--------------------------------------------------------------------------------+
| Questions                                                                       |
+------------------------------+-------------------------------------------------+
| Left pane                    | Right pane                                      |
| Scope [drpQuestionScope]     | Question editor                                 |
| [galQuestionsAdmin]          | Text [txtQuestionText]                          |
|  - DisplayOrder + Text       | Type [drpQuestionType]                          |
|  - AppliesTo                 | AppliesTo [drpAppliesTo]                        |
|  - Course                    | Course [drpCourseForQuestion]                   |
|  - [btnMoveUp]               | Order [txtDisplayOrder]                         |
|                              | Active [tglQuestionActive]                      |
|                              | [btnSaveQuestion]                               |
|                              |--------------------------------------------------|
|                              | Single-choice options                            |
|                              | [galChoices]                                     |
|                              | Label [txtChoiceLabel] Value [txtChoiceValue]   |
|                              | Order [txtChoiceOrder] [btnAddChoice]           |
+------------------------------+-------------------------------------------------+
```

### Controls & bindings
When a course is selected in `galCourses`, the right panel immediately shows that course's number, title, active status, the supported PIs list, the available (not-yet-supported) PIs list, and existing CSOs.

- `galQuestionsAdmin.Items` -> formula **A6**
- `galQuestionsAdmin.OnSelect`:
```powerfx
Set(varSelectedQuestion, ThisItem)
```
- `btnSaveQuestion.OnSelect` -> formula **A7**
- `galChoices.Items` -> formula **A8** (items)
- `btnAddChoice.OnSelect` -> formula **A8** (add)
- `btnMoveUp.OnSelect` -> formula **A9**

---

## Screen AD-4: `scrSemesterDashboard`
Purpose: Track completion and send reminders.

### Layout (wireframe)
```text
+--------------------------------------------------------------------------------+
| Semester Dashboard                                                              |
+--------------------------------------------------------------------------------+
| Semester [drpSemester]   [btnSendReminderNow]                                  |
|--------------------------------------------------------------------------------|
| Card: Pending Count                                                            |
| Card: Submitted Count                                                          |
|--------------------------------------------------------------------------------|
| Gallery [galAssignmentsBySemester]                                             |
|  - Instructor                                                                  |
|  - Course                                                                      |
|  - FormStatus                                                                  |
+--------------------------------------------------------------------------------+
```

### Controls & bindings
When a course is selected in `galCourses`, the right panel immediately shows that course's number, title, active status, the supported PIs list, the available (not-yet-supported) PIs list, and existing CSOs.

- Pending card text -> formula **A10** (pending)
- Submitted card text -> formula **A10** (submitted)
- `btnSendReminderNow.OnSelect` -> formula **A11**
- `galAssignmentsBySemester.Items` example:
```powerfx
Filter(TeachingAssignments, Semester.Id = drpSemester.Selected.ID)
```

---

## 3) Naming Consistency Checklist

To avoid broken formulas, keep these names exactly:

- Toggles: `tglShowActiveOnly`, `tglCourseActive`, `tglQuestionActive`
- Text inputs: `txtCourseNumber`, `txtCourseTitle`, `txtQuestionText`, `txtDisplayOrder`, `txtChoiceLabel`, `txtChoiceValue`, `txtChoiceOrder`
- Dropdowns: `drpQuestionScope`, `drpQuestionType`, `drpAppliesTo`, `drpCourseForQuestion`, `drpSemester`
- PI/CSO controls: `galSupportedPIs`, `galAvailablePIs`, `btnAddPI`, `btnRemovePI`, `galCSOs`, `btnAddCSO`, `btnRemoveCSO`, `galEvalItems`, `drpScore`
- Variables: `varIsAdmin`, `varUserEmail`, `varSelectedCourse`, `varSelectedQuestion`, `varAssignmentId`, `varCourseId`
- Collections: `colMyAssignments`, `colQuestions`, `colResponses`

If you prefer different control names, update the formula references consistently.
