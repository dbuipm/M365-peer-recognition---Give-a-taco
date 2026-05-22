# System Architecture — Give a Taco

## Overview

Give a Taco is a no-code peer-recognition system running entirely on Microsoft 365. There are no servers, no custom applications, and no third-party services. Everything is wired together using native M365 connectors.

---

## Components

```
┌──────────────────────────────────────────────────────────┐
│                    Microsoft Forms                        │
│                                                          │
│  "Give a TACO!" form                                     │
│  Q1: Who? (radio choice)   Q2: Why? (free text)          │
└────────────────────────┬─────────────────────────────────┘
                         │ new response
                         ▼
┌──────────────────────────────────────────────────────────┐
│              Power Automate — TacoSubmission              │
│                                                          │
│  1. Get form response details                            │
│  2. Get submitter profile (Office 365)                   │
│  3. Write to SharePoint → TacoTracker                    │
│  4. Refresh Power BI dataset                             │
│  5. Self-taco check ──── submitter == recipient?         │
│       YES → stop                                         │
│       NO  → Post to Teams "🌮 TACO ALERT!"              │
│  6. Run DAX milestone query on Power BI                  │
│  7. For each milestone hit:                              │
│       Already in TacoMilestoneLog? → skip               │
│       Not logged yet?                                    │
│         → Post milestone message to Teams                │
│         → Write to TacoMilestoneLog                      │
└───────┬──────────────────────────┬───────────────────────┘
        │                          │
        ▼                          ▼
┌───────────────┐       ┌──────────────────────┐
│  SharePoint   │       │   Microsoft Teams     │
│               │       │                      │
│ TacoTracker   │       │  "Taco Alert! 🌮"    │
│ (all tacos)   │       │  group chat          │
│               │       │                      │
│ TacoMilestone │       │  • Per-taco alert    │
│ Log (dedup)   │       │  • Milestone reached │
└───────┬───────┘       │  • Team total reward │
        │               └──────────────────────┘
        ▼
┌──────────────────────────────────────────────────────────┐
│                    Power BI Dashboard                     │
│                                                          │
│  Connected to: TacoTracker (SharePoint)                  │
│                                                          │
│  • Leaderboard bar chart                                 │
│  • Team Total Tacos card  ←── data-driven alert at 300   │
│  • Taco Champ card                                       │
│  • Compliments feed ("Why?" submissions)                 │
└────────────────────────┬─────────────────────────────────┘
                         │ alert fires at 300
                         ▼
┌──────────────────────────────────────────────────────────┐
│           Power Automate — TacoTeamAlert                  │
│                                                          │
│  Posts to Teams: "Team hit 300 tacos — Team Lunch!!!"    │
└──────────────────────────────────────────────────────────┘
```

---

## Data Flow

| Event | What happens |
|---|---|
| Form submitted | PA reads form answers + fetches submitter's Office 365 profile |
| Data written | One row added to `TacoTracker` with submitter name, recipient, reason, timestamp |
| Self-taco | Silently blocked — no Teams message, data still logged |
| Valid taco | Teams "Taco Alert!" posted to group chat |
| Dataset refreshed | Power BI reflects new taco count before milestone check runs |
| Milestone hit (first time) | Teams milestone message posted + row written to `TacoMilestoneLog` |
| Milestone hit (repeat) | `TacoMilestoneLog` check finds existing row → skipped |
| Team total hits 300 | Power BI alert fires → TacoTeamAlert flow → Teams announcement |

---

## SharePoint Lists

| List | Purpose |
|---|---|
| `TacoTracker` | Primary store — one row per taco given |
| `TacoMilestoneLog` | Deduplication guard — one row per person per milestone |

Power BI reads from `TacoTracker` only. `TacoMilestoneLog` is written and read exclusively by Power Automate.

---

## Identity & Trust

- The form is **org-restricted** — only authenticated M365 users can submit
- Submitter identity comes from **Office 365 `Get User Profile (V2)`** using the form's captured email — not from a self-reported field
- The self-taco check compares the Office 365 Display Name (authoritative) against the form Q1 answer (user-selected) — both must match exactly to block

---

## Milestone Logic (Two-Layer Dedup)

**Layer 1 — Detect (DAX query)**
Power BI returns rows where a person is currently at a milestone count. This runs fresh on every submission because the dataset is refreshed first.

**Layer 2 — Guard (TacoMilestoneLog)**
Before posting any celebration, Power Automate checks if `Person + Milestone` already exists in `TacoMilestoneLog`. If it does, the message is skipped. If not, the message is sent and the record is created.

This means milestone notifications are safe to re-trigger — no duplicate spam regardless of retries or flow reruns.
