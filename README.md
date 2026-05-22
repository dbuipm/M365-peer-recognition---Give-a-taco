# 🌮 Give a Taco — Peer Recognition System

A peer-recognition system built entirely on Microsoft 365. Team members give each other tacos to celebrate great work. The system tracks submissions, posts notifications to Teams, celebrates milestones, and displays a live leaderboard — no extra licenses or third-party tools required.

---

## Tech Stack

| Component | Tool |
|---|---|
| Submission Form | Microsoft Forms |
| Data Storage | SharePoint Lists |
| Automation | Power Automate |
| Dashboard | Power BI |
| Notifications | Microsoft Teams |
| Hosting | SharePoint |

---

## How It Works

1. A team member opens the **Give a TACO!** form and picks a colleague + writes a reason
2. **Power Automate** fires on submission — logs the taco to SharePoint, blocks self-tacos, and posts a Teams notification
3. The flow checks if the recipient just hit a **milestone** (20, 40, 60… tacos). If yes, it posts a celebration message and logs it so it never fires twice
4. The **Power BI dashboard** updates in real time — leaderboard, team total, and a feed of all compliments
5. When the team hits **300 tacos total**, a separate flow fires and announces a team reward

---

## SharePoint Lists

### TacoTracker
Every taco submission lands here.

| Column | Type | Source |
|---|---|---|
| Title | Single line | Form Response ID |
| Completion time | Date/Time | Form submission timestamp |
| Email | Single line | Responder's email |
| Name | Single line | Office 365 Display Name |
| Who do you want to give a taco? | Single line | Form Q1 |
| Why? | Multi-line | Form Q2 |

### TacoMilestoneLog
Prevents duplicate milestone notifications. One row per person per milestone.

| Column | Type |
|---|---|
| Person | Single line |
| Milestone | Number |

---

## Microsoft Form

**Title:** Give a TACO!

- **Q1:** Who do you want to give a taco? *(choice, single answer — lists all team members)*
- **Q2:** Why? *(long text)*

Set to org-only responses so identity is captured automatically. Embed on a SharePoint page with a taco-themed header image.

---

## Power Automate — TacoSubmission Flow

**Trigger:** When a new response is submitted (Microsoft Forms)

| Step | Action | Detail |
|---|---|---|
| 1 | Get response details | Pulls Q1 and Q2 from the form |
| 2 | Get user profile (V2) | Office 365 — gets submitter Display Name |
| 3 | Create item | Writes to `TacoTracker` |
| 4 | Refresh dataset | Refreshes Power BI `TacoLeaderboard` |
| 5 | Condition 2 | `Display Name == Who do you want to give a taco?` |
| ↳ False | Post Teams message | "🌮 TACO ALERT!" to group chat |
| 6 | Run query against dataset | DAX milestone check on `TacoLeaderboard` |
| 7 | Apply to each | Loops over milestone query results |
| 8 | Get items | Checks `TacoMilestoneLog` for existing record |
| 9 | Condition | Already logged? |
| ↳ False | Post Teams + Create item | Milestone celebration + log to `TacoMilestoneLog` |

**Taco Alert message:**
```
🌮 TACO ALERT! 🌮
Someone just got a delicious, well-deserved taco! 🎉
[Recipient] just received a taco from [Sender] for:
👉 [Why?]
```

**Milestone message:**
```
🎉 [Person] has just reached [N] tacos! 🏆 Reward unlocked!
```

---

## Power Automate — TacoTeamAlert Flow

**Trigger:** Power BI data-driven alert (Team Total Tacos ≥ 300)

Posts to Teams:
```
🎉 Team Tacos Milestone Reached!
The team just hit 300 tacos — reward unlocked! Team Lunch!!!
```

---

## Power BI Dashboard

**Dataset:** `TacoLeaderboard` — connected to `TacoTracker` SharePoint list

| Visual | Description |
|---|---|
| Leaderboard | Horizontal bar chart, tacos per person, sorted descending |
| Team Total Tacos | Card — sum of all tacos |
| The Taco Champ is | Card — person with the most tacos |
| The assistance and compliments | Scrollable table of all "Why?" submissions |

Set a **data-driven alert** on the Team Total card at threshold 300 to trigger the TacoTeamAlert flow.

---

## DAX — Milestone Query

Runs in Power Automate after each submission. Returns rows where a person is currently sitting at a milestone count.

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

`TacosByPerson` is a calculated table in Power BI that aggregates `TacoTracker` by recipient name and count.

Result is accessed in Power Automate via:
```
outputs('Run_a_query_against_a_dataset')?['body/firstTableRows']
```

---

## Setup

**Prerequisites:** Microsoft 365 (SharePoint, Forms, Power Automate, Teams) + Power BI Pro

1. **SharePoint** — Create `TacoTracker` and `TacoMilestoneLog` lists with columns above
2. **Microsoft Forms** — Create the form, add team members as choices, set org-only
3. **Power BI** — Connect to `TacoTracker`, build visuals, publish, set data-driven alert at 300
4. **Power Automate** — Build `TacoSubmission` flow (Forms trigger) and `TacoTeamAlert` flow (Power BI alert trigger)
5. **SharePoint page** — Embed the form and Power BI report web parts

---

## Customization

| What | Where |
|---|---|
| Team member list | Form Q1 choices + DAX `IN { }` block |
| Individual milestone thresholds | DAX `IN { 20, 40, ... }` |
| Team total milestone | Power BI data-driven alert threshold |
| Reward messages | Teams message text in each flow |
| Teams channel | "Post message" step in both flows |

---

## Tips

- **Refresh before query** — The dataset refresh (step 4) runs before the DAX milestone check so the query always sees the latest counts. Don't reorder these.
- **Exact name match** — The self-taco check and the DAX query both rely on Display Names matching form choices exactly. When adding someone, update both places.
- **Milestone dedup** — `TacoMilestoneLog` is the single source of truth. Safe to rerun or test flows — each milestone only ever notifies once per person.

---

## Repo Structure

```
give-a-taco/
├── README.md               ← Full project guide (you are here)
├── SYSTEM.md               ← System architecture overview
└── docs/
    ├── sharepoint.md       ← List setup & schema
    ├── power-automate.md   ← Both flows, step by step
    └── power-bi.md         ← Dashboard, DAX, and alerts
```
