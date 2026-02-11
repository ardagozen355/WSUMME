# Power Apps Template Formulas

## 1) Load instructor's pending assignments (App OnStart)
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

## 2) Build dynamic question set for selected assignment (OnSelect of assignment row)
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

## 3) Control visibility for answer input
### Long text input control `Visible`
```powerfx
ThisItem.QuestionType.Value = "LongText"
```

### Single choice dropdown `Visible`
```powerfx
ThisItem.QuestionType.Value = "SingleChoice"
```

### Single choice dropdown `Items`
```powerfx
SortByColumns(
    Filter(QuestionChoices, Question.Id = ThisItem.ID),
    "DisplayOrder",
    Ascending
)
```

## 4) Save draft answer (TextInput OnChange)
```powerfx
Patch(
    colResponses,
    LookUp(colResponses, ID = ThisItem.ID),
    { AnswerTextLocal: Self.Text }
)
```

## 5) Save draft answer (Dropdown OnChange)
```powerfx
Patch(
    colResponses,
    LookUp(colResponses, ID = ThisItem.ID),
    { AnswerChoiceLocal: Self.Selected.ChoiceValue }
)
```

## 6) Submit button (OnSelect)
```powerfx
// Basic validation
If(
    CountRows(
        Filter(
            colResponses,
            (QuestionType.Value = "LongText" && IsBlank(AnswerTextLocal)) ||
            (QuestionType.Value = "SingleChoice" && IsBlank(AnswerChoiceLocal))
        )
    ) > 0,
    Notify("Please answer all questions before submitting.", NotificationType.Error),

    // Persist all response rows
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

    // Update assignment status
    Patch(
        TeachingAssignments,
        LookUp(TeachingAssignments, ID = varAssignmentId),
        { FormStatus: { Value: "Submitted" } }
    );

    // Optional: trigger a flow for Excel export
    // SubmitAssessmentFlow.Run(varAssignmentId);

    Notify("Assessment submitted successfully.", NotificationType.Success)
)
```
