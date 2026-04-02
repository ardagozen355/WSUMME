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
    IfError(
        SortByColumns(
            Questions,
            "DisplayOrder",
            Ascending
        ),
        SortByColumns(
            Questions,
            "ID",
            Ascending
        )
    )
);

// Prepare a response working collection
ClearCollect(
    colResponses,
    AddColumns(
        colQuestions,
        "AnswerTextLocal", Blank(),
        "AnswerChoiceLocal", Blank(),
        "IsRequiredLocal", Coalesce(IsRequired, true)
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
    { AnswerChoiceLocal: Self.Selected.ChoiceLabel }
)
```

### 6) Submit button (OnSelect)
```powerfx
If(
    CountRows(
        Filter(
            colResponses,
            IsRequiredLocal &&
            (
                (QuestionType.Value = "LongText" && IsBlank(AnswerTextLocal)) ||
                (QuestionType.Value = "SingleChoice" && IsBlank(AnswerChoiceLocal))
            )
        )
    ) > 0,
    Notify("Please answer all required questions before submitting.", NotificationType.Error),

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

### 7) Instructor evaluations for PIs and CSOs (PI option-based rating + assessment tools)
> Add list `OutcomeEvaluations` with fields:
- `Assignment` (Lookup -> TeachingAssignments)
- `EvaluationType` (Choice: PI, CSO)
- `ReferenceId` (Number)
- `ReferenceCode` (Text)
- `Score` (Text: for PI rows this is selected option label from `PIGradingOptions`; CSO can still use numeric/text scheme)
- `AssessmentTools` (Multiple lines of text)
- `SubmittedAt` (DateTime)

> Build evaluation items when opening an assignment:
```powerfx
ClearCollect(
    colEvalItems,
    AddColumns(
        LookUp(Courses, ID = varCourseId).SupportedPIs,
        "EvalType", "PI",
        "EvalId", Id,
        "EvalCode", Value,
        "ScoreLocal", Blank(),
        "OptionItems", SortByColumns(Filter(PIGradingOptions, PerformanceIndicator.Id = Id && IsActive = true), "DisplayOrder", Ascending),
        "AssessmentToolsLocal", Blank()
    )
);
Collect(
    colEvalItems,
    AddColumns(
        Filter(CourseSpecificOutcomes, Course.Id = varCourseId && IsActive = true),
        "EvalType", "CSO",
        "EvalId", ID,
        "EvalCode", CSOCode,
        "ScoreLocal", Blank(),
        "OptionItems", Table({ OptionLabel: "1" }, { OptionLabel: "2" }, { OptionLabel: "3" }, { OptionLabel: "4" }, { OptionLabel: "5" }),
        "AssessmentToolsLocal", Blank()
    )
)
```

> Rating dropdown `drpScore.Items`:
```powerfx
ThisItem.OptionItems
```

> Rating dropdown `OnChange`:
```powerfx
Patch(
    colEvalItems,
    ThisItem,
    { ScoreLocal: Coalesce(Self.Selected.OptionLabel, Self.Selected.Value) }
)
```

> Assessment tools text input `OnChange`:
```powerfx
Patch(
    colEvalItems,
    ThisItem,
    { AssessmentToolsLocal: Trim(Self.Text) }
)
```

> Save ratings on submit (append to existing submit logic):
```powerfx
If(
    CountRows(
        Filter(
            colEvalItems,
            IsBlank(ScoreLocal) || IsBlank(AssessmentToolsLocal)
        )
    ) > 0,
    Notify("Please select a rating option and enter assessment tools for each PI/CSO.", NotificationType.Error),

    ForAll(
        colEvalItems,
        Patch(
            OutcomeEvaluations,
            Defaults(OutcomeEvaluations),
            {
                Assignment: LookUp(TeachingAssignments, ID = varAssignmentId),
                EvaluationType: { Value: EvalType },
                ReferenceId: EvalId,
                ReferenceCode: EvalCode,
                Score: ScoreLocal,
                AssessmentTools: AssessmentToolsLocal,
                SubmittedAt: Now()
            }
        )
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
// btnOutcomesPIs.OnSelect
Navigate(scrOutcomesAndPIs, ScreenTransition.Fade)
```

```powerfx
// btnFaculty.OnSelect
Navigate(scrFaculty, ScreenTransition.Fade)
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

### A2b) When a course is selected, populate right panel fields
> `galCourses.OnSelect`:
```powerfx
Set(varSelectedCourse, ThisItem);

// Preload edit controls from selected course
Set(varCourseNumberLocal, ThisItem.CourseNumber);
Set(varCourseTitleLocal, ThisItem.CourseTitle);
Set(varCourseActiveLocal, ThisItem.IsActive);

// Build typed PI collections for stable gallery schemas
ClearCollect(colAllPIs, PerformanceIndicators);
ClearCollect(
    colSupportedPIs,
    Filter(
        colAllPIs,
        !IsBlank(varSelectedCourse) &&
        CountIf(varSelectedCourse.SupportedPIs, ID = ThisRecord.Id) > 0
    )
);
ClearCollect(
    colAvailablePIs,
    Filter(
        colAllPIs,
        IsBlank(varSelectedCourse) ||
        CountIf(varSelectedCourse.SupportedPIs, ID = ThisRecord.Id) = 0
    )
);

ClearCollect(
    colCSOs,
    SortByColumns(
        Filter(CourseSpecificOutcomes, Course.Id = varSelectedCourse.ID && IsActive = true),
        "CSOCode",
        "Ascending"
    )
);
Set(varSelectedCSO, Blank());
Set(varCSOCodeLocal, "");
Set(varCSODescriptionLocal, "")
```

> Bind right-panel controls so selected course content is immediately visible:
```powerfx
// txtCourseNumber.Default
Coalesce(varCourseNumberLocal, "")
```

```powerfx
// txtCourseTitle.Default
Coalesce(varCourseTitleLocal, "")
```

```powerfx
// tglCourseActive.Default
Coalesce(varCourseActiveLocal, true)
```

```powerfx
// colAllPIs caches PerformanceIndicators once per refresh
// galSupportedPIs.Items -> guarded colSupportedPIs formula, galAvailablePIs.Items -> colAvailablePIs
// galCSOs.Items -> colCSOs
```

This makes course number, title, active state, supported PIs, and current CSOs appear in the right panel immediately after selecting a course.

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

### A3a) New course button (`btnNewCourse.OnSelect`)
```powerfx
Set(varSelectedCourse, Blank());
Set(varCourseNumberLocal, "");
Set(varCourseTitleLocal, "");
Set(varCourseActiveLocal, true);
Clear(colSupportedPIs);
Clear(colAvailablePIs);
Clear(colCSOs);
Set(varSelectedCSO, Blank());
Set(varCSOCodeLocal, "");
Set(varCSODescriptionLocal, "");
Reset(txtCourseNumber);
Reset(txtCourseTitle);
Reset(tglCourseActive);
Reset(txtCSOCode);
Reset(txtCSODescription)
```

### A3b) Delete course button (`btnDeleteCourse.OnSelect`) — permanent delete
```powerfx
If(
    IsBlank(varSelectedCourse),
    Notify("Select a course first.", NotificationType.Warning),
    RemoveIf(CourseSpecificOutcomes, Course.Id = varSelectedCourse.ID);
    Remove(Courses, varSelectedCourse);
    Set(varSelectedCourse, Blank());
    Set(varCourseNumberLocal, "");
    Set(varCourseTitleLocal, "");
    Set(varCourseActiveLocal, true);
    Clear(colSupportedPIs);
    Clear(colAvailablePIs);
    Clear(colCSOs);
    Set(varSelectedCSO, Blank());
    Set(varCSOCodeLocal, "");
    Set(varCSODescriptionLocal, "");
    Reset(txtCourseNumber);
    Reset(txtCourseTitle);
    Reset(tglCourseActive);
    Reset(txtCSOCode);
    Reset(txtCSODescription);
    Notify("Course and related CSOs deleted.", NotificationType.Information)
)
```

### A5) Supported PI editor (show supported + available, add/remove)
> Why `AddColumns` is *not* required for supported vs available logic:
- The "not supported" calculation is done by `Filter(...)` + `LookUp(...)` on IDs.
- `AddColumns(...)` was only for creating a display/sort label column.
- Different tenants expose lookup/alias fields differently, so `AddColumns(...)` often becomes the source of parser errors.

> Which data source should `galSupportedPIs` use?
- In the gallery control, choose a **blank vertical gallery**.
- Keep the designer data-source setting unset/blank.
- Use the typed local collection populated in `galCourses.OnSelect`.

> `galSupportedPIs.Items`:
```powerfx
If(
    IsBlank(varSelectedCourse) || IsBlank(varSelectedCourse.ID),
    Filter(colAllPIs, false),
    colSupportedPIs
)
```

> Build `galAvailablePIs` step-by-step (recommended):
1. Insert a **Vertical gallery (blank)** in the right panel and rename it to `galAvailablePIs`.
2. Keep the gallery's designer data source unset (blank); do not bind it in the right-hand data pane.
3. Set `galAvailablePIs.Items` to `colAvailablePIs` (the collection prepared in `galCourses.OnSelect`).
4. Inside the gallery template, add a **Label** named `lblAvailablePI` (and turn on `Wrap`).
5. Set `lblAvailablePI.Text` to the available-row label formula below so each row shows PI code + PI description.
6. Inside the same row, add a **Button** (or icon button) named `btnAddPI` with text such as `"Add"`.
7. Set `btnAddPI.OnSelect` to the add formula below so clicking a row appends that PI to `Courses.SupportedPIs` and refreshes `varSelectedCourse`.
   - The formula uses row-context `ThisItem` (captured via `With`) so each button click applies only to that row.

> `galAvailablePIs.Items`:
```powerfx
colAvailablePIs
```

> Supported PI row label (`lblSupportedPI.Text`):
```powerfx
Coalesce(ThisItem.IndicatorCode, Text(ThisItem.ID)) & Char(10) &
Coalesce(ThisItem.IndicatorDescription, "(No PI description)")
```

> Optional for readability:
```powerfx
// lblSupportedPI.Wrap
true
```

> If `ThisItem` only shows `IsSelected` in `lblSupportedPI.Text`:
- Confirm `lblSupportedPI` is **inside** the `galSupportedPIs` row template.
- Confirm `galSupportedPIs.Items` uses the guarded A5 formula and that `colSupportedPIs` is rebuilt in `galCourses.OnSelect`.
- In Studio, reselect a course (or re-run `OnSelect`) so collections repopulate before editing row formulas.
- Ensure `btnAddPI` and `btnRemovePI` are inside their gallery templates so `ThisItem.ID` comes from the clicked row.

> Available PI row label (`lblAvailablePI.Text`):
```powerfx
Coalesce(ThisItem.IndicatorCode, Text(ThisItem.ID)) & Char(10) &
Coalesce(ThisItem.IndicatorDescription, "(No PI description)")
```

> Optional for readability:
```powerfx
// lblAvailablePI.Wrap
true
```

> Why this collection approach helps:
- `colAllPIs` is loaded once, then `colSupportedPIs`/`colAvailablePIs` are split locally so `ThisItem` keeps stable typed fields (`ID`, `IndicatorCode`, `IndicatorDescription`, `Title`).
- It avoids the previous `&&` / `!` filter warning pattern on `galSupportedPIs.Items`.
- After add/remove operations, refresh `updatedCourse` from SharePoint first, then rebuild collections from `updatedCourse.SupportedPIs` so galleries update immediately without re-selecting the course.

> Add PI button in `galAvailablePIs` row (`btnAddPI.OnSelect`):
```powerfx
With(
    {
        addId: ThisItem.ID,
        addValue: Coalesce(ThisItem.IndicatorCode, ThisItem.Title, Text(ThisItem.ID))
    },
    Patch(
        Courses,
        varSelectedCourse,
        {
            SupportedPIs:
                If(
                    CountIf(varSelectedCourse.SupportedPIs, Id = addId) > 0,
                    varSelectedCourse.SupportedPIs,
                    Ungroup(
                        Table(
                            { x: varSelectedCourse.SupportedPIs },
                            { x: Table({ Id: addId, Value: addValue }) }
                        ),
                        x
                    )
                )
        }
    )
);

With(
    { updatedCourse: LookUp(Courses, ID = varSelectedCourse.ID) },
    Set(varSelectedCourse, updatedCourse);

    // Rebuild typed PI collections after update
    ClearCollect(colAllPIs, PerformanceIndicators);
    ClearCollect(
        colSupportedPIs,
        Filter(
            colAllPIs,
            !IsBlank(updatedCourse) &&
            CountIf(updatedCourse.SupportedPIs, Id = ThisRecord.Id) > 0
        )
    );
    ClearCollect(
        colAvailablePIs,
        Filter(
            colAllPIs,
            IsBlank(updatedCourse) ||
            CountIf(updatedCourse.SupportedPIs, Id = ThisRecord.Id) = 0
        )
    )
);

Notify("PI added to course.", NotificationType.Success)
```

> Remove PI button in `galSupportedPIs` row (`btnRemovePI.OnSelect`):
```powerfx
With(
    { removeId: ThisItem.ID },
    Patch(
        Courses,
        varSelectedCourse,
        {
            SupportedPIs:
                Filter(
                    varSelectedCourse.SupportedPIs,
                    Id <> removeId
                )
        }
    )
);

With(
    { updatedCourse: LookUp(Courses, ID = varSelectedCourse.ID) },
    Set(varSelectedCourse, updatedCourse);

    // Rebuild typed PI collections after update
    ClearCollect(colAllPIs, PerformanceIndicators);
    ClearCollect(
        colSupportedPIs,
        Filter(
            colAllPIs,
            !IsBlank(updatedCourse) &&
            CountIf(updatedCourse.SupportedPIs, Id = ThisRecord.Id) > 0
        )
    );
    ClearCollect(
        colAvailablePIs,
        Filter(
            colAllPIs,
            IsBlank(updatedCourse) ||
            CountIf(updatedCourse.SupportedPIs, Id = ThisRecord.Id) = 0
        )
    )
);

Notify("PI removed from course.", NotificationType.Information)
```


### A5c) Student Outcomes + Performance Indicators management screen
> Screen idea: `scrOutcomesAndPIs` with two **blank vertical galleries** and embedded row controls.
- Left gallery `galStudentOutcomesAdmin` (all SO rows)
- Right gallery `galPIsByOutcome` (PIs for selected SO)

> `galStudentOutcomesAdmin.Items`:
```powerfx
IfError(
    SortByColumns(StudentOutcomes, "DisplayOrder", Ascending),
    SortByColumns(StudentOutcomes, "OutcomeCode", Ascending)
)
```

> `galStudentOutcomesAdmin.OnSelect` (load selected outcome into right-pane inputs):
```powerfx
Set(varSelectedOutcome, ThisItem);
Set(varOutcomeCodeLocal, Coalesce(ThisItem.OutcomeCode, ""));
Set(varOutcomeDescriptionLocal, Coalesce(ThisItem.OutcomeDescription, ""));
Reset(txtOutcomeCode);
Reset(txtOutcomeDescription)
```

> Right-pane defaults for outcome editor:
```powerfx
// txtOutcomeCode.Default
Coalesce(varOutcomeCodeLocal, "")

// txtOutcomeDescription.Default
Coalesce(varOutcomeDescriptionLocal, "")
```

> Row delete button (`btnDeleteOutcomeRow.OnSelect`) with cascade delete of related PIs:
```powerfx
RemoveIf(PerformanceIndicators, StudentOutcome.Id = ThisItem.ID);
Remove(StudentOutcomes, ThisItem);
If(varSelectedOutcome.ID = ThisItem.ID, Set(varSelectedOutcome, Blank()));
Notify("Outcome and related PIs deleted.", NotificationType.Information)
```

> Row move-up button (`btnMoveUpOutcome.OnSelect`):
```powerfx
Set(varOutcomeOrder, Coalesce(ThisItem.DisplayOrder, 0));
Set(varSwapOutcome,
    LookUp(StudentOutcomes, DisplayOrder = varOutcomeOrder - 1)
);
If(
    !IsBlank(varSwapOutcome),
    Patch(StudentOutcomes, varSwapOutcome, { DisplayOrder: varOutcomeOrder });
    Patch(StudentOutcomes, ThisItem, { DisplayOrder: varOutcomeOrder - 1 })
)
```

> Outcome save button (`btnNewOutcome.OnSelect`) — create new when none selected, otherwise save selected:
```powerfx
If(
    IsBlank(Trim(txtOutcomeCode.Text)) || IsBlank(Trim(txtOutcomeDescription.Text)),
    Notify("Outcome code and description are required.", NotificationType.Error),
    If(
        IsBlank(varSelectedOutcome),
        Patch(
            StudentOutcomes,
            Defaults(StudentOutcomes),
            {
                OutcomeCode: Trim(txtOutcomeCode.Text),
                OutcomeDescription: Trim(txtOutcomeDescription.Text),
                DisplayOrder: CountRows(StudentOutcomes) + 1
            }
        );
        Notify("Student outcome added.", NotificationType.Success),
        Patch(
            StudentOutcomes,
            varSelectedOutcome,
            {
                OutcomeCode: Trim(txtOutcomeCode.Text),
                OutcomeDescription: Trim(txtOutcomeDescription.Text)
            }
        );
        ForAll(
            Filter(PerformanceIndicators, StudentOutcome.Id = varSelectedOutcome.ID),
            Patch(PerformanceIndicators, ThisRecord, { SOCode: Trim(txtOutcomeCode.Text) })
        );
        Notify("Outcome updated.", NotificationType.Success)
    );
    Set(varSelectedOutcome, Blank());
    Set(varOutcomeCodeLocal, "");
    Set(varOutcomeDescriptionLocal, "");
    Reset(txtOutcomeCode);
    Reset(txtOutcomeDescription)
)
```

> `galPIsByOutcome.Items`:
```powerfx
With(
    { selectedOutcomeId: Coalesce(varSelectedOutcome.ID, Blank()) },
    SortByColumns(
        Filter(
            PerformanceIndicators,
            !IsBlank(selectedOutcomeId) && StudentOutcome.Id = selectedOutcomeId
        ),
        "IndicatorCode",
        Ascending
    )
)
```

> Why this shape matters: returning `[]` can drop row schema in some tenants. Filtering `PerformanceIndicators` keeps a typed table, so row controls can resolve `IndicatorCode`/`IndicatorDescription` reliably.

> PI row label (`lblPIAdmin.Text`):
```powerfx
Coalesce(ThisItem.IndicatorCode, Text(ThisItem.ID)) & " - " &
Coalesce(ThisItem.IndicatorDescription, "")
```

> If `ThisItem` only exposes `IsSelected` in `lblPIAdmin`:
- Confirm `lblPIAdmin` is inside the `galPIsByOutcome` row template (not outside the gallery).
- Confirm `galPIsByOutcome.Items` is set to formula **A5c**.
- Re-select an outcome in `galStudentOutcomesAdmin` so `varSelectedOutcome` refreshes and the PI gallery repopulates.

> Row delete button (`btnDeletePIFromOutcome.OnSelect`):
```powerfx
Remove(PerformanceIndicators, ThisItem);
Notify("PI deleted.", NotificationType.Information)
```

> Row move-up button (`btnMoveUpPI.OnSelect`):
```powerfx
Set(varPIOrder, Coalesce(ThisItem.DisplayOrder, 0));
Set(varSwapPI,
    LookUp(
        PerformanceIndicators,
        StudentOutcome.Id = varSelectedOutcome.ID && DisplayOrder = varPIOrder - 1
    )
);
If(
    !IsBlank(varSwapPI),
    Patch(PerformanceIndicators, varSwapPI, { DisplayOrder: varPIOrder });
    Patch(PerformanceIndicators, ThisItem, { DisplayOrder: varPIOrder - 1 })
)
```

> PI save button (`btnNewPIForOutcome.OnSelect`) — create new when none selected, otherwise save selected:
```powerfx
If(
    IsBlank(varSelectedOutcome),
    Notify("Select an outcome first.", NotificationType.Warning),
    IsBlank(Trim(txtNewPIIndicatorCode.Text)) || IsBlank(Trim(txtNewPIIndicatorDescription.Text)),
    Notify("PI code and description are required.", NotificationType.Error),
    If(
        IsBlank(varSelectedPIAdmin),
        Patch(
            PerformanceIndicators,
            Defaults(PerformanceIndicators),
            {
                IndicatorCode: Trim(txtNewPIIndicatorCode.Text),
                IndicatorDescription: Trim(txtNewPIIndicatorDescription.Text),
                StudentOutcome: {
                    Id: varSelectedOutcome.ID,
                    Value: varSelectedOutcome.OutcomeCode
                },
                SOCode: varSelectedOutcome.OutcomeCode,
                DisplayOrder: CountRows(Filter(PerformanceIndicators, StudentOutcome.Id = varSelectedOutcome.ID)) + 1
            }
        );
        Notify("PI added.", NotificationType.Success),
        Patch(
            PerformanceIndicators,
            varSelectedPIAdmin,
            {
                IndicatorCode: Trim(txtNewPIIndicatorCode.Text),
                IndicatorDescription: Trim(txtNewPIIndicatorDescription.Text)
            }
        );
        Notify("PI updated. Existing PI options were preserved.", NotificationType.Success)
    );
    Set(varSelectedPIAdmin, Blank());
    Set(varPIIndicatorCodeLocal, "");
    Set(varPIIndicatorDescriptionLocal, "");
    Reset(txtNewPIIndicatorCode);
    Reset(txtNewPIIndicatorDescription)
)
```



> PI-specific grading options (five configurable options per PI) on `scrOutcomesAndPIs`:
- Gallery: `galPIGradeOptions` (filtered by selected PI row)
- Row controls: `txtPIOptionLabelRow`, `txtPIOptionOrderRow`, `btnSavePIOptionRow`, `btnDeletePIOptionRow`
- New option button: `btnNewPIOption`

> `galPIGradeOptions.Items`:
```powerfx
If(
    IsBlank(varSelectedPIAdmin),
    Filter(PIGradingOptions, false),
    SortByColumns(
        Filter(PIGradingOptions, PerformanceIndicator.Id = varSelectedPIAdmin.ID && IsActive = true),
        "DisplayOrder",
        Ascending
    )
)
```

> `galPIsByOutcome.OnSelect` (load selected PI into right-pane inputs + options editor):
```powerfx
Set(varSelectedPIAdmin, ThisItem);
Set(varPIIndicatorCodeLocal, Coalesce(ThisItem.IndicatorCode, ""));
Set(varPIIndicatorDescriptionLocal, Coalesce(ThisItem.IndicatorDescription, ""));
Reset(txtNewPIIndicatorCode);
Reset(txtNewPIIndicatorDescription)
```

> Right-pane defaults for PI editor:
```powerfx
// txtNewPIIndicatorCode.Default
Coalesce(varPIIndicatorCodeLocal, "")

// txtNewPIIndicatorDescription.Default
Coalesce(varPIIndicatorDescriptionLocal, "")
```

> `btnNewPIOption.OnSelect`:
```powerfx
If(
    IsBlank(varSelectedPIAdmin),
    Notify("Select a PI first.", NotificationType.Warning),
    Patch(
        PIGradingOptions,
        Defaults(PIGradingOptions),
        {
            PerformanceIndicator: { Id: varSelectedPIAdmin.ID, Value: varSelectedPIAdmin.IndicatorCode },
            OptionLabel: "New option",
            DisplayOrder: CountRows(Filter(PIGradingOptions, PerformanceIndicator.Id = varSelectedPIAdmin.ID && IsActive = true)) + 1,
            IsActive: true
        }
    )
)
```


> PI option row save/delete (inside `galPIGradeOptions`):
```powerfx
// btnSavePIOptionRow.OnSelect
Patch(
    PIGradingOptions,
    ThisItem,
    {
        OptionLabel: Trim(txtPIOptionLabelRow.Text),
        DisplayOrder: Value(txtPIOptionOrderRow.Text),
        IsActive: true
    }
)

// btnDeletePIOptionRow.OnSelect
Remove(PIGradingOptions, ThisItem)
```
### A5d) Faculty directory management screen
> Screen idea: `scrFaculty` with a **blank vertical gallery** on the left and edit form controls on the right.

> `galFaculty.Items`:
```powerfx
SortByColumns(Faculty, "LastName", Ascending, "FirstName", Ascending)
```

> `galFaculty.OnSelect`:
```powerfx
Set(varSelectedFaculty, ThisItem);
Set(varFacultyFirstNameLocal, Coalesce(ThisItem.FirstName, ""));
Set(varFacultyLastNameLocal, Coalesce(ThisItem.LastName, ""));
Set(varFacultyEmailLocal, Coalesce(ThisItem.Email, ""));
Set(varFacultyCampusLocal, Coalesce(ThisItem.Campus.Value, "Pullman"));
Reset(txtFacultyFirstName);
Reset(txtFacultyLastName);
Reset(txtFacultyEmail);
Reset(drpFacultyCampus)
```

> Editor defaults:
```powerfx
// txtFacultyFirstName.Default
Coalesce(varFacultyFirstNameLocal, "")

// txtFacultyLastName.Default
Coalesce(varFacultyLastNameLocal, "")

// txtFacultyEmail.Default
Coalesce(varFacultyEmailLocal, "")

// drpFacultyCampus.Items
["Pullman", "Everett", "Bremerton"]

// drpFacultyCampus.Default
Coalesce(varFacultyCampusLocal, "Pullman")
```

> New faculty button (`btnNewFaculty.OnSelect`):
```powerfx
Set(varSelectedFaculty, Blank());
Set(varFacultyFirstNameLocal, "");
Set(varFacultyLastNameLocal, "");
Set(varFacultyEmailLocal, "");
Set(varFacultyCampusLocal, "Pullman");
Reset(txtFacultyFirstName);
Reset(txtFacultyLastName);
Reset(txtFacultyEmail);
Reset(drpFacultyCampus)
```

> Save faculty button (`btnSaveFaculty.OnSelect`):
```powerfx
If(
    IsBlank(Trim(txtFacultyFirstName.Text)) ||
    IsBlank(Trim(txtFacultyLastName.Text)) ||
    IsBlank(Trim(txtFacultyEmail.Text)),
    Notify("First name, last name, and email are required.", NotificationType.Error),
    Patch(
        Faculty,
        If(IsBlank(varSelectedFaculty), Defaults(Faculty), varSelectedFaculty),
        {
            FirstName: Trim(txtFacultyFirstName.Text),
            LastName: Trim(txtFacultyLastName.Text),
            Email: Lower(Trim(txtFacultyEmail.Text)),
            Campus: { Value: drpFacultyCampus.Selected.Value }
        }
    );
    Notify("Faculty saved.", NotificationType.Success)
)
```

> Delete faculty button (`btnDeleteFaculty.OnSelect`):
```powerfx
If(
    IsBlank(varSelectedFaculty),
    Notify("Select a faculty member first.", NotificationType.Warning),
    Remove(Faculty, varSelectedFaculty);
    Set(varSelectedFaculty, Blank());
    Reset(txtFacultyFirstName);
    Reset(txtFacultyLastName);
    Reset(txtFacultyEmail);
    Reset(drpFacultyCampus);
    Notify("Faculty deleted.", NotificationType.Information)
)
```

### A5b) Course-specific outcomes (CSOs) CRUD
> Use list `CourseSpecificOutcomes` with fields:
- `Course` (Lookup -> Courses)
- `CSOCode` (Text)
- `CSODescription` (Text)
- `IsActive` (Yes/No)

> Build `galCSOs` step-by-step:
1. Insert a **Vertical gallery (blank)** and rename it `galCSOs`.
2. Set `galCSOs.Items` to `colCSOs` so the list refreshes from the selected course context.
3. Inside each row add:
   - `lblCSOCode.Text`:
   ```powerfx
   ThisItem.CSOCode
   ```
   - `lblCSODescription.Text`:
   ```powerfx
   ThisItem.CSODescription
   ```
   - `btnRemoveCSO.OnSelect` formula below.
4. Add text inputs below the gallery:
   - `txtCSOCode` for code entry
   - `txtCSODescription` for description entry
   - `btnNewCSO.OnSelect` formula below
   - `btnAddCSO.OnSelect` formula below (save add/edit)

> `galCSOs.Items`:
```powerfx
colCSOs
```

> `galCSOs.OnSelect` (load selected CSO into text inputs for editing):
```powerfx
Set(varSelectedCSO, ThisItem);
Set(varCSOCodeLocal, Coalesce(ThisItem.CSOCode, ""));
Set(varCSODescriptionLocal, Coalesce(ThisItem.CSODescription, ""));
Reset(txtCSOCode);
Reset(txtCSODescription)
```

> Input defaults:
```powerfx
// txtCSOCode.Default
Coalesce(varCSOCodeLocal, "")

// txtCSODescription.Default
Coalesce(varCSODescriptionLocal, "")
```

> New CSO button `OnSelect` (clear selected CSO + text boxes):
```powerfx
Set(varSelectedCSO, Blank());
Set(varCSOCodeLocal, "");
Set(varCSODescriptionLocal, "");
Reset(txtCSOCode);
Reset(txtCSODescription)
```

> Add/Save CSO button `OnSelect` (create if new, update if selected):
```powerfx
If(
    IsBlank(varSelectedCourse),
    Notify("Select a course first.", NotificationType.Warning),
    IsBlank(Trim(txtCSOCode.Text)) || IsBlank(Trim(txtCSODescription.Text)),
    Notify("CSO code and description are required.", NotificationType.Error),
    Patch(
        CourseSpecificOutcomes,
        If(
            IsBlank(varSelectedCSO),
            Defaults(CourseSpecificOutcomes),
            LookUp(CourseSpecificOutcomes, ID = varSelectedCSO.ID)
        ),
        {
            Course: {
                Id: varSelectedCourse.ID,
                Value: varSelectedCourse.CourseNumber
            },
            CSOCode: Upper(Trim(txtCSOCode.Text)),
            CSODescription: Trim(txtCSODescription.Text),
            IsActive: true
        }
    );
    ClearCollect(
        colCSOs,
        SortByColumns(
            Filter(CourseSpecificOutcomes, Course.Id = varSelectedCourse.ID && IsActive = true),
            "CSOCode",
            Ascending
        )
    );
    Set(varSelectedCSO, Blank());
    Set(varCSOCodeLocal, "");
    Set(varCSODescriptionLocal, "");
    Reset(txtCSOCode);
    Reset(txtCSODescription);
    Notify("CSO saved.", NotificationType.Success)
)
```

> Remove CSO button `OnSelect` (hard delete):
```powerfx
Remove(CourseSpecificOutcomes, ThisItem);
If(!IsBlank(varSelectedCSO) && varSelectedCSO.ID = ThisItem.ID,
    Set(varSelectedCSO, Blank());
    Set(varCSOCodeLocal, "");
    Set(varCSODescriptionLocal, "");
    Reset(txtCSOCode);
    Reset(txtCSODescription)
);
ClearCollect(
    colCSOs,
    SortByColumns(
        Filter(CourseSpecificOutcomes, Course.Id = varSelectedCourse.ID && IsActive = true),
        "CSOCode",
        Ascending
    )
);
Notify("CSO deleted.", NotificationType.Information)
```

### A6) Questions gallery `Items` (global question bank)
> Build `galQuestionsAdmin` as a **blank vertical gallery** (same pattern as `galAvailablePIs`):
1. Insert a **Vertical gallery (blank)** named `galQuestionsAdmin`.
2. Keep the gallery's designer data source unset (blank).
3. Set `galQuestionsAdmin.Items` to the formula below.
4. Add labels `lblOrderAndText`, `lblType`, and `lblRequired` inside the row template.

> `galQuestionsAdmin.Items`:
```powerfx
IfError(
    SortByColumns(
        Questions,
        "DisplayOrder",
        Ascending
    ),
    SortByColumns(
        Questions,
        "ID",
        Ascending
    )
)
```

> Suggested row labels inside `galQuestionsAdmin`:
- `lblOrderAndText.Text`
```powerfx
Text(Coalesce(ThisItem.DisplayOrder, ThisItem.ID)) & " - " & Left(ThisItem.QuestionText, 120)
```
- `lblType.Text`
```powerfx
ThisItem.QuestionType.Value
```
- `lblRequired.Text`
```powerfx
If(Coalesce(ThisItem.IsRequired, true), "Required", "Optional")
```

> `galQuestionsAdmin.OnSelect` (load selected question into right-pane inputs):
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

### A7) Create/update a question
> `btnNewQuestion.OnSelect` (clear right panel for new entry):
```powerfx
Set(varSelectedQuestion, Blank());
Set(varQuestionTextLocal, "");
Set(varQuestionTypeLocal, "LongText");
Set(varQuestionRequiredLocal, true);
Set(varQuestionOrderLocal, "");
Reset(txtQuestionText);
Reset(drpQuestionType);
Reset(tglQuestionRequired);
Reset(txtDisplayOrder)
```

> `drpQuestionType.Items`:
```powerfx
Choices(Questions.QuestionType)
```

> Right-pane input defaults (so selected question values appear in corresponding inputs):
- `txtQuestionText.Default`
```powerfx
varQuestionTextLocal
```
- `drpQuestionType.Default`
```powerfx
Coalesce(
    LookUp(Choices(Questions.QuestionType), Value = varQuestionTypeLocal).Value,
    "LongText"
)
```
- `tglQuestionRequired.Default`
```powerfx
varQuestionRequiredLocal
```
- `txtDisplayOrder.Default`
```powerfx
varQuestionOrderLocal
```

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
            IsRequired: tglQuestionRequired.Value,
            DisplayOrder: Value(txtDisplayOrder.Text)
        }
    );

    Notify("Question saved.", NotificationType.Success)
)
```

### A7a) Delete selected question (permanent delete)
```powerfx
If(
    IsBlank(varSelectedQuestion),
    Notify("Select a question to delete.", NotificationType.Warning),

    Remove(Questions, varSelectedQuestion);
    Set(varSelectedQuestion, Blank());
    Set(varQuestionTextLocal, "");
    Set(varQuestionTypeLocal, "LongText");
    Set(varQuestionRequiredLocal, true);
    Set(varQuestionOrderLocal, "");
    Reset(txtQuestionText);
    Reset(drpQuestionType);
    Reset(tglQuestionRequired);
    Reset(txtDisplayOrder);
    Notify("Question deleted.", NotificationType.Information)
)
```

### A8) Maintain single-choice options for selected question (embedded row controls)
> Build `galChoices` as a **blank vertical gallery** (same pattern as `galQuestionsAdmin`):
1. Insert **Vertical gallery (blank)** named `galChoices`.
2. Keep the gallery data source unset in the designer.
3. Set `galChoices.Items` to the formula below.
4. Inside each row add embedded controls:
   - `txtChoiceOrderRow`
   - `txtChoiceLabelRow`
   - `btnSaveChoiceRow`
   - `btnDeleteChoiceRow`

> Optional visibility (show only for single-choice questions):
```powerfx
Coalesce(varQuestionTypeLocal, "LongText") = "SingleChoice"
```

> `galChoices.Items`:
```powerfx
If(
    IsBlank(varSelectedQuestion),
    [],
    SortByColumns(
        Filter(QuestionChoices, Question.Id = varSelectedQuestion.ID),
        "DisplayOrder",
        Ascending
    )
)
```

> Embedded row defaults:
- `txtChoiceOrderRow.Default`
```powerfx
Text(ThisItem.DisplayOrder)
```
- `txtChoiceLabelRow.Default`
```powerfx
ThisItem.ChoiceLabel
```
> Save row button (`btnSaveChoiceRow.OnSelect`):
```powerfx
Patch(
    QuestionChoices,
    ThisItem,
    {
        ChoiceLabel: Trim(txtChoiceLabelRow.Text),
        DisplayOrder: Value(txtChoiceOrderRow.Text)
    }
);
Notify("Choice updated.", NotificationType.Success)
```

> Delete row button (`btnDeleteChoiceRow.OnSelect`):
```powerfx
Remove(QuestionChoices, ThisItem);
Notify("Choice deleted.", NotificationType.Information)
```

> Note: `Question` is a SharePoint lookup, so this patch uses lookup-record shape (`Id` + `Value`) to avoid schema mismatch errors.

> New choice button (`btnNewChoice.OnSelect`):
```powerfx
If(
    IsBlank(varSelectedQuestion),
    Notify("Select a question first.", NotificationType.Warning),
    Patch(
        QuestionChoices,
        Defaults(QuestionChoices),
        {
            Question: {
                Id: varSelectedQuestion.ID,
                Value: Left(Coalesce(varSelectedQuestion.QuestionText, Text(varSelectedQuestion.ID)), 255)
            },
            ChoiceLabel: "",
            DisplayOrder: CountRows(Filter(QuestionChoices, Question.Id = varSelectedQuestion.ID)) + 1
        }
    )
)
```

### A9) Reorder question (Move Up button)
```powerfx
Set(varCurrentOrder, ThisItem.DisplayOrder);
Set(varSwapQuestion,
    LookUp(
        Questions,
        DisplayOrder = varCurrentOrder - 1
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
