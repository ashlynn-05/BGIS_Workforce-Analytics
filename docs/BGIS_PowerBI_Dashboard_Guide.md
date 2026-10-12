# Building the BGIS Workforce Dashboard in Power BI

A step-by-step guide to rebuild the HR dashboard (Overview, Trades, Overtime, Non-billable Cost, Retention) as an interactive Power BI report, using the same merged data and figures from this project.

---

## 1. Import and connect the data

Get Data → Excel/CSV, and load each source file as its own table. Keep them separate — don't pre-merge in Excel, let Power BI's model do the joining so the visuals stay interactive.

Tables to load:
- **Astea Utilisation** — the row-level activity log (person_id, demand_id, hours, dates)
- **Financial Demands** — cost/revenue/billing fields (billable_code, RequestType, Regular Cost, Regular Price)
- **Employee Pay** — person_id, over_duration_1 (OT1), over_duration_2 (OT2), Approved Leave, Private Activities, Lost Time, travel_duration, holiday_duration, pay_basis, employment_status
- **Price Book Lines** — pbookh_id, activity_code, demand_type, actual_price, unit_cost
- **Customer Assignments** — pbookh_id, customer_id, customer (the sheet that maps customers to price books)
- **Technician Type Costing** — SA Type ID, Regular Cost/Price, 1st/2nd Overtime Cost %, Weekend/Holiday %

In Power Query, do your cleaning here (same steps your team already used to build master.pkl): fix data types, trim text, handle blanks in billable_code, flag missing Person Number matches. This keeps a documented, repeatable pipeline instead of one-off Python scripts.

## 2. Build the data model (star schema)

Set relationships in Model view:
- Financial Demands (fact) → Astea Utilisation on `demand_id`
- Financial Demands → Employee Pay on `person_id`
- Financial Demands → Price Book Lines on `activity_code` (and pbookh_id)
- Price Book Lines → Customer Assignments on `pbookh_id`
- Employee Pay → Technician Type Costing on `sa_type_id`

Add a **Date table** (Modeling → New Table → `DateTable = CALENDAR(MIN(...), MAX(...))`) and mark it as a date table. Relate it to your date fields so quarter/quarter-over-quarter visuals work natively instead of hardcoded arrays like the ones in the HTML prototype.

Add a **Trade** dimension table (ELEC, PLUMB, BUILD, HVAC, FIRE, OTHER) if trade_clean isn't already clean/deduplicated — this keeps slicers and legends consistent everywhere.

## 3. Core DAX measures

Create these in a dedicated "Measures" table (New Table with no columns, just a home for measures — keeps the model tidy).

**Utilisation & hours**
```
Billable Hours = CALCULATE(SUM(Astea[hours]), 'Financial Demands'[billable_code] = "S")
Total Paid Hours = SUM('Employee Pay'[regular_hours]) + SUM('Employee Pay'[over_duration_1])
    + SUM('Employee Pay'[over_duration_2]) + SUM('Employee Pay'[Approved Leave])
    + SUM('Employee Pay'[Private Activities]) + SUM('Employee Pay'[Lost Time])
Utilisation Rate = DIVIDE([Billable Hours], [Total Paid Hours])
```

**Overtime**
```
OT1 Hours = SUM('Employee Pay'[over_duration_1])
OT2 Hours = SUM('Employee Pay'[over_duration_2])
OT % of Regular Hours = DIVIDE([OT1 Hours] + [OT2 Hours], SUM('Employee Pay'[regular_hours]))
```

**Non-billable cost**
```
Non-billable Cost = CALCULATE(SUM('Financial Demands'[cost]), 'Financial Demands'[billable_code] = "N")
Non-billable % of Total = DIVIDE([Non-billable Cost], SUM('Financial Demands'[cost]))
Non-billable RM Share = DIVIDE(
    CALCULATE(SUM('Financial Demands'[cost]), 'Financial Demands'[billable_code]="N", 'Financial Demands'[RequestType]="RM"),
    [Non-billable Cost])
```

**Retention**
```
Terminations = CALCULATE(DISTINCTCOUNT('Employee Pay'[person_id]), 'Employee Pay'[employment_status]="Former")
Active Headcount = CALCULATE(DISTINCTCOUNT('Employee Pay'[person_id]), 'Employee Pay'[employment_status]="Active")
```

**Private Activities cost (the actionable slice of "non-working time")**
```
Private Activities Cost = SUMX('Employee Pay',
    'Employee Pay'[Private Activities] * RELATED('Technician Type Costing'[Regular Cost]))
```

## 4. Report pages — mirror the 5-tab structure

Create one Power BI **page** per tab (use Page navigation buttons or the built-in page tabs at the bottom).

**Page 1 — Overview**
- 5 Card visuals across the top: Active Workforce, Utilisation Rate, Non-billable Cost, Unbillable Overtime Cost, Q1 2026 Terminations
- Clustered column chart: Hires vs Departures by quarter
- Line chart: Non-billable Cost by quarter
- Table or list visual for the alert/insight bullets (static text box is fine, or a disconnected table if you want it to update)

**Page 2 — Trades**
- Table visual: Trade, Headcount, Demand/person, Non-billable Cost, Third-party count — sortable natively, no need to hand-build it
- Matrix visual (rows = Trade, columns = Quarter, values = OT %) with conditional formatting (background color scale) — this replaces the hand-rolled CSS heatmap for free
- Slicer: Trade — use this as a page-level or report-level filter so every visual responds

**Page 3 — Overtime**
- Line chart: OT % of total hours by quarter, with a trend line (Analytics pane → Trend line) instead of manually computing the forecast
- Donut chart: OT1 vs OT2 hours split
- Bar chart: Private Activities cost by trade
- Add a text box noting OT1/OT2 are paid at the same rate per Technician Type Costing — don't let the chart imply a cost difference

**Page 4 — Non-billable Cost**
- 3 cards: Total non-billable cost, % from top customer, % that's RM
- Bar chart (sorted descending): Top 10 customers by non-billable cost, colored by whether their price book has a callout fee defined (custom column: `HasCalloutFee = IF(...)`)
- Table: same billed-vs-non-billed count per activity code, filterable to one customer via a slicer — this is how you'd let the client explore the "same code billed 40%, non-billed 60%" finding interactively themselves
- Text box with the recommendation bullets

**Page 5 — Retention**
- Column chart: Terminations by quarter, with a reference line for the 2025 average (Analytics pane)
- Donut chart: Terminations by trade, Q1 2026
- Consider a decomposition tree visual (Power BI's built-in) if you want to let the client click through termination drivers themselves

## 5. Interactivity

- Add a **Trade slicer** and a **Date/Quarter slicer**, set to apply across all pages (Format slicer → Sync slicers, or use a slicer panel visible on every page via a bookmark-based nav pane).
- Use **drillthrough**: right-click the customer bar chart → set Page 4 as a drillthrough target filtered to that customer, so clicking "Defence NNSW" opens a detail page.
- **Tooltips**: build a small tooltip page (Page settings → tooltip) showing the quarterly trend when hovering a bar, similar to the sparklines in the HTML version.

## 6. Publish and share

File → Publish → publish to your Power BI workspace (needs a Microsoft/Deakin 365 login with Power BI licensing — check this is available before building, since it gates the "give the client a live link" pitch). Once published, you can share a read-only link or embed it, which is the actual answer to the client's Q6 about ongoing visibility — a real live Power BI dashboard beats a static export.

---

### Notes carried over from this project's data-quality findings
- ~32% of Person Number values don't match between Astea and the pay file — build a "Data Quality" measure/card so this caveat is visible in the report, not just the presentation.
- billable_code has 26.3% missing/blank rows — treat "Missing" as its own category in every billing-status visual, don't drop it.
- Non-billable cost drops sharply in the last 1–2 quarters of the extract — almost certainly a data-completeness artifact, not a real trend. Flag this with a text box or a "data through" note on the date slicer.
