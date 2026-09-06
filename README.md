# OVERVIEW

This repository contains Big Query code to build an automated 13-Week Cash Flow modelling system built using **Google Sheets** (data input), **Google BigQuery** (transformation & building the cash flow enging), and **Data Studio** (executive dashboard & granular ledger). The architecture moves heavy financial math—such as dynamic date spines, rolling cash balances, and variance aggregations—directly into BigQuery SQL views. This ensures fast Data Studio rendering, eliminates client-side metric lag, and makes the reporting pipeline **100% dynamic and zero-maintenance**.

# DASHBOARD & GOOGLE SHEET TEMPLATE

Google Sheet Template - https://docs.google.com/spreadsheets/d/1-17dIbItTSk_YAnB-5KVlWBnzcrYf8_-Sy7J-6swSJA/edit?usp=sharing

Data Studio Dashboard - https://datastudio.google.com/reporting/c7850be5-a9b9-48d7-8018-45977b806615

# KEY DIFFERENCES: GOOGLE SHEETS v DATA WAREHOUSE

Atraditional **Google Sheets 13-Week Cash Flow model** relies exclusively on manual inputs and is almost strictly **Forecast-only**, this BigQuery-powered architecture introduces a **Dynamic Variance Engine** by seamlessly combining both **Actuals** and **Forecasts**.

| Feature / Metric | Google Sheets (Standalone) | BigQuery Warehouse Architecture |
| :--- | :--- | :--- |
| **Primary Focus** | Forward-looking estimates only (**Forecast-only**). | Blended evaluation (**Forecast vs. Actuals**). |
| **Actuals Tracking** | Requires manual cell overwrites or fragile multi-tab sheet formulas. | Fully automated blending via BigQuery dynamic date spines and SQL union layers. |
| **Forecast Accuracy** | Hard to measure without breaking historical baseline models. | Real-time variance accounting (`£` delta and `%` accuracy) without touching raw logs. |
| **Maintenance** | High risk of broken cell references as transaction volume grows. | **Zero-maintenance**: dynamic view boundaries handle new incoming transactions automatically. |

### When to Use Each Approach?

* **Google Sheets Only:** Ideal for small, early-stage businesses where financial tracking is simple, transaction volume is low, and side-by-side variance evaluation is not yet critical.
* **BigQuery Warehouse Approach:** Essential for scaling or complex businesses that need real-time budget leakage tracking, automated runway evaluation, and reliable investor/board reporting. 

# GOOGLE SHEETS - DATA INPUT

Raw transaction logs and driver forecasts are maintained in Google Sheets. The sheet contains two main tabs: **`Forecast`** and **`Actual`**. Data must remain flat (row-by-row) to ensure continuous ingestion into BigQuery

- In the Google Sheet template shared is how categories and sub cateogires are aligned, this will need to be decided in advance
- There are two tabs in the Google Sheet for Actuals and Forecasts, Forecasts is the focal point of the 13-week cashflow but the goal should be to also populate the Actuals provides addtional insights. Without actuals, you can never answer: "How good are our forecasts?"

### GOOGLE SHEET SCHEMA & DATA POINT DEFINITIONS

| Field Name | Data Type | Description & Usage | Example |
| :--- | :--- | :--- | :--- |
| **`Date`** | `DATE` (YYYY-MM-DD) | The specific date of the actual cash transaction or projected forecast item. | `2026-07-20` |
| **`Category`** | `STRING` | Top-level financial classification (`Revenue`, `Cost of Goods Sold`, `Operating Expenses`, `Financing Activities`). | `Operating Expenses` |
| **`Sub_Category`** | `STRING` | Granular breakdown of the category used for cost analysis and drill-downs. | `Marketing & Advertising` |
| **`Amount`** | `NUMERIC` | Net monetary value. **Inflows must be positive numbers; outflows must be negative numbers.** | `-6000` |
| **`Notes`** | `STRING` | Operational notes, vendor names, or context explaining the transaction line item. | `Meta & Google ad spend` |


> **Note:** The tab name (`Forecast` or `Actual`) automatically maps to the `Type` / `actual_v_forecast` column inside BigQuery views.

## BIG QUERY VIEWS 

The repository contains four core SQL view definitions powering the reporting layer:

### `1.0_master-view` (Granular Ledger View)
* **Purpose:** Combines `Forecast` and `Actual` logs into a unified dataset, calculates calendar/ISO week numbers, and formats `Week_Label` strings
* **Primary Use:** Powers **Page 1 & 2 (Page 1: Scorecards + 13 Week Cash Balance Forecasted + Weekly Cash In & Cash Out + 13 Week Expense Outflow + Weekly Net Cash Flow. Page 2: Scorecards + Detailed Weekly Cash Flow)**

#### Schema Breakdown
| Column Name | Type | Key Calculation / Notes |
| :--- | :--- | :--- |
| `Date` | `DATE` | Raw transaction date from Google Sheets. |
| `Year` | `INTEGER` | Extracted calendar year (`EXTRACT(YEAR FROM Date)`). |
| `Week_Number` | `INTEGER` | ISO week number (`EXTRACT(ISOWEEK FROM Date)`). |
| `Week_Label` | `STRING` | Formatted year-week identifier (`YYYY-WXX`, e.g., `2026-W30`). |
| `Category` | `STRING` | Cash flow category. |
| `Sub_Category` | `STRING` | Detailed line item category. |
| `Amount` | `NUMERIC` | Transaction amount (+ for inflows, - for outflows). |
| `Notes` | `STRING` | Qualitative context/memo. |
| `Type` | `STRING` | Source classification (`Actual` vs `Forecast`). |

### `1.1_weekly-summary-view` (Rolling Cash Flow Engine)
* **Purpose:** Aggregates net cash flows by week and applies SQL window functions (`SUM() OVER (...)`) to compute exact rolling `Opening Cash` and `Ending Cash` positions week-over-week
* **Primary Use:** Powers **Page 2 (Weekly Cash Flow)**

#### Schema Breakdown
| Column Name | Type | Key Calculation / Notes |
| :--- | :--- | :--- |
| `week_label` | `STRING` | ISO week identifier (e.g., `2026-W30`). |
| `week_start_date` | `DATE` | Start date (Monday) of the respective week. |
| `actual_v_forecast` | `STRING` | Distinguishes whether the weekly summary represents `Actual` or `Forecast`. |
| `amount` | `NUMERIC` | Net weekly cash flow (`SUM(Amount)` for that week). |
| `opening_cash` | `NUMERIC` | Cash balance at the start of the week. Calculated dynamically from baseline starting cash + prior cumulative net flows. |
| `ending_cash` | `NUMERIC` | Cash balance at the end of the week (`opening_cash + amount`). 

### 1.2_actual-v-forecast_summary (Macro Variance Engine)

* **Purpose:** Pivots raw transaction rows into weekly side-by-side totals and computes absolute and percentage variance using a dynamic daily date grid.
* **Primary Use:** **Powers Page 3 (Scorecards + Weekly Net Cash Flow + Weekly Variance Summary)**

#### Schema Breakdown

| Column Name | Type | Key Calculation / Notes |
| :--- | :--- | :--- |
| **`Week_Label`** | `STRING` | Formatted year-week identifier (`YYYY-WXX`, e.g., `2026-W18`). |
| **`week_start_date`** | `DATE` | Start date (Monday) of the week (`DATE_TRUNC(Date, ISOWEEK)`). |
| **`forecast_net_flow`** | `NUMERIC` | Total baseline projected cash movement (`SUM(CASE WHEN Type = 'Forecast' THEN Amount ELSE 0 END)`). |
| **`actual_net_flow`** | `NUMERIC` | Total realized bank transaction movement (`SUM(CASE WHEN Type = 'Actual' THEN Amount ELSE 0 END)`). |
| **`variance_amount`** | `NUMERIC` | Absolute monetary difference (`actual_net_flow - forecast_net_flow`). |
| **`variance_pct`** | `PERCENT` | Relative performance delta (`SAFE_DIVIDE(variance_amount, ABS(forecast_net_flow))`). |

### 1.3_actual-v-forecast_category-variance (Category & Cost Leakage Breakdown)

* **Purpose:** Aggregates performance by category and sub-category to pinpoint specific operational budget overspends and revenue variances.
* **Primary Use:** **Powers Page 3 (Cost & Expense Leakage Breakdown Table)**

#### Schema Breakdown

| Column Name | Type | Key Calculation / Notes |
| :--- | :--- | :--- |
| **`Week_Label`** | `STRING` | Formatted year-week identifier (`YYYY-WXX`). |
| **`week_start_date`** | `DATE` | Start date (Monday) of the week. |
| **`Category`** | `STRING` | Top-level financial classification (`Revenue`, `Operating Expenses`, `COGS`). |
| **`Sub_Category`** | `STRING` | Detailed operational line item category (e.g., `Marketing & Advertising`). |
| **`forecast_amount`** | `NUMERIC` | Total projected amount for the category/sub-category. |
| **`actual_amount`** | `NUMERIC` | Total realized actual amount for the category/sub-category. |
| **`variance_amount`** | `NUMERIC` | Absolute monetary difference (`actual_amount - forecast_amount`). |

### 1.4_live-starting-cash-view (Live Starting Cash Baseline)

* **Purpose:** Establishes the dynamic opening liquidity balance derived from live bank settlements to seed the rolling 13-week forecast model.
* **Primary Use:** Powers the "Live Starting Cash" scorecard on Page 1 (Executive Summary) and feeds baseline calculations across dashboard components.

#### Schema Breakdown

| Column Name | Type | Key Calculation / Notes |
| :--- | :--- | :--- |
| `Date` | `DATE` | Specific date corresponding to the cash balance entry. |
| `Year` | `INTEGER` | Calendar year identifier (e.g., `2026`). |
| `Week_Number` | `INTEGER` | Sequential numerical week identifier (`1` to `52`). |
| `Week_Label` | `STRING` | Formatted year-week identifier (`YYYY-WXX`). |
| `Starting_Cash` | `NUMERIC` | The starting bank cash balance for Week 1 of the selected 13-week period |

## KPI METRICS DICTIONARY

| KPI Name | Page No. | GitHub No. | Description & Example |
| :--- | :---: | :---: | :--- |
| **Live Starting Cash** | Page 1 | 1.4_live-starting-cash-view | Displays the opening bank balance at the start of the 13-week forecast period.<br>**Example:** `£13,800` |
| **13 Week Minimum Cash Balance** | Page 1 | 1.0_master-view | Represents the lowest projected cash position over the 13-week forecast horizon.<br>**Example:** `£21,650` |
| **13 Week Net Cash Flow** | Page 1 | 1.0_master-view | Total net cash movement (inflows minus outflows) forecasted across the entire 13 weeks.<br>**Example:** `£710,519` |
| **Ending Cash Balance** | Page 1 | 1.0_master-view | Projected ending cash balance progression per week (`2026-W01` to `2026-W13`).<br>**Example:** `£21,650` in W01 growing to `£154,803` in W13 |
| **Filtered Net Cash Flow** | Page 2 | 1.0_master-view | Dynamic scorecard showing total net cash movement based on active filters (e.g. Category, Sub Category).<br>**Example:** `£710,519` |
| **Forecast Net Flow** | Page 3 | View 1 | Baseline 13-week projected net cash movement.<br>**Example:** `£710,519` |
| **Actual Net Flow** | Page 3 | View 1 | Total realized net cash movement recorded to date.<br>**Example:** `£829,200` |
| **Net Variance** | Page 3 | View 1 | Absolute monetary variance between Actual and Forecasted cash flows (`Actual - Forecast`).<br>**Example:** `£118,681` |
| **Net Variance %** | Page 3 | View 1 | Percentage variance of actual performance against budget/forecast (`Net Variance / Forecast`).<br>**Example:** `16.70%` |
| **Opening Cash** | Page 3 | View 2 | Starting balance for the chronologically earliest week in the selected view range (rendered as 1-row table scorecard).<br>**Example:** `£15,870` (or `£154,803` for W14) |
| **Actual Weekly Cash Flow** | Page 3 / Page 4 | View 2 | High-level weekly summary table with fields: *Week Label*, *Week Starting Date*, *Opening Cash*, *Amount*, and *Ending Cash*.<br>**Example:** W01 Amount = `£7,850`, Ending Cash = `£23,720` |


