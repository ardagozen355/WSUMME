# Power Apps Screen Schematic + UI-to-Formula Map

This template describes **how to lay out app screens** and **which formula blocks should be attached to each UI item**.

It is intended to be used side-by-side with `templates/power-apps-formulas.md`.

---

## 1) Instructor App Schematic

### Screen: `scrInstructorHome`
Purpose: show pending assignments for the signed-in instructor.

```
App OnStart
  └─ Load current user + pending assignments

scrInstructorHome
  ├─ lblWelcome
  ├─ galAssignments (pending assignments)
  │   ├─ lblCourseNumber
  │   ├─ lblCourseTitle
  │   ├─ lblSemester
  │   └─ btnOpenAssessment
  └─ btnRefresh
```

#### UI item formula mapping

| UI item | Property | Formula block in `power-apps-formulas.md` | What it does |
|---|---|---|---|
| App | `OnStart` | **1) Load instructor's pending assignments** | Captures `User().Email` and loads non-submitted assignments into `colMyAssignments`. |
| `galAssignments` | `Items` | Uses `colMyAssignments` loaded in step 1 | Displays only the instructor’s pending items. |
| `btnOpenAssessment` inside gallery row | `OnSelect` | **2) Build dynamic question set for selected assignment** | Stores assignment/course IDs and builds question + local response collections. |
| `btnRefresh` | `OnSelect` | Re-run of step 1 logic (optional) | Refreshes pending assignment list without restarting app. |

---

### Screen: `scrAssessmentForm`
Purpose: render dynamic questions and capture answers.

```
scrAssessmentForm
  ├─ lblAssessmentTitle
  ├─ galQuestions (dynamic)
  │   ├─ lblQuestionText
  │   ├─ txtLongAnswer
  │   └─ drpSingleChoice
  ├─ btnSubmit
  └─ btnBack
```

#### UI item formula mapping

| UI item | Property | Formula block in `power-apps-formulas.md` | What it does |
|---|---|---|---|
| `galQuestions` | `Items` | Uses `colQuestions` from step 2 | Shows active global + course-specific questions in display order. |
| `txtLongAnswer` | `Visible` | **3) Long text input control `Visible`** | Shows text input only when `QuestionType = LongText`. |
| `drpSingleChoice` | `Visible` | **3) Single choice dropdown `Visible`** | Shows dropdown only when `QuestionType = SingleChoice`. |
| `drpSingleChoice` | `Items` | **3) Single choice dropdown `Items`** | Loads options from `QuestionChoices` for current question row. |
| `txtLongAnswer` | `OnChange` | **4) Save draft answer (TextInput OnChange)** | Writes local text answer to `colResponses`. |
| `drpSingleChoice` | `OnChange` | **5) Save draft answer (Dropdown OnChange)** | Writes local selected choice to `colResponses`. |
| `btnSubmit` | `OnSelect` | **6) Submit button (OnSelect)** | Validates all responses, writes final rows to `Responses`, marks assignment submitted. |

---

## 2) Admin App Schematic

### Screen: `scrAdminHome`
Purpose: role gate and navigate to admin modules.

```
App OnStart
  └─ Validate admin membership

scrAdminHome
  ├─ lblAdminTitle
  ├─ btnCourses
  ├─ btnQuestions
  ├─ btnSemesterDashboard
  └─ lblAccessMessage (if not admin)
```

#### UI item formula mapping

| UI item | Property | Formula block in `power-apps-formulas.md` | What it does |
|---|---|---|---|
| App | `OnStart` | **A1) Role-gate admin screens** | Checks whether user exists in `AdminUsers` list and sets `varIsAdmin`. |
| admin screens/containers | `Visible` | `varIsAdmin` (recommended usage) | Hides admin UX for non-admin users while showing access notification. |

---

### Screen: `scrCourses`
Purpose: create/edit active courses and map performance indices.

```
scrCourses
  ├─ tglShowActiveOnly
  ├─ galCourses
  │   ├─ lblCourseNumber
  │   ├─ lblCourseTitle
  │   ├─ icnEdit
  │   └─ icnActiveState
  ├─ frmCourseEditor area
  │   ├─ txtCourseNumber
  │   ├─ txtCourseTitle
  │   ├─ tglCourseActive
  │   └─ btnSaveCourse
  ├─ cmbIndices
  └─ btnSaveIndices
```

#### UI item formula mapping

| UI item | Property | Formula block in `power-apps-formulas.md` | What it does |
|---|---|---|---|
| `galCourses` | `Items` | **A2) Courses gallery `Items`** | Sorts courses and optionally filters to active-only based on toggle. |
| `btnSaveCourse` | `OnSelect` | **A3) Add/update a course** | Validates required fields, patches course row, preserves active state from toggle. |
| `cmbIndices` | `Items` | **A4) Link performance indices** (`cmbIndices.Items`) | Loads available performance indices sorted by code. |
| `btnSaveIndices` | `OnSelect` | **A4) Link performance indices** (`Save button OnSelect`) | Rebuilds course-index junction rows from current combo selections. |

> Note: there is no deactivate button formula in this template. Course availability is controlled from `tglCourseActive` in the save workflow.

---

### Screen: `scrQuestions`
Purpose: manage question bank and choice options.

```
scrQuestions
  ├─ drpQuestionScope (All / Global / CourseSpecific)
  ├─ galQuestions
  │   ├─ lblQuestionText
  │   ├─ lblType
  │   └─ btnMoveUp
  ├─ frmQuestionEditor area
  │   ├─ txtQuestionText
  │   ├─ drpQuestionType
  │   ├─ drpAppliesTo
  │   ├─ drpCourseForQuestion
  │   ├─ txtDisplayOrder
  │   ├─ tglQuestionActive
  │   └─ btnSaveQuestion
  ├─ galChoices
  │   ├─ lblChoiceLabel
  │   ├─ lblChoiceValue
  │   └─ lblChoiceOrder
  ├─ txtChoiceLabel
  ├─ txtChoiceValue
  ├─ txtChoiceOrder
  └─ btnAddChoice
```

#### UI item formula mapping

| UI item | Property | Formula block in `power-apps-formulas.md` | What it does |
|---|---|---|---|
| `galQuestions` | `Items` | **A5) Questions gallery `Items`** | Filters active questions by scope and sorts by display order. |
| `btnSaveQuestion` | `OnSelect` | **A6) Create/update a question** | Adds/updates question text, type, scope, optional course, order, active flag. |
| `galChoices` | `Items` | **A7) Maintain single-choice options** (`Choices gallery Items`) | Shows options tied to selected question. |
| `btnAddChoice` | `OnSelect` | **A7) Maintain single-choice options** (`Add option button OnSelect`) | Inserts a new choice row for selected question. |
| `btnMoveUp` | `OnSelect` | **A8) Reorder question (Move Up button)** | Swaps display order with previous question within same scope grouping. |

---

### Screen: `scrSemesterDashboard`
Purpose: monitor submission progress and trigger reminders.

```
scrSemesterDashboard
  ├─ drpSemester
  ├─ cardPendingCount
  ├─ cardSubmittedCount
  └─ btnSendReminderNow
```

#### UI item formula mapping

| UI item | Property | Formula block in `power-apps-formulas.md` | What it does |
|---|---|---|---|
| `cardPendingCount` | `Text` | **A9) Semester dashboard cards (Pending count)** | Counts non-submitted assignments for selected semester. |
| `cardSubmittedCount` | `Text` | **A9) Semester dashboard cards (Submitted count)** | Counts submitted assignments for selected semester. |
| `btnSendReminderNow` | `OnSelect` | **A10) Trigger reminder flow manually** | Calls `SendReminderNowFlow` with selected semester ID. |

---

## 3) Implementation Notes (Template Defaults)

- Use consistent variable names from the formulas file:
  - `varUserEmail`, `varIsAdmin`, `varAssignmentId`, `varCourseId`
  - `varSelectedCourse`, `varSelectedQuestion`
  - `colMyAssignments`, `colQuestions`, `colResponses`
- Keep gallery/item names aligned with formula snippets to reduce remapping effort.
- If you rename controls, update formula references immediately (especially `Self`, `ThisItem`, and dropdown names).
- Prefer one screen per module (Courses, Questions, Dashboard) to keep app behavior easy to troubleshoot.
