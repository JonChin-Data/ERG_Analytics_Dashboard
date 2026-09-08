# ERG Analytics Dashboard - Design Rationale & Data Model

## Project Context

Employee Resource Groups at this organization are currently coordinated
through Microsoft Teams and Microsoft Forms, with no centralized reporting
layer. The Director of People Operations, who oversees both Learning &
Development and Talent Acquisition, needed a way to see ERG participation,
spend, event engagement, and membership trends at a glance, similar to
purpose-built platforms like Chezie, without adopting new software.

This document walks through how I scoped the requirements, the assumptions
I made to keep the project moving without a full technical discovery
phase, and the resulting data model. It's meant to be read both as a build
guide and as a record of the reasoning behind each design decision.

---

## Requirements

I translated the director's questions into five concrete asks and assessed
each for feasibility against the tools already in use:

| # | Question | Likely Source | Feasibility |
|---|---|---|---|
| 1 | What are the ERGs? | Teams group directory | Straightforward, if each ERG maps to one group |
| 2 | What is each ERG's spend, in dollars? | No existing source | **Gap** - needs a data source |
| 3 | What percentage of staff belong to an ERG? | Workday headcount + Teams membership | Straightforward, once membership is exportable |
| 4 | How should event management be tracked? | Teams/Forms, loosely | **Underspecified** - needed clarification |
| 5 | How many staff leave ERGs, and which ones, over time? | Teams membership | **Gap** - current tools show a snapshot, not history |

Two of the five questions were answerable directly from existing tools. Two
surfaced real data gaps worth flagging before committing to a delivery
timeline. One needed further scoping before it could be built.

---

## Design Assumptions

Rather than pausing the project for a full IT discovery cycle, I made five
assumptions to keep the build moving, each chosen specifically because it
would be cheap to correct: it's easier for a stakeholder to react to "this
isn't quite right" against a working prototype than to answer a lengthy
requirements questionnaire before any output exists.

1. **ERG structure** - I modeled each ERG as a Teams channel backed by an
   M365 Group. Membership becomes exportable via the Teams admin center or
   the Microsoft Graph API (`GET /teams/{id}/channels/{id}/members`, or the
   team-level members endpoint if an ERG is structured as its own Team
   rather than a channel).

2. **Membership history** - Teams only exposes current-state membership;
   there's no native join/leave log. I addressed this by treating each
   monthly membership export as a dated snapshot and diffing consecutive
   snapshots to reconstruct joins and leaves. This is the same
   snapshot-diffing pattern I used for tracking role changes in a separate
   workforce planning project, adapted here to ERG membership instead of
   employment status.

3. **Event registration** - Events are registered through Microsoft Forms.
   Since event titles can and do repeat (a recurring Speaker Series, for
   example), the raw Forms export alone can't distinguish one occurrence
   from another. I addressed this by requiring an Event_ID to be tagged
   onto each batch of Forms responses at compile time, matching the ID
   already assigned in the events reference table.

4. **Budget and spend** - No system of record currently exists for ERG
   budgets. I built this as a manually maintained table, populated by ERG
   leads or Finance, with the expectation that it gets replaced by a real
   source (a general ledger export, for instance) if one is identified
   later.

5. **Event management scope** - Since the original request didn't specify
   what "tracking event management" should mean, I chose to track cadence
   (events per ERG per period), registration volume, attendance where
   captured, and cost, covering the four most likely interpretations rather
   than guessing at a single one.

---

## Data Model

The model follows a standard star-schema approach: two dimension tables
(ERGs, Employees) and five fact tables that reference them.

- **ERGs** - ERG name, associated Teams team/channel, sponsor, founding
  date, category, and status.
- **Employees** - sourced from Workday: worker ID, department, hire and
  termination dates, employment status. This table is what makes
  "percentage of staff in an ERG" calculable.
- **ERG_Membership_Snapshot** - one row per employee per ERG per snapshot
  date, sourced from a monthly Teams membership export. Snapshots are
  appended, never overwritten, since the trend analysis depends on having
  history.
- **ERG_Membership_Changes** - derived, not imported. Diffing consecutive
  snapshot dates per employee/ERG produces Joined/Left records with an
  effective date.
- **ERG_Budget** - ERG, fiscal year, budgeted amount, actual spend.
  Manually maintained pending a confirmed system source.
- **ERG_Events** - event name, ERG, date, type, planned attendance, cost.
- **ERG_Event_Registrations** - one row per Forms response, keyed on
  Event_ID rather than event name to handle recurring event titles
  correctly. Rolls up into per-event and per-ERG registration counts.

---

## Question-to-Table Mapping

| Question | Answered by |
|---|---|
| What are the ERGs? | ERGs table, directory view |
| Spend per ERG | ERG_Budget |
| Participation rate | Employees joined against the latest ERG_Membership_Snapshot; unmatched employees are unaffiliated |
| Event management | ERG_Events + ERG_Event_Registrations |
| Membership movement over time | ERG_Membership_Changes, derived from snapshot diffing |

---

## Build Sequence

The project was staged to deliver value incrementally rather than as a
single large release:

1. ERGs directory + Employees import - answers "what are the ERGs"
2. Membership snapshot + employee join - answers participation rate
3. Membership change diffing - answers movement over time
4. Events + registrations, keyed on Event_ID - answers event management
5. Budget table - answers spend
6. Consolidated dashboard tying all five together

Since real Teams and Forms exports weren't yet available at the time of
the initial build, the workbook runs on synthetic data shaped to match
real export formats exactly, so the structure and logic are provable
before a single real export exists.

---

## Status

The Excel version is complete and validated against the assumptions above.
I'm currently migrating the model to Power BI to support live filtering
and cross-highlighting across visuals, an interactivity layer Excel can't
provide. See `ERG_PowerBI_DAX_Guide.md` for the measure definitions and
visual build steps.
