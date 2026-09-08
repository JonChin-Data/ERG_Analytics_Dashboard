# ERG Analytics Dashboard

A people-analytics project built to give a Director of People Operations
visibility into Employee Resource Group participation, spend, event
engagement, and membership trends, using data sources the organization
already had (Microsoft Teams, Microsoft Forms, Workday) rather than
adopting a new platform.

## The Problem

ERGs were coordinated informally through Teams channels with no
centralized reporting. Leadership had no consistent way to answer basic
questions: which ERGs exist, what they cost, how many staff participate,
how event engagement is trending, or how membership changes over time.

## What This Project Answers

1. What are the ERGs?
2. What is each ERG's spend, in dollars?
3. What percentage of staff belong to an ERG, and who doesn't?
4. How is event management tracked (cadence, turnout, cost)?
5. How many staff leave ERGs, and which ones, over time?

## Approach

I built this in two stages. The first is a fully functional Excel
workbook, star-schema data model, calculated fields, conditional
formatting, and a KPI dashboard built entirely from formulas. The second
stage migrates that same model into Power BI to add live cross-filtering
and slicers, capabilities Excel can't provide.

Since real Teams and Forms exports weren't available at project start,
the workbook runs on synthetic data shaped to match real export formats
exactly (Teams admin center membership exports, Microsoft Forms response
exports), so the structure and logic are provable before a single real
export exists.

## Repository Structure

erg-analytics-dashboard/
  workbook/   -> ERG_Analytics_Demo.xlsx (Excel version, complete)
  powerbi/    -> Director_ERG_Dashboard.pbix (Power BI version, in progress)
  docs/
    ERG_Design_Notes.md         -> requirements, assumptions, data model rationale
    ERG_PowerBI_DAX_Guide.md    -> DAX measures and visual build steps for the Power BI migration

## Tools Used

- **Excel**: Power Query-style structured tables, formula-driven KPIs,
  conditional formatting, native charts
- **Power BI**: star-schema relationship modeling, DAX measures,
  cross-filtered visuals
- **Python**: synthetic dataset generation, shaped to mirror real Teams
  and Microsoft Forms export formats

## Design Decisions Worth Reading

The full reasoning behind every assumption, including why membership
history required a snapshot-diffing approach and why event registrations
are keyed on an ID rather than an event name, is documented in
`docs/ERG_Design_Notes.md`.

## Status

- [x] Requirements scoped against stakeholder questions
- [x] Data model designed (star schema, 2 dimension tables, 5 fact tables)
- [x] Excel workbook built and validated
- [x] Power BI data model imported and relationships corrected
- [ ] DAX measures implemented
- [ ] Dashboard visuals built
- [ ] Power BI report published

This section will be updated as the Power BI build progresses.
