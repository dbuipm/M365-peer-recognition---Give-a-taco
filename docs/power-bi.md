# Power BI — Give a Taco

---

## Dataset

**Name:** `TacoLeaderboard`  
**Source:** `TacoTracker` SharePoint list (DirectQuery or scheduled refresh)

---

## Calculated Table — TacosByPerson

Create this table in Power BI Desktop. It aggregates the raw list into one row per person with their total taco count. The DAX milestone query in Power Automate runs against this table.

```dax
TacosByPerson =
SUMMARIZE(
    TacoTracker,
    TacoTracker[Who do you want to give a taco?],
    "Who", TacoTracker[Who do you want to give a taco?],
    "TotalTacos", COUNT(TacoTracker[Title])
)
```

---

## Measures

**Team Total Tacos**
```dax
Team Total Tacos = COUNTROWS(TacoTracker)
```

**Taco Champ**
```dax
Taco Champ =
FIRSTNONBLANK(
    TOPN(
        1,
        TacosByPerson,
        TacosByPerson[TotalTacos],
        DESC
    ),
    TacosByPerson[Who]
)
```

---

## Report Visuals

| Visual | Type | Fields | Notes |
|---|---|---|---|
| Leaderboard | Horizontal bar chart | Y: `Who`, X: `TotalTacos` | Sort descending; color gradient top to bottom |
| Team Total Tacos | Card | `Team Total Tacos` measure | Used for data-driven alert |
| The Taco Champ is | Card | `Taco Champ` measure | |
| The assistance and compliments | Table | `Why?` column | Scrollable; shows all recognition reasons |

---

## Data-Driven Alert (Team Milestone)

Set on the **Team Total Tacos** card to trigger the `TacoTeamAlert` Power Automate flow.

1. Power BI Service → open the report
2. Click the Team Total Tacos card → 🔔 → **Manage alerts**
3. **+ Add alert rule**
   - Condition: Above
   - Threshold: `300`
   - Notification frequency: At most once per day
4. Save — this alert now appears in the Power Automate Power BI trigger dropdown

To add more team milestones (e.g. 100, 200, 500), create a separate alert for each and wire each to its own Power Automate flow.

---

## Embedding in SharePoint

1. Publish the report to Power BI Service
2. On your SharePoint page → **Edit** → **Add web part** → **Power BI**
3. Paste the report embed link
4. Repeat for any additional report pages (e.g. a separate leaderboard page)

The two Power BI apps visible in SharePoint (`TacoTracker` and `TacoLeaderboard`) are just embedded report pages — no separate app configuration needed beyond the web part embed.

---

## Milestone Query (used in Power Automate)

This DAX runs inside the **Run a query against a dataset** step in Power Automate after every submission. It returns rows from `TacosByPerson` where the person is currently sitting at a milestone count.

```dax
EVALUATE
FILTER (
    'TacosByPerson',
    'TacosByPerson'[Who] IN {
        "Alex Johnson",
        "Morgan Lee",
        "Taylor Smith",
        "Jordan Rivera",
        "Casey Brown",
        "Riley Davis",
        "Quinn Martinez",
        "Avery Wilson",
        "Cameron Garcia",
        "Drew Thompson",
        "Skyler Nguyen"
    } &&
    'TacosByPerson'[TotalTacos] IN { 20, 40, 60, 66, 80, 100, 120 }
)
```

**To update:**
- Add/remove team members in the `IN { }` name list
- Add/remove milestone numbers in the `TotalTacos IN { }` list
- The `66` threshold is intentional — a mid-point reward between 60 and 80

**Accessed in Power Automate via:**
```
outputs('Run_a_query_against_a_dataset')?['body/firstTableRows']
```
