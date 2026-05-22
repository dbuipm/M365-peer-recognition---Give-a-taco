# SharePoint — Give a Taco

Two lists on your SharePoint site power the entire system.

---

## TacoTracker

Primary data store. One row per taco submitted.

**Create:** SharePoint site → New → List → name it `TacoTracker`

| Column | Type | Notes |
|---|---|---|
| Title | Single line of text | Built-in — stores Form Response ID |
| Completion time | Date and Time | Include time of day |
| Email | Single line of text | Submitter's email |
| Name | Single line of text | Submitter's Office 365 Display Name |
| Who do you want to give a taco? | Single line of text | Form Q1 answer |
| Why? | Multiple lines of text | Form Q2 answer |

**Power Automate mapping (Create item step):**

| List column | Dynamic value |
|---|---|
| Title | `Response Id` (Forms) |
| Completion time | `Submission time` (Forms) |
| Email | `Responders' Email` (Forms) |
| Name | `Display Name` (Office 365 Get User Profile) |
| Who do you want to give a taco? | `Who do you want to give a taco?` (Forms Q1) |
| Why? | `Why?` (Forms Q2) |

---

## TacoMilestoneLog

Deduplication guard. Prevents the same milestone notification from firing twice for the same person.

**Create:** SharePoint site → New → List → name it `TacoMilestoneLog`

| Column | Type | Notes |
|---|---|---|
| Title | Single line of text | Built-in |
| Person | Single line of text | Must match exactly the name in `TacosByPerson[Who]` |
| Milestone | Number | The taco count threshold (e.g. 20, 40, 60…) |

**Filter query used in Power Automate Get items step:**
```
Person eq 'PERSON_NAME' and Milestone eq MILESTONE_NUMBER
```

If this returns zero items → milestone not yet celebrated → fire notification + create row.  
If it returns any items → already logged → skip.

---

## Notes

- Column internal names can't contain spaces. If you create columns with display names that have spaces, SharePoint generates internal names with `_x0020_` — this can break OData filter queries in Power Automate. Create columns without spaces first, then rename the display name.
- Both lists live on the same SharePoint site where the Power BI report is embedded.
- `TacoMilestoneLog` only needs the two columns above — keep it simple.
