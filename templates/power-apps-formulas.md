# Power Apps Template Formulas

This file includes both:
- **Instructor App formulas** (assessment completion)
- **Admin App formulas** (courses, indices, questions, ordering, semester monitoring)

---

## Instructor App Formulas

### 1) Load instructor's pending assignments (App OnStart)
```powerfx
Set(varUserEmail, Lower(User().Email));
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

---

## Admin App Formulas

### A1) Role-gate admin screens (App OnStart)
> Create a SharePoint list `AdminUsers` with a text column `Email`.

```powerfx
Set(varUserEmail, Lower(User().Email));
Set(
    varIsAdmin,
    CountRows(Filter(AdminUsers, Lower(Email) = varUserEmail)) > 0
);

If(
    !varIsAdmin,
    Notify("You do not have access to the admin interface.", NotificationType.Error)
)
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

### A4) Link performance indices to a course (multi-select combo + save)
> Combo box `cmbIndices.Items`:
```powerfx
SortByColumns(PerformanceIndices, "IndexCode", Ascending)
```

> Save button `OnSelect`:
```powerfx
// Remove existing links
RemoveIf(CoursePerformanceIndices, Course.Id = varSelectedCourse.ID);

// Add selected links
ForAll(
    cmbIndices.SelectedItems,
    Patch(
        CoursePerformanceIndices,
        Defaults(CoursePerformanceIndices),
        {
            Course: varSelectedCourse,
            PerformanceIndex: ThisRecord
        }
    )
);

Notify("Performance indices updated.", NotificationType.Success)
```

### A5) Questions gallery `Items` (filtered by course + global)
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

### A6) Create/update a question
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

### A7) Maintain single-choice options for selected question
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

### A8) Reorder question (Move Up button)
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

### A9) Semester dashboard cards (counts)
```powerfx
// Pending count
CountRows(Filter(TeachingAssignments, Semester.Id = drpSemester.Selected.ID && FormStatus.Value <> "Submitted"))
```

```powerfx
// Submitted count
CountRows(Filter(TeachingAssignments, Semester.Id = drpSemester.Selected.ID && FormStatus.Value = "Submitted"))
```

### A10) Trigger reminder flow manually
> Add a Power Automate flow connection named `SendReminderNowFlow`.

```powerfx
SendReminderNowFlow.Run(drpSemester.Selected.ID);
Notify("Reminder flow started.", NotificationType.Success)
```
