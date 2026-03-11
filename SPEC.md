# Talk Assigner — Product Spec

## Problem
Assigning sacrament meeting talks is manual, error-prone, and hard to track. Bishopric members lose track of who spoke recently, struggle to suggest topics, and have no system for tracking outreach status.

## Core Rules

### Speaking Frequency
| Age Group | Frequency |
|-----------|-----------|
| 12–17 (Youth) | Once per year |
| 18+ (Adults) | Every other year (once per 2 years) |

- No speakers under 12.
- The system should surface members who are **overdue** first, then those **eligible**, sorted by longest time since last talk.
- Members can be marked as **exempt** (e.g., health, calling conflicts, moved away) — deferred to v2.

### Scheduling Window
- Talks are scheduled **3 weeks out** from the current date.
- The system should always show the next 3 upcoming Sundays and their assignment status.

---

## Features

### 1. Member Roster
- Import/manage ward member list (name, age/birthdate, phone, email, household)
- Track last talk date per member
- Track total talks given
- Exempt/inactive flag with reason
- Notes field (e.g., "prefers shorter talks", "new member")

### 2. Talk Assignment Pipeline
Each assignment moves through a workflow:

```
SUGGESTED → CONTACTED → ACCEPTED / DECLINED → TOPIC_SHARED → REMINDER_SENT → COMPLETED
```

| Status | Description |
|--------|-------------|
| `SUGGESTED` | System or bishopric member suggested this person |
| `CONTACTED` | Outreach made (call/text/in-person), awaiting response |
| `ACCEPTED` | Member agreed to speak |
| `DECLINED` | Member declined — log reason, re-enter pool |
| `TOPIC_SHARED` | Topic/resources sent to the speaker |
| `REMINDER_SENT` | Automated or manual reminder sent X days before |
| `COMPLETED` | Talk was given |
| `NO_SHOW` | Member didn't show — note for future reference |

### 3. Auto-Suggest Topics

#### Come Follow Me Integration
- Store the Come Follow Me schedule (scripture blocks by week).
- For a given Sunday, look up the CFM reading for **that week** (the week following the talk date, since members study that week's material).
- Suggest a topic derived from the CFM scripture block (e.g., "Faith in adversity — Alma 32" for an Alma 32 week).

#### Conference Talk Matching
- Maintain a searchable index of recent General Conference talks (title, speaker, topic tags, URL).
- For each CFM week, auto-match 2–3 conference talks with overlapping themes/scriptures.
- Present these as suggested resources the speaker can use.

#### Topic Output Example
```
Sunday: April 5, 2026
CFM Reading: Doctrine & Covenants 30–36
Suggested Topics:
  - "Answering the Call to Serve" (D&C 31–33)
  - "Seeking Personal Revelation"
Recommended Conference Talks:
  - Elder Soares, "Called to the Work" (Oct 2025)
  - Sister Freeman, "Listening for His Voice" (Apr 2025)
```

### 4. Reminders
- Configurable reminder timing (default: **7 days before** the talk).
- Reminder includes: date, time, talk length expectation, topic, and any linked resources.
- Delivery method: SMS (Twilio) or email.
- Second optional reminder at 2 days before.
- Bishopric dashboard shows reminder status per assignment.

### 5. Dashboard Views

#### Upcoming Sundays (Next 3 Weeks)
| Sunday | Speaker 1 | Speaker 2 | Youth/Primary | Status |
|--------|-----------|-----------|---------------|--------|
| Mar 29 | John Smith | Jane Doe | — | All confirmed |
| Apr 5 | (open) | Maria Lopez | (open) | Needs assignments |
| Apr 12 | — | — | — | Not yet scheduled |

#### Member Queue
- Sorted by eligibility priority (most overdue first).
- Filters: age group, last spoke date, exempt status.
- Quick-assign button per member.

#### History Log
- Full history of all talks: who, when, topic, status.
- Searchable and exportable.

---

## Data Model

### `members`
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | |
| name | string | |
| birthdate | date | Used to calculate age group |
| phone | string | For SMS reminders |
| email | string | For email reminders |
| household_id | UUID | Group family members |
| exempt | boolean | |
| exempt_reason | string | |
| notes | string | |
| created_at | timestamp | |

### `assignments`
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | |
| member_id | UUID | FK → members |
| sunday_date | date | The Sunday they speak |
| status | enum | SUGGESTED → ... → COMPLETED |
| topic | string | Assigned/suggested topic |
| cfm_reference | string | e.g., "D&C 30–36" |
| conference_talks | JSON | Array of suggested talk links |
| contacted_at | timestamp | |
| accepted_at | timestamp | |
| topic_shared_at | timestamp | |
| reminder_sent_at | timestamp | |
| completed_at | timestamp | |
| declined_reason | string | |
| notes | string | |

### `sundays`
| Field | Type | Notes |
|-------|------|-------|
| date | date | PK |
| cfm_week | string | Come Follow Me reference |
| cfm_topic | string | Human-readable topic |
| suggested_topics | JSON | Auto-generated topic suggestions |
| conference_talks | JSON | Matched conference talks |
| notes | string | e.g., "Stake Conference — no talks" |

### `conference_talks`
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | |
| title | string | |
| speaker | string | |
| session | string | e.g., "Oct 2025 General" |
| url | string | churchofjesuschrist.org link |
| tags | string[] | Topic tags for matching |
| scriptures | string[] | Referenced scriptures |

---

## Tech Stack (Recommended)

| Layer | Choice | Rationale |
|-------|--------|-----------|
| Frontend | Next.js 15 + Tailwind + shadcn/ui | Matches your existing stack |
| Database | SQLite (via Drizzle ORM) or Postgres | Simple relational data; SQLite fine for single-ward use |
| Auth | Clerk or NextAuth | Bishopric-only access |
| SMS | Twilio | Reminders |
| Email | Resend | Reminders + topic sharing |
| Hosting | Vercel | Free tier sufficient |
| Cron | Vercel Cron or external | Daily check for due reminders |

---

## MVP Scope (v1)

1. **Member roster** — manual add/edit, age-based frequency rules
2. **Assignment pipeline** — create assignments, track status through workflow
3. **Eligibility engine** — auto-sort members by who's due, respecting age rules
4. **Sunday calendar** — view next 3 weeks, assign speakers
5. **Basic reminders** — email reminder 7 days before

### Deferred to v2
- Come Follow Me auto-topic suggestions
- Conference talk matching
- SMS via Twilio
- Household grouping logic (avoid scheduling spouses same Sunday)
- CSV import of ward directory
- Multi-ward support

---

## Open Questions

1. **How many speakers per Sunday?** Assumed 2 adults + optional youth/primary. Confirm.
2. **Talk length expectations?** Should the system store/communicate expected length (e.g., 10 min, 5 min for youth)?
3. **Who has access?** Bishopric only, or also executive secretary / ward clerk?
4. **Notification preferences?** Should members opt into SMS vs email?
5. **Come Follow Me data source?** Scrape from churchofjesuschrist.org or maintain manually? (API availability TBD.)
6. **Conference talk data source?** Same question — scrape or manual entry?
7. **Special Sundays?** How to handle fast Sunday, stake conference, general conference weekends, holidays?
