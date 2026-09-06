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
| `Starting_Cash` | `NUMERIC` | Opening total liquid bank account balance for the given week. |

## DATA STUDIO DASHBOARD & KPI METRICS 

The dashboard is structured into three main pages to serve high-level executive reviews, operational auditing, and variance accounting. Each visual component maps directly back to the SQL views in the repository:

### Visual Component & SQL View Mapping

| Dashboard Page | Visual Component / Chart Title | SQL View / Data Source Code | Key Fields Used |
| :--- | :--- | :--- | :--- |
| **Page 1: Executive 13-Week Overview** | Scorecards (`LIVE STARTING CASH`, `13 WEEK MINIMUM CASH BALANCE`, `13 WEEK NET CASH FLOW`) | **`1.1_weekly_summary_view`** | `opening_cash`, `ending_cash`, `amount` |
| | `WEEKLY CASH IN v CASH OUT` (Stacked Bar) | **`1.0_master_view`** | `Week_Label`, `Category`, `Amount` |
| | `13 WEEK EXPENSE OUTFLOW BREAKDOWN BY CATEGORY` (Horizontal Bar) | **`1.0_master_view`** | `Category`, `Amount` |
| | `13 WEEK EXPENSE OUTFLOW BREAKDOWN BY SUB CATEGORY` (Horizontal Bar) | **`1.0_master_view`** | `Sub_Category`, `Amount` |
| | `WEEKLY NET CASH FLOW` (Bar Chart) | **`1.1_weekly_summary_view`** | `week_label`, `amount` |
| | `13 WEEK CASH BALANCE FORECASTED` (Bar Chart) | **`1.1_weekly_summary_view`** | `week_label`, `ending_cash` |
| | `WEEKLY CASH FLOW` (Summary Table) | **`1.1_weekly_summary_view`** | `week_start_date`, `week_label`, `opening_cash`, `amount`, `ending_cash` |
| **Page 2: Detailed Breakdown & Ledger** | Scorecards (`LIVE STARTING CASH`, `13 WEEK MINIMUM CASH BALANCE`, `13 WEEK NET CASH FLOW`) | **`1.1_weekly_summary_view`** | `opening_cash`, `ending_cash`, `amount` |
| | `DETAILED WEEKLY CASH FLOW` (Granular Audit Table) | **`1.0_master_view`** | `Week_Label`, `Date`, `Category`, `Sub_Category`, `Notes`, `Amount`, `Ending Cash` |
| **Page 3: Variance & Forecast Accuracy** | Scorecards (`FORECAST NET FLOW`, `ACTUAL NET FLOW`, `NET VARIANCE`, `NET VARIANCE %`) | **`1.2_actual_v_forecast_summary`** | `forecast_net_flow`, `actual_net_flow`, `variance_amount`, `variance_pct` |
| | `WEEKLY NET CASH FLOW` (Grouped Bar Chart) | **`1.2_actual_v_forecast_summary`** | `Week_Label`, `forecast_net_flow`, `actual_net_flow` |
| | `WEEKLY VARIANCE SUMMARY` (Macro Summary Table) | **`1.2_actual_v_forecast_summary`** | `Week_Label`, `forecast_net_flow`, `actual_net_flow`, `variance_amount`, `variance_pct` |
| | `COST & EXPENSE LEAKAGE BREAKDOWN` (Category Variance Table) | **`1.3_actual_v_forecast_category_variance`** | `Category`, `Sub_Category`, `forecast_amount`, `actual_amount`, `variance_amount` |

---

### PAGE 1: EXECUTIVE 13-WEEEK OVERVIEW

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
2. **Weekly Cash In v Cash Out (Column Chart):**
   * **Dimension:** `categoty` & `sub-categoty`
   * **Metric:** `amount` (`SUM` aggregation)
   * **Purpose:** Split by category showing the cash in v cash out for the selected 13 week period.
2. **13 Week Expense Outflow by Category & by Sub Category (Column Chart):**
   * **Dimension:** `week_label`
   * **Metric:** `Total Spend` (`SUM` aggregation) (Add Calulated Field ABS(Amount))
   * **Purpose:** Split by category / Sub-Category showing the cash out for the selected 13 week period.
4. **Weekly Cash Flow Summary Table:**
   * **Columns:** `Week Starting Date` ➔ `Week Label` ➔ `Opening Cash` ➔ `Amount` ➔ `Ending Cash`
   * **Purpose:** Displays full accounting rolling math where each week's `Ending Cash` carries over as the next week's `Opening Cash`.

### PAGE 2: DETAILED BREAKDOWN & LEDGER

* **Data Source:** Connected directly to `master_13weekcashflow_view`.
* **Table Fields:** `Date`, `Week_Label`, `Category`, `Sub_Category`, `Notes`, `Type`, `Amount`.
* **Purpose:** Line-by-line audit ledger allowing teams to inspect individual transactions, vendor payments, and specific marketing/inventory allocations.

### PAGE 3: VARIANCE & FORECAST ACCURANCY ANALYSIS

#### Scorecard Metrics

| Metric Name | Underlying Field | Aggregation | Definition & Meaning |
| :--- | :--- | :--- | :--- |
| **FORECAST NET FLOW** | `forecast_net_flow` | `SUM` | Total projected cash movement across the selected date range. |
| **ACTUAL NET FLOW** | `actual_net_flow` | `SUM` | Total realized bank cash movement across the selected date range. |
| **NET VARIANCE** | `variance_amount` | `SUM` | Absolute monetary difference between Actuals and Forecast (`Actual - Forecast`). |
| **NET VARIANCE %** | Calculated Metric | Formula | Relative performance percentage (`(SUM(actual) - SUM(forecast)) / ABS(SUM(forecast))`). |

#### Charts & Tables

1. **Weekly Net Cash Flow (Grouped Column Chart):**
   * **Dimension:** `Week_Label` *(Sorted Ascending by `week_start_date`)*
   * **Metrics:** `forecast_net_flow` (Forecast Net Flow), `actual_net_flow` (Actual Net Flow)
   * **Purpose:** Visualizes side-by-side weekly performance comparison between baseline projections and realized cash flows.

2. **Weekly Variance Summary Table:**
   * **Dimensions:** `Week_Label`
   * **Metrics:** `forecast_net_flow` $\rightarrow$ `actual_net_flow` $\rightarrow$ `variance_amount` $\rightarrow$ `variance_pct`
   * **Purpose:** Provides a weekly line-by-line monetary and percentage variance breakdown for executive reviews.

3. **Cost & Expense Leakage Breakdown Table:**
   * **Dimensions:** `Category` $\rightarrow$ `Sub_Category`
   * **Metrics:** `forecast_amount` $\rightarrow$ `actual_amount` $\rightarrow$ `variance_amount`
   * **Table Filter:** `Exclude Category = 'Revenue'`
   * **Purpose:** Granular category drill-down surfacing operational budget overspends and cost leakage (highlighted in red for negative variances).

## HOW TO OPERATE & FILTER

1. **Page-Level Control Dropdown (`Type` / `actual_v_forecast`):**
   * Located at the top of the dashboard.
   * Toggle to **`Actual`** to review historical performance and audit real ledger entries.
   * Toggle to **`Forecast`** to view projected 13-week liquidity, future minimum cash balances, and budget allocations.
2. **Date Range Picker:**
   * Adjusts the 13-week rolling window dynamically across both dashboard pages simultaneously without breaking underlying BigQuery calculations.
