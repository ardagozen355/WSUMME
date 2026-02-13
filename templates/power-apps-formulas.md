# Power Apps Template Formulas

This file includes both:
- **Instructor App formulas** (assessment completion)
- **Admin App formulas** (courses, student outcomes, performance indicators, questions, ordering, semester monitoring)

---

## Important: How to handle `OnStart` (one app vs two apps)

You have two valid implementation patterns:

1. **Two separate Canvas apps (recommended for simplicity)**
   - Instructor app uses the Instructor `OnStart` formula only.
   - Admin app uses the Admin role-check `OnStart` formula only.

2. **One combined Canvas app with role-based navigation**
   - Use a single merged `App.OnStart` formula below.
   - The app decides whether to open admin home or instructor assignments.

### Combined `App.OnStart` (single-app pattern)
```powerfx
Set(varUserEmail, Lower(User().Email));
Set(varDisplayName, Coalesce(User().FullName, User().Email));

// Determine admin membership
Set(
    varIsAdmin,
    CountRows(Filter(AdminUsers, Lower(Email) = varUserEmail)) > 0
);

// Always preload instructor assignments (safe for admins too)
ClearCollect(
    colMyAssignments,
    Filter(
        TeachingAssignments,
        Lower(InstructorEmail) = varUserEmail && FormStatus.Value <> "Submitted"
    )
);

// Do not Navigate() from OnStart; use App.StartScreen for routing.
```

### Preview note: why `OnStart` may look like it is not running
In Power Apps Studio preview, `App.OnStart` is not always re-executed automatically.
Use this checklist:

1. In Studio, run **App -> Run OnStart** after editing startup formulas.
2. Add a temporary debug label with:
```powerfx
"user=" & Coalesce(varUserEmail, "<blank>") &
" | role=" & If(varIsAdmin, "admin", "non-admin")
```
3. Prefer **`App.StartScreen`** for first-screen routing (instead of `Navigate()` in `OnStart`).

### `App.StartScreen` formula (recommended)
Set the app's `StartScreen` property to:
```powerfx
If(varIsAdmin, scrAdminHome, scrMyAssignments)
```

> If `varIsAdmin` is blank on first load in your tenant, use this deterministic StartScreen formula that does not depend on `OnStart` timing:
```powerfx
If(
    CountRows(Filter(AdminUsers, Lower(Email) = Lower(User().Email))) > 0,
    scrAdminHome,
    scrMyAssignments
)
```

### Optional: refresh on screen visibility
Because `App.OnStart` does not rerun automatically after publish/session changes, place this on `scrMyAssignments.OnVisible` too:

```powerfx
Set(varUserEmail, Lower(User().Email));
ClearCollect(
    colMyAssignments,
    Filter(
        TeachingAssignments,
        Lower(InstructorEmail) = varUserEmail && FormStatus.Value <> "Submitted"
    )
)
```

---

## Instructor App Formulas

### 1) Load instructor's pending assignments (App OnStart)
```powerfx
Set(varUserEmail, Lower(User().Email));
Set(varDisplayName, Coalesce(User().FullName, User().Email));
ClearCollect(
    colMyAssignments,
    Filter(
        TeachingAssignments,
        Lower(InstructorEmail) = varUserEmail && FormStatus <> "Submitted"
    )
);
```

### 2) Build dynamic question set for selected assignment (OnSelect of assignment row)
```powerfx
Set(varAssignmentId, ThisItem.ID);
Set(varCourseId, ThisItem.Course.Id);
ClearCollect(
    colQuestions,
    SortByColumns(
        Filter(
            Questions,
            IsActive = true &&
            (
                AppliesTo.Value = "Global" ||
                (AppliesTo.Value = "CourseSpecific" && Course.Id = varCourseId)
            )
        ),
        "DisplayOrder",
        Ascending
    )
);

// Prepare a response working collection
ClearCollect(
    colResponses,
    AddColumns(
        colQuestions,
        "AnswerTextLocal", Blank(),
        "AnswerChoiceLocal", Blank()
    )
);
```

### 3) Control visibility for answer input
#### Long text input control `Visible`
```powerfx
ThisItem.QuestionType.Value = "LongText"
```

#### Single choice dropdown `Visible`
```powerfx
ThisItem.QuestionType.Value = "SingleChoice"
```

#### Single choice dropdown `Items`
```powerfx
SortByColumns(
    Filter(QuestionChoices, Question.Id = ThisItem.ID),
    "DisplayOrder",
    Ascending
)
```

### 4) Save draft answer (TextInput OnChange)
```powerfx
Patch(
    colResponses,
    LookUp(colResponses, ID = ThisItem.ID),
    { AnswerTextLocal: Self.Text }
)
```

### 5) Save draft answer (Dropdown OnChange)
```powerfx
Patch(
    colResponses,
    LookUp(colResponses, ID = ThisItem.ID),
    { AnswerChoiceLocal: Self.Selected.ChoiceValue }
)
```

### 6) Submit button (OnSelect)
```powerfx
If(
    CountRows(
        Filter(
            colResponses,
            (QuestionType.Value = "LongText" && IsBlank(AnswerTextLocal)) ||
            (QuestionType.Value = "SingleChoice" && IsBlank(AnswerChoiceLocal))
        )
    ) > 0,
    Notify("Please answer all questions before submitting.", NotificationType.Error),

    ForAll(
        colResponses,
        Patch(
            Responses,
            Defaults(Responses),
            {
                Assignment: LookUp(TeachingAssignments, ID = varAssignmentId),
                Question: LookUp(Questions, ID = ThisRecord.ID),
                AnswerText: ThisRecord.AnswerTextLocal,
                AnswerChoice: ThisRecord.AnswerChoiceLocal,
                SubmittedAt: Now()
            }
        )
    );

    Patch(
        TeachingAssignments,
        LookUp(TeachingAssignments, ID = varAssignmentId),
        { FormStatus: { Value: "Submitted" } }
    );

    Notify("Assessment submitted successfully.", NotificationType.Success)
)
```

### 7) Instructor ratings for PIs and CSOs (1-5 scale)
> Add list `OutcomeEvaluations` with fields:
- `Assignment` (Lookup -> TeachingAssignments)
- `EvaluationType` (Choice: PI, CSO)
- `ReferenceId` (Number)
- `ReferenceCode` (Text)
- `Score` (Number)
- `SubmittedAt` (DateTime)

> Build evaluation items when opening an assignment:
```powerfx
ClearCollect(
    colEvalItems,
    AddColumns(
        LookUp(Courses, ID = varCourseId).SupportedPIs,
        "EvalType", "PI",
        "EvalCode", StudentOutcome.OutcomeCode & "-" & IndicatorCode,
        "ScoreLocal", Blank()
    )
);
Collect(
    colEvalItems,
    AddColumns(
        Filter(CourseSpecificOutcomes, Course.Id = varCourseId && IsActive = true),
        "EvalType", "CSO",
        "EvalCode", CSOCode,
        "ScoreLocal", Blank()
    )
)
```

> Rating dropdown `drpScore.Items`:
```powerfx
[1,2,3,4,5]
```

> Rating dropdown `OnChange`:
```powerfx
Patch(
    colEvalItems,
    ThisItem,
    { ScoreLocal: Value(Self.Selected.Value) }
)
```

> Save ratings on submit (append to existing submit logic):
```powerfx
ForAll(
    colEvalItems,
    Patch(
        OutcomeEvaluations,
        Defaults(OutcomeEvaluations),
        {
            Assignment: LookUp(TeachingAssignments, ID = varAssignmentId),
            EvaluationType: { Value: EvalType },
            ReferenceId: ID,
            ReferenceCode: EvalCode,
            Score: ScoreLocal,
            SubmittedAt: Now()
        }
    )
)
```

---

## Admin App Formulas

### A1) Role-gate admin screens (App OnStart)
> Create a SharePoint list `AdminUsers` with a text column `Email`.

```powerfx
Set(varUserEmail, Lower(User().Email));
Set(varDisplayName, Coalesce(User().FullName, User().Email));
Set(
    varIsAdmin,
    CountRows(Filter(AdminUsers, Lower(Email) = varUserEmail)) > 0
);

If(
    !varIsAdmin,
    Notify("You do not have access to the admin interface.", NotificationType.Error)
)
```

### A1a) How AdminUsers is recognized by the app (setup checklist)
1. Add SharePoint list **AdminUsers** as a data source in the app:
   - Power Apps Studio -> Data -> Add data -> SharePoint -> select site -> choose `AdminUsers`.
2. Ensure list has a text column named exactly **Email**.
3. Store admin emails in lowercase (recommended), e.g. `jane.doe@university.edu`.
4. Grant app users at least **Read** permission to `AdminUsers` list.
5. On app load, `User().Email` is captured into `varUserEmail`, then matched by:

```powerfx
CountRows(Filter(AdminUsers, Lower(Email) = varUserEmail)) > 0
```

If that expression returns `true`, `varIsAdmin` becomes `true` and admin buttons/screens are enabled.

### A1a-Troubleshooting quick checks
```powerfx
// Put in a temporary debug label (Text)
"user=" & varUserEmail &
" | matches=" & Text(CountRows(Filter(AdminUsers, Lower(Email) = varUserEmail))) &
" | isAdmin=" & Text(varIsAdmin)
```

Common causes when admins are not recognized:
- `AdminUsers` list was not added as a data source in the app.
- Column name is not exactly `Email`.
- Email value has trailing spaces or different account alias than `User().Email`.
- User lacks read permission to `AdminUsers`.

### A1b) Admin Home labels and navigation buttons
> Note: `varDisplayName` is used instead of directly calling `User().FullName` to avoid blank-name tenant/profile edge cases.

```powerfx
// lblAdminName.Text
"Admin: " & varDisplayName
```

```powerfx
// lblRoleValue.Text
If(varIsAdmin, "Admin access granted", "No admin access")
```

```powerfx
// btnCourses.OnSelect
Navigate(scrCourses, ScreenTransition.Fade)
```

```powerfx
// btnQuestions.OnSelect
Navigate(scrQuestions, ScreenTransition.Fade)
```

```powerfx
// btnSemesterDashboard.OnSelect
Navigate(scrSemesterDashboard, ScreenTransition.Fade)
```

```powerfx
// btnImports.OnSelect (optional screen)
Navigate(scrImports, ScreenTransition.Fade)
```

```powerfx
// Any admin nav button DisplayMode
If(varIsAdmin, DisplayMode.Edit, DisplayMode.Disabled)
```

### A2) Courses gallery `Items`
```powerfx
SortByColumns(
    Filter(Courses, IsActive = tglShowActiveOnly.Value || !tglShowActiveOnly.Value),
    "CourseNumber",
    Ascending
)
```

### A3) Add/update a course (Save button `OnSelect`)
```powerfx
If(
    IsBlank(txtCourseNumber.Text) || IsBlank(txtCourseTitle.Text),
    Notify("Course number and title are required.", NotificationType.Error),

    Patch(
        Courses,
        If(IsBlank(varSelectedCourse), Defaults(Courses), varSelectedCourse),
        {
            CourseNumber: Upper(Trim(txtCourseNumber.Text)),
            CourseTitle: Trim(txtCourseTitle.Text),
            IsActive: tglCourseActive.Value
        }
    );

    Notify("Course saved.", NotificationType.Success);
    Reset(txtCourseNumber);
    Reset(txtCourseTitle)
)
```

### A4) Soft-delete (deactivate) a course
```powerfx
Patch(
    Courses,
    varSelectedCourse,
    { IsActive: false }
);
Notify("Course deactivated.", NotificationType.Information)
```

### A5) Map supported PIs on a course (no separate CoursePerformanceIndicators list)
> Assumption: `Courses` has a **multi-lookup column** `SupportedPIs` -> `PerformanceIndicators`.

> Combo box `cmbSupportedPIs.Items` (explicit SO -> PI display):
```powerfx
SortByColumns(
    AddColumns(
        PerformanceIndicators,
        "SO_PI_Label",
        StudentOutcome.OutcomeCode & " - " & IndicatorCode
    ),
    "SO_PI_Label",
    Ascending
)
```

> Optional label inside combo template (`lblPILink.Text`):
```powerfx
ThisItem.StudentOutcome.OutcomeCode & " - " & ThisItem.IndicatorCode
```

> Save button `OnSelect`:
```powerfx
Patch(
    Courses,
    varSelectedCourse,
    {
        SupportedPIs: cmbSupportedPIs.SelectedItems
    }
);
Notify("Supported PIs updated for course.", NotificationType.Success)
```

### A5b) Course-specific outcomes (CSOs) CRUD
> Use list `CourseSpecificOutcomes` with fields:
- `Course` (Lookup -> Courses)
- `CSOCode` (Text)
- `CSODescription` (Text)
- `IsActive` (Yes/No)

> `galCSOs.Items`:
```powerfx
SortByColumns(
    Filter(CourseSpecificOutcomes, Course.Id = varSelectedCourse.ID && IsActive = true),
    "CSOCode",
    Ascending
)
```

> Add CSO button `OnSelect`:
```powerfx
Patch(
    CourseSpecificOutcomes,
    Defaults(CourseSpecificOutcomes),
    {
        Course: varSelectedCourse,
        CSOCode: Upper(Trim(txtCSOCode.Text)),
        CSODescription: Trim(txtCSODescription.Text),
        IsActive: true
    }
);
Notify("Course-specific outcome added.", NotificationType.Success)
```

> Remove CSO button `OnSelect` (soft delete):
```powerfx
Patch(
    CourseSpecificOutcomes,
    ThisItem,
    { IsActive: false }
)
```

### A6) Questions gallery `Items` (filtered by course + global)
```powerfx
SortByColumns(
    Filter(
        Questions,
        IsActive = true &&
        (
            drpQuestionScope.Selected.Value = "All" ||
            (drpQuestionScope.Selected.Value = "Global" && AppliesTo.Value = "Global") ||
            (drpQuestionScope.Selected.Value = "CourseSpecific" && AppliesTo.Value = "CourseSpecific")
        )
    ),
    "DisplayOrder",
    Ascending
)
```

### A7) Create/update a question
```powerfx
If(
    IsBlank(txtQuestionText.Text),
    Notify("Question text is required.", NotificationType.Error),

    Patch(
        Questions,
        If(IsBlank(varSelectedQuestion), Defaults(Questions), varSelectedQuestion),
        {
            QuestionText: Trim(txtQuestionText.Text),
            QuestionType: { Value: drpQuestionType.Selected.Value },
            AppliesTo: { Value: drpAppliesTo.Selected.Value },
            Course: If(drpAppliesTo.Selected.Value = "CourseSpecific", drpCourseForQuestion.Selected, Blank()),
            DisplayOrder: Value(txtDisplayOrder.Text),
            IsActive: tglQuestionActive.Value
        }
    );

    Notify("Question saved.", NotificationType.Success)
)
```

### A8) Maintain single-choice options for selected question
> Choices gallery `Items`:
```powerfx
SortByColumns(
    Filter(QuestionChoices, Question.Id = varSelectedQuestion.ID),
    "DisplayOrder",
    Ascending
)
```

> Add option button `OnSelect`:
```powerfx
Patch(
    QuestionChoices,
    Defaults(QuestionChoices),
    {
        Question: varSelectedQuestion,
        ChoiceLabel: Trim(txtChoiceLabel.Text),
        ChoiceValue: Trim(txtChoiceValue.Text),
        DisplayOrder: Value(txtChoiceOrder.Text)
    }
);
Notify("Choice added.", NotificationType.Success)
```

### A9) Reorder question (Move Up button)
```powerfx
Set(varCurrentOrder, ThisItem.DisplayOrder);
Set(varSwapQuestion,
    LookUp(
        Questions,
        DisplayOrder = varCurrentOrder - 1 &&
        ((AppliesTo.Value = "Global" && ThisItem.AppliesTo.Value = "Global") ||
         (AppliesTo.Value = "CourseSpecific" && Course.Id = ThisItem.Course.Id))
    )
);

If(
    !IsBlank(varSwapQuestion),
    Patch(Questions, varSwapQuestion, { DisplayOrder: varCurrentOrder });
    Patch(Questions, ThisItem, { DisplayOrder: varCurrentOrder - 1 })
)
```

### A10) Semester dashboard cards (counts)
```powerfx
// Pending count
CountRows(Filter(TeachingAssignments, Semester.Id = drpSemester.Selected.ID && FormStatus.Value <> "Submitted"))
```

```powerfx
// Submitted count
CountRows(Filter(TeachingAssignments, Semester.Id = drpSemester.Selected.ID && FormStatus.Value = "Submitted"))
```

### A11) Trigger reminder flow manually
> Add a Power Automate flow connection named `SendReminderNowFlow`.

```powerfx
SendReminderNowFlow.Run(drpSemester.Selected.ID);
Notify("Reminder flow started.", NotificationType.Success)
```
