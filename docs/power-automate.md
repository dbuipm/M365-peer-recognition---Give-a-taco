# Power Automate — Give a Taco

Two flows. One handles every taco submission. The other handles the team total milestone.

---

## Flow 1: TacoSubmission

**Trigger:** Microsoft Forms — When a new response is submitted → select your "Give a TACO!" form

### Steps

**1. Get response details**
- Connector: Microsoft Forms
- Form Id: your form
- Response Id: `List of response notifications Response Id` (from trigger)

**2. Get user profile (V2)**
- Connector: Office 365 Users
- User (UPN): `Responders' Email` (from step 1)
- This gives you the submitter's `Display Name` for the self-taco check and list logging

**3. Create item — TacoTracker**
- Connector: SharePoint
- Site: your SharePoint site
- List: `TacoTracker`
- Map fields as shown in `docs/sharepoint.md`

**4. Refresh a dataset**
- Connector: Power BI
- Workspace: your workspace
- Dataset: `TacoLeaderboard`
- Must run before the DAX query so milestone counts are current

**5. Condition 2 — Self-Taco Block**
- Expression: `Display Name` **is equal to** `Who do you want to give a taco?`
- **True branch:** no actions (submission silently dropped)
- **False branch:** continue below

> False branch — Post message in a chat or channel
> - Connector: Microsoft Teams
> - Post as: Flow bot
> - Post in: Group chat
> - Group chat: `Taco Alert! 🌮`
> - Message:
> ```
> 🌮 TACO ALERT! 🌮
> Someone just got a delicious, well-deserved taco! 🎉
> [Who do you want to give a taco?] just received a taco from [Display Name] for:
> 👉 [Why?]
> ```

**6. Run a query against a dataset**
- Connector: Power BI
- Dataset: `TacoLeaderboard`
- Query:
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
Replace names with your team. Replace milestones with your reward thresholds.

**7. Apply to each**
- Input: `outputs('Run_a_query_against_a_dataset')?['body/firstTableRows']`

> Inside the loop:

**7a. Get items — TacoMilestoneLog**
- Connector: SharePoint
- List: `TacoMilestoneLog`
- Filter Query: `Person eq '[TacosByPerson[Who]]' and Milestone eq [TacosByPerson[TotalTacos]]`

**7b. Condition — Already logged?**
- Check if `value` from Get items is empty
- **True (already logged):** no actions
- **False (new milestone):**

> Post message in Teams:
> ```
> 🎉 [TacosByPerson[Who]] has just reached [TacosByPerson[TotalTacos]] tacos! 🏆 Reward unlocked!
> ```
>
> Create item in `TacoMilestoneLog`:
> - Person: `TacosByPerson[Who]`
> - Milestone: `TacosByPerson[TotalTacos]`

### Required Connectors

| Connector | Permission |
|---|---|
| Microsoft Forms | Access to the form |
| Office 365 Users | Read user profiles |
| SharePoint | Read/Write to both lists |
| Power BI | Refresh dataset, Run queries |
| Microsoft Teams | Post as Flow bot |

---

## Flow 2: TacoTeamAlert

**Trigger:** Power BI — When a data driven alert is triggered → select your team total alert

### Steps

**1. Post message in a chat or channel**
- Connector: Microsoft Teams
- Post as: Flow bot
- Post in: Group chat
- Group chat: `Taco Alert! 🌮`
- Message:
```
🎉 Team Tacos Milestone Reached!
The team just hit 300 tacos — reward unlocked! Team Lunch!!!
```

### Setting up the Power BI alert

1. Open your Power BI report in the Power BI Service
2. Click the **Team Total Tacos** card → bell icon → **Manage alerts**
3. Create alert: condition **Above**, threshold **300**
4. Frequency: at most once per day
5. This alert now appears as a trigger option in Power Automate
