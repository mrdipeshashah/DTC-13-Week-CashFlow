# DTC 13-WEEK CASH FLOW

An automated 13-Week Cash Flow modeling system built using **Google Sheets** (data input), **Google BigQuery** (transformation & building the cash flow enging), and **Data Studio** (executive dashboard & granular ledger)

Google Sheet Template - https://docs.google.com/spreadsheets/d/1-17dIbItTSk_YAnB-5KVlWBnzcrYf8_-Sy7J-6swSJA/edit?usp=sharing
Data Studio Dashboard - https://datastudio.google.com/reporting/c7850be5-a9b9-48d7-8018-45977b806615

# 1. GOOGLE SHEETS - DATA INPUT

Raw transaction logs and driver forecasts are maintained in Google Sheets. The sheet contains two main tabs: **`Forecast`** and **`Actual`**. Data must remain flat (row-by-row) to ensure continuous ingestion into BigQuery

- In the Google Sheet tempalte shared is how categories and sub cateogires are aligned, this will need to be decided in advance
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

## 2.BIG QUERY VIEWS 

The repository contains four core SQL view definitions powering the reporting layer:

### `1.0_master_view` (Granular Ledger View)
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

### `1.1_weekly_summary_view` (Rolling Cash Flow Engine)
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

### 1.2_actual_v_forecast_summary (Macro Variance Engine)

* **Purpose:** Pivots raw transaction rows into weekly side-by-side totals and computes absolute and percentage variance using a dynamic daily date grid.
* **Primary Use:** Powers Page 3 (Scorecards + Weekly Net Cash Flow + Weekly Variance Summary)**

#### Schema Breakdown

| Column Name | Type | Key Calculation / Notes |
| :--- | :--- | :--- |
| **`Week_Label`** | `STRING` | Formatted year-week identifier (`YYYY-WXX`, e.g., `2026-W18`). |
| **`week_start_date`** | `DATE` | Start date (Monday) of the week (`DATE_TRUNC(Date, ISOWEEK)`). |
| **`forecast_net_flow`** | `NUMERIC` | Total baseline projected cash movement (`SUM(CASE WHEN Type = 'Forecast' THEN Amount ELSE 0 END)`). |
| **`actual_net_flow`** | `NUMERIC` | Total realized bank transaction movement (`SUM(CASE WHEN Type = 'Actual' THEN Amount ELSE 0 END)`). |
| **`variance_amount`** | `NUMERIC` | Absolute monetary difference (`actual_net_flow - forecast_net_flow`). |
| **`variance_pct`** | `PERCENT` | Relative performance delta (`SAFE_DIVIDE(variance_amount, ABS(forecast_net_flow))`). |

### 1.3_actual_v_forecast_category_variance (Category & Cost Leakage Breakdown)

* **Purpose:** Aggregates performance by category and sub-category to pinpoint specific operational budget overspends and revenue variances.
* **Primary Use:** Powers Page 3 (Cost & Expense Leakage Breakdown Table)**

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

## 3. Looker Studio Dashboard & KPI Metrics

The dashboard is structured into two main views to serve both high-level executive reviews and detailed auditing.

### Page 1: Executive 13-Week Overview

#### Scorecard Metrics
| Metric Name | Underlying Field | Aggregation | Definition & Meaning |
| :--- | :--- | :--- | :--- |
| **LIVE STARTING CASH** | `opening_cash` | `MIN` or `MAX` | The starting bank cash balance for Week 1 of the selected 13-week period. |
| **13 WEEK MINIMUM CASH BALANCE** | `ending_cash` | `MIN` | The lowest predicted cash balance over the next 13 weeks. Serves as a **liquidity safety metric** to flag cash crunch risks. |
| **13 WEEK NET CASH FLOW** | `amount` | `SUM` | Total net cash generated or consumed across the entire 13-week period (Sum of Inflows - Sum of Outflows). |

#### Charts & Tables
1. **13-Week Cash Balance Forecasted (Column Chart):**
   * **Dimension:** `week_label`
   * **Metric:** `ending_cash` (`MAX` aggregation)
   * **Purpose:** Visualizes liquidity trends and ending weekly cash trajectory over time.
2. **Weekly Cash Flow Summary Table:**
   * **Columns:** `Week Starting Date` ➔ `Week Label` ➔ `Opening Cash` ➔ `Amount` ➔ `Ending Cash`
   * **Purpose:** Displays full accounting rolling math where each week's `Ending Cash` carries over as the next week's `Opening Cash`.

---

### Page 2: Detailed Breakdown & Ledger

* **Data Source:** Connected directly to `master_13weekcashflow_view`.
* **Table Fields:** `Date`, `Week_Label`, `Category`, `Sub_Category`, `Notes`, `Type`, `Amount`.
* **Purpose:** Line-by-line audit ledger allowing teams to inspect individual transactions, vendor payments, and specific marketing/inventory allocations.

---

## 4. How to Operate & Filter

1. **Page-Level Control Dropdown (`Type` / `actual_v_forecast`):**
   * Located at the top of the dashboard.
   * Toggle to **`Actual`** to review historical performance and audit real ledger entries.
   * Toggle to **`Forecast`** to view projected 13-week liquidity, future minimum cash balances, and budget allocations.
2. **Date Range Picker:**
   * Adjusts the 13-week rolling window dynamically across both dashboard pages simultaneously without breaking underlying BigQuery calculations.
