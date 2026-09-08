# ERG Dashboard - Power BI Migration: DAX Measures & Visual Build Guide

## Why Power BI

The Excel version of this dashboard proved the data model and answered all
five of the director's original questions, but it's static: every KPI is a
worksheet formula, and there's no way for a viewer to click one ERG and see
every visual filter in response. Power BI's relational model and DAX
measure layer solve both problems, at the cost of migrating formula logic
into a proper semantic model. This document records that migration.

---

## Pre-Migration Cleanup

Four columns in the source workbook were Excel formula results, frozen at
the moment the file was last saved rather than live calculations:

- `tblERGs.Current_Members`
- `tblEvents.Actual_Registrations`
- `tblBudget.Variance_GBP`
- `tblBudget.Percent_Utilized`

I removed all four in Power Query during import. Carrying them over would
have meant the dashboard silently showing stale numbers the moment new
data loads, since Excel formulas don't survive the transition to a Power BI
data model as live calculations. Each is replaced below by a DAX measure
that recalculates automatically.

---

## Data Model

Two dimension tables (`tblEmployees`, `tblERGs`) and five fact tables, each
related back to one or both dimensions:

| Fact table | Relationship |
|---|---|
| tblTeamsExport | Worker_ID → tblEmployees; ERG_ID → tblERGs |
| tblERGChanges | Worker_ID → tblEmployees; ERG_ID → tblERGs |
| tblEvents | ERG_ID → tblERGs |
| tblBudget | ERG_ID → tblERGs |
| tblFormsReg | Event_ID → tblEvents |

No fact table relates directly to another fact table. This was a deliberate
correction: Power BI's relationship autodetect initially linked fact tables
to each other wherever a column name matched (e.g. `ERG_ID` appearing in
both `tblBudget` and `tblEvents`), which created redundant and occasionally
inactive relationship paths. Enforcing a clean star schema, everything
routes through the two dimension tables, kept the filter logic predictable.

---

## Measures

### Helper Measure

```dax
Latest Snapshot Date = MAX(tblTeamsExport[Snapshot_Date])
```

Every measure that needs "current" membership references this instead of a
hardcoded date, so the dashboard stays accurate as new monthly snapshots
are loaded.

### Participation

```dax
Active Headcount =
CALCULATE(COUNTROWS(tblEmployees), tblEmployees[Worker_Status] = "Active")

Staff in >=1 ERG =
CALCULATE(
    DISTINCTCOUNT(tblTeamsExport[Worker_ID]),
    tblTeamsExport[Snapshot_Date] = [Latest Snapshot Date]
)

Staff Unaffiliated = [Active Headcount] - [Staff in >=1 ERG]

Participation Rate = DIVIDE([Staff in >=1 ERG], [Active Headcount], 0)

Current Members =
CALCULATE(
    DISTINCTCOUNT(tblTeamsExport[Worker_ID]),
    tblTeamsExport[Snapshot_Date] = [Latest Snapshot Date]
)
```

`Current Members` and `Staff in >=1 ERG` share the same definition but
behave differently depending on filter context: placed against `ERG_Name`
on a chart axis, `Current Members` returns membership for that specific
ERG. This is the direct replacement for the frozen `Current_Members`
column removed during cleanup.

### Spend

```dax
Total Budgeted = SUM(tblBudget[Budgeted_GBP])

Total Actual Spend = SUM(tblBudget[Actual_Spend_GBP])

Budget Variance = [Total Budgeted] - [Total Actual Spend]

Budget Utilized % = DIVIDE([Total Actual Spend], [Total Budgeted], 0)

ERGs Over Budget =
CALCULATE(
    COUNTROWS(tblBudget),
    FILTER(tblBudget, tblBudget[Actual_Spend_GBP] > tblBudget[Budgeted_GBP])
)
```

### Event Management

```dax
Events Count = COUNTROWS(tblEvents)

Total Registrations = COUNTROWS(tblFormsReg)

Avg Registrations per Event = DIVIDE([Total Registrations], [Events Count], 0)

Total Event Cost = SUM(tblEvents[Cost_GBP])
```

`Total Registrations` filters correctly by ERG even though
`tblFormsReg` has no direct relationship to `tblERGs` - the filter
context propagates transitively through `tblERGs → tblEvents →
tblFormsReg`, since each relationship in that chain is single-direction
one-to-many.

### Membership Movement

```dax
Joins =
CALCULATE(COUNTROWS(tblERGChanges), tblERGChanges[Change_Type] = "Joined")

Leaves =
CALCULATE(COUNTROWS(tblERGChanges), tblERGChanges[Change_Type] = "Left")

Net Change = [Joins] - [Leaves]

ERGs Tracked = DISTINCTCOUNT(tblERGs[ERG_ID])
```

These measures intentionally reflect all loaded data rather than a fixed
trailing window, unlike the Excel version, which hardcoded a six-month
range. This removes a maintenance burden: the range no longer needs manual
updating as new months of data are added. A date-range slicer can be added
if a rolling window view is preferred later.

---

## Visual Build Guide

### ERG Directory
Table visual: `ERG_Name`, `Sponsor`, `Category`, `Status` from `tblERGs`.

### Participation
- Card visuals: `Active Headcount`, `Staff in >=1 ERG`, `Staff
  Unaffiliated`, `Participation Rate`
- Stacked bar: `Staff in >=1 ERG` and `Staff Unaffiliated` as two measures,
  in place of a true donut chart, since donut visuals require a category
  column rather than two separate measures
- Horizontal bar chart: axis `tblERGs[ERG_Name]`, value `Current Members`

### Spend
- Card visuals: `Total Budgeted`, `Total Actual Spend`, `Budget Utilized
  %`, `ERGs Over Budget`
- Clustered column chart: axis `tblERGs[ERG_Name]`, values `Total
  Budgeted` and `Total Actual Spend`

### Event Management
- Card visuals: `Events Count`, `Total Registrations`, `Avg Registrations
  per Event`, `Total Event Cost`
- Pie chart: legend `tblEvents[Event_Type]`, value `Total Registrations`

### Membership Movement
- Card visuals: `Joins`, `Leaves`, `Net Change`, `ERGs Tracked`
- Line chart: axis `tblERGChanges[Snapshot_Date]` (date hierarchy disabled
  so it plots by month rather than drilling into year/quarter/day), values
  `Joins` and `Leaves`

### Interactivity

- ERG slicer (`tblERGs[ERG_Name]`) - filters every visual on the page at
  once, since all fact tables trace back to this dimension.
- Snapshot date slicer (`tblTeamsExport[Snapshot_Date]` or
  `tblERGChanges[Snapshot_Date]`) - lets a viewer inspect a specific month
  rather than only the latest snapshot.

This cross-filtering behavior is the primary capability gained by moving
off Excel: clicking one ERG updates every card, chart, and table on the
page simultaneously.

---

## Validation

The Power BI totals were cross-checked against the Excel dashboard's
published figures to confirm the migration preserved the underlying logic:
7 staff in an ERG against 13 unaffiliated, £13,500 budgeted against
roughly £8,460 actual spend, 17 events generating 109 registrations, and
10 joins against 14 leaves over the trailing period. Discrepancies at this
stage were consistently traceable to relationship or filter-direction
issues in the model rather than errors in the DAX itself.
