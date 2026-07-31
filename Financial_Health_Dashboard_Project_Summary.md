# Financial Health Dashboard — Project Summary

## Task Brief

**Objective:** Develop a dashboard analyzing an organization's financial status with real-time insights, tailored for SMEs.

**Key requirements:**
- Visualize income statements, balance sheets, and cash flows.
- Analyze profitability trends over time.
- Provide forecasting for budgeting and financial planning.

**Deliverable:** An interactive Power BI report with dynamic visualizations and actionable insights.

This document walks through, step by step, how each requirement was met — from raw dataset to a three-tab interactive report.

---

## 1. Dataset Overview

The source file (`financial_dataset.csv`) contains 2,000 rows and 30 columns: 50 companies (`Company_ID`), tracked quarterly from 2015 to 2024 (`Year`, `Quarter`). No missing values.

Beyond the core financial statement fields, the file also includes alt-data and ML-labelling columns (sentiment scores, social buzz, macro indicators, fraud/anomaly flags) that belong to a different type of task and are out of scope for a financial health dashboard.

A quick statistical check showed that `Inflation_Rate`, `Interest_Rate`, `Exchange_Rate`, and `Global_Economic_Score` vary randomly per company even within the same year and quarter (standard deviation of about 2 within a single quarter). This confirmed they are not genuine shared macro-economic indicators, just per-row noise — and were excluded on that basis.

---

## 2. Data Cleaning (Power Query)

### 2.1 Columns removed (13 total)

| Group | Columns | Reason |
|---|---|---|
| Market data | `Stock_Price`, `Volume_Traded` | Public market data, irrelevant to a private SME |
| Alt-data / sentiment | `News_Sentiment_Score`, `Social_Media_Buzz`, `Sector_Trend_Index` | Out of scope for a financial dashboard |
| Noisy pseudo-macro | `Global_Economic_Score`, `Inflation_Rate`, `Exchange_Rate`, `Interest_Rate` | Confirmed to be per-row noise, not real macro data |
| Fraud/anomaly labels | `Audit_Flag`, `Fraud_Flag`, `Market_Shock_Flag`, `Policy_Change_Flag`, `Target_Anomaly_Class` | Belong to a different (fraud detection) task |

### 2.2 Columns kept

`Year`, `Quarter`, `Company_ID`, `Revenue`, `Expenses`, `Operating_Income`, `Net_Income`, `Assets`, `Liabilities`, `Equity`, `Cash_Flow`, `EPS`, `ROE`, `ROA`, `Debt_to_Equity`, and `Target_Revenue_Next_Qtr` (kept specifically to power the forecasting tab).

### 2.3 Formatting rules applied

- `Year`, `Quarter`: Whole Number.
- `Revenue`, `Expenses`, `Operating_Income`, `Net_Income`, `Assets`, `Liabilities`, `Equity`, `Cash_Flow`: Fixed Decimal Number, Currency format.
- `ROE`, `ROA`: already expressed as percentage points (verified `ROE = Net_Income / Equity × 100` exactly) — formatted as Decimal Number with custom format `0.00"%"` rather than Power BI's native Percentage type, to avoid a double multiplication by 100.
- `Debt_to_Equity`: Decimal Number, 2 decimals, optional `x` suffix (e.g. `1.02x`).

---

## 3. Data Model — Star Schema

One fact table and two dimension tables — the minimal structure that still forms a proper star schema for this dataset.

**FactFinancials** ← linked to → **DimDate** and **DimCompany**

### 3.1 Build steps

1. Load the CSV, remove the 13 columns above, set correct data types.
2. Add a custom column `DateKey = #date(Year, (Quarter-1)*3+1, 1)` — turns Year + Quarter into a real date (first day of the quarter). This single step is what enables time intelligence and Power BI's native forecasting feature later on.
3. Reference the query twice to build the two dimensions:
   - **DimDate**: keep only `DateKey`, remove duplicates, add:
     - `Quarter Label = Text.From(Date.Year([DateKey])) & " Q" & Text.From(Date.QuarterOfYear([DateKey]))` → e.g. `2015 Q1`
     - `YearQuarterSort = Date.Year([DateKey]) * 10 + Date.QuarterOfYear([DateKey])` → used as the sort-by column so the axis always orders chronologically regardless of the label text.
     - Marked as the **Date Table** in Model view.
   - **DimCompany**: keep only `Company_ID`, remove duplicates, add a cleaner `CompanyName` label.
4. In the original query, drop `Year` and `Quarter` (now redundant), rename the query to `FactFinancials`.
5. Relationships: `DimDate[DateKey]` (1) → `FactFinancials[DateKey]` (many); `DimCompany[CompanyKey]` (1) → `FactFinancials[CompanyKey]` (many). Single direction on both.

### 3.2 Modelling note

`Assets`, `Liabilities`, and `Equity` are balance-sheet **snapshots**, not additive flows — summing them across quarters is not meaningful. All "latest period" balance-sheet measures use `LASTNONBLANK` rather than a plain `SUM`.

---

## 4. DAX Measures

### 4.1 Core measures

```DAX
Total Revenue = SUM(FactFinancials[Revenue])
Net Income = SUM(FactFinancials[Net_Income])
Net Profit Margin % = DIVIDE([Net Income], [Total Revenue])
Operating Margin % = DIVIDE(SUM(FactFinancials[Operating_Income]), [Total Revenue])

Latest Assets =
CALCULATE(SUM(FactFinancials[Assets]), LASTNONBLANK(DimDate[DateKey], [Total Revenue]))

Latest Equity =
CALCULATE(SUM(FactFinancials[Equity]), LASTNONBLANK(DimDate[DateKey], [Total Revenue]))
```

### 4.2 Cash flow measure

```DAX
Net Cash Flow = SUM(FactFinancials[Cash_Flow])
```
A plain `SUM` is valid here (unlike Assets/Liabilities/Equity) because cash flow is an additive flow, not a snapshot.

```DAX
Net Cash Flow (Cumulative) =
CALCULATE(
    [Net Cash Flow],
    FILTER(
        ALLSELECTED(DimDate[DateKey]),
        DimDate[DateKey] <= MAX(DimDate[DateKey])
    )
)
```

### 4.3 Forecasting & budgeting measures

```DAX
Forecast Revenue (3Q Avg) =
AVERAGEX(
    DATESINPERIOD(DimDate[DateKey], LASTDATE(DimDate[DateKey]), -3, MONTH),
    [Total Revenue]
)

Actual Revenue (Next Qtr) =
CALCULATE(
    SUM(FactFinancials[Target_Revenue_Next_Qtr]),
    LASTNONBLANK(DimDate[DateKey], [Total Revenue])
)

Forecast Variance % (Latest) =
CALCULATE(
    DIVIDE(
        [Actual Revenue (Next Qtr)] - [Forecast Revenue (3Q Avg)],
        [Actual Revenue (Next Qtr)]
    ),
    LASTNONBLANK(DimDate[DateKey], [Total Revenue])
)

Forecast Variance % =
DIVIDE(
    SUM(FactFinancials[Target_Revenue_Next_Qtr]) - [Forecast Revenue (3Q Avg)],
    SUM(FactFinancials[Target_Revenue_Next_Qtr])
)

Forecast Variance $ =
[Actual Revenue (Next Qtr)] - [Forecast Revenue (3Q Avg)]
```
Two variance measures are kept deliberately: the `(Latest)` version for the KPI card (single most-recent value), and the plain version for the variance bar chart (one value per quarter).

`Target_Revenue_Next_Qtr` was genuinely useful here: it enables a real forecast-vs-actual comparison for the next quarter without inventing a target — exactly what an SME budgeting tool needs.

### 4.4 Supporting measures for visuals

```DAX
Benchmark Margin = 0.10   -- formatted as %

Gauge Max =
MAXX(
    ALL(DimDate[DateKey]),
    CALCULATE([Net Profit Margin %])
) * 1.2

Gauge Min = 0

Waterfall Amount =
SWITCH(
    SELECTEDVALUE(WaterfallSteps[Step]),
    "Revenue", [Total Revenue],
    "Expenses", -SUM(FactFinancials[Expenses]),
    "Net Income", [Net Income]
)
```
`Waterfall Amount` relies on a small disconnected table `WaterfallSteps` (`Step`, `Order`) created manually via *Enter Data*, since Power BI has no native "step" category for waterfall charts.

---

## 5. Report Layout — Three Tabs

Each tab maps directly to one of the three brief requirements.

### 5.1 Tab 1 — Financial Statements Overview
*(requirement: visualize income statements, balance sheets, and cash flows)*

| Visual | Type | Title | Purpose |
|---|---|---|---|
| Company / Year / Quarter filters | Slicer | — | Drill into one company or period without touching the model |
| Revenue, Expenses, Net Income, Cash Flow | KPI cards | *Current quarter* | The four numbers an SME owner checks first |
| Revenue → Net Income | Waterfall chart | **Revenue to Net Income** | Shows visually where money leaks between revenue and profit |
| Assets vs. Liabilities + Equity | Stacked column + line combo | **Assets vs. Liabilities + Equity** | Confirms the accounting equation visually, shows funding mix over time |
| Cash flow over time | Area chart with zero-line | **Cash Flow Over Time** | Zero baseline instantly flags negative-cash-flow quarters |
| Quarterly P&L detail | Table with data bars | **Quarterly P&L Detail** | Exact numbers for reconciliation or export |

Tooltip ideas: report-page tooltip on the waterfall showing "% of revenue" per bar; Debt-to-Equity as an extra tooltip field on the balance sheet chart.

### 5.2 Tab 2 — Profitability & Performance Trends
*(requirement: analyze profitability trends over time)*

| Visual | Type | Title | Purpose |
|---|---|---|---|
| Net margin, Operating margin, ROE, ROA | KPI cards | *Profitability at a glance* | — |
| ROE vs. ROA, bubble = revenue, colour = company | Bubble chart | **ROE vs. ROA by Company** | Shows which companies generate the most return per dollar of capital |
| Margin vs. benchmark | Gauge | **Net Margin vs. Benchmark** | Instant check against a 10% industry target |
| Top/bottom companies by net income | Ranked bar chart | **Top & Bottom Companies by Net Income** | Surfaces winners and problem units immediately |
| ROE by company × year | Matrix heatmap | **ROE by Company and Year** | Pattern recognition across 50 companies and 10 years at once |

Tooltip ideas: exact margin number and gap to benchmark on the gauge; report-page tooltip with ROE, ROA, and Net Income together on heatmap cells.

### 5.3 Tab 3 — Forecasting & Budget Planning
*(requirement: provide forecasting for budgeting and financial planning)*

| Visual | Type | Title | Purpose |
|---|---|---|---|
| Forecast, Actual, Variance | KPI cards | *Next-quarter revenue outlook* | The headline planning number and whether results track above/below it |
| Revenue line with native forecast | Line chart + built-in forecasting | **Revenue Forecast** | Right-click → Analytics pane → Forecast; draws the projection and confidence band with no extra DAX |
| Actual revenue vs. `Target_Revenue_Next_Qtr` | Combo chart (column + line) | **Actual vs. Target Revenue** | Classic budget-vs-actual view |
| Forecast variance % by quarter | Variance bar chart (green/red) | **Forecast Variance % by Quarter** | Shows whether the forecasting method is getting more or less accurate over time |
| Quarter / actual / forecast / variance $ / variance % | Table | **Budget Summary** | Exportable budget sheet for board packs or the accountant |

Tooltip ideas: that quarter's net profit margin for context on the combo chart; the 3-quarter average used to compute the forecast on the variance bars.

---

## 6. Tooltips (applies to all three tabs)

- **Field-level tooltips**: drag extra measures into the "Tooltips" bucket of any visual — added to the default hover box instantly.
- **Report-page tooltips** (used for the ideas above): create a small new page, set *Page information → Tooltip* to On and size to Tooltip (320×240 px), add 1–2 focused visuals, then in the source visual's *Format → Tooltips*, set Type to "Report page" and select that page.

---

## 7. Key Implementation Notes & Fixes Made Along the Way

- **Quarter Label ordering**: initial `"Q" & Quarter & " " & Year"` format sorted alphabetically and broke chronological order. Fixed by rebuilding the label from `DateKey` (`Date.Year([DateKey])` / `Date.QuarterOfYear([DateKey])`) as `"2015 Q1"`, combined with `YearQuarterSort` set as the *Sort by column* on `Quarter Label` in Power BI Desktop.
- **Percentage double-scaling risk**: `ROE`/`ROA` are already percentage points, while ratio-based DAX measures (`Net Profit Margin %`, `Forecast Variance %`) are decimal ratios — conditional-formatting rules and gauge min/max/target values must stay consistent with whichever scale a given measure uses.
- **KPI card vs. trend measures**: variance measures were split into a `(Latest)` version (`LASTNONBLANK`, for cards) and a per-period version (plain `CALCULATE`, for the variance bar chart) to avoid every quarter showing the same value.
- **Native forecasting prerequisite**: the *Forecast* option in the Analytics pane only works with a continuous date axis, which is exactly why `DateKey` was built as a true Date type and `DimDate` marked as the Date Table.

---

## 8. Suggested Time Budget (2 days / ~12–15 hrs)

| Task | Time |
|---|---|
| Power Query cleanup + star schema + relationships | 2–3 hrs |
| Core DAX measures (margins, ratios, forecast, variance) | 2 hrs |
| Tab 1 (statements) | 2–3 hrs |
| Tab 2 (profitability) | 2–3 hrs |
| Tab 3 (forecasting) + native forecast setup | 2 hrs |
| Tooltips + formatting polish + review | 2 hrs |

The native forecast feature and the reuse of `Target_Revenue_Next_Qtr` saved the effort that would otherwise have gone into building a custom forecasting model from scratch.

---

## 9. Requirement Coverage Summary

| Brief requirement | Delivered via |
|---|---|
| Visualize income statements, balance sheets, and cash flows | Tab 1 — waterfall, stacked column, area chart, detailed table |
| Analyze profitability trends over time | Tab 2 — bubble chart, gauge, ranked bar chart, heatmap |
| Forecasting for budgeting and financial planning | Tab 3 — native Power BI forecast, budget-vs-actual combo chart, variance analysis, exportable table |
| Interactive report with dynamic visualizations | Company/Year/Quarter slicers, cross-filtering across all three tabs, drill-down via `DimDate`/`DimCompany` |
| Actionable insights | Zero-line cash flow alerts, benchmark gauge, ranked winners/problem units, forecast variance trend |
