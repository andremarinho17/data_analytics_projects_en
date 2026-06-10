# Power BI Semantic Model — Sales Report

**Comprehensive Technical Documentation** | Generated: June 2026

---

## 1. Overview of the Semantic Model

### Purpose & Objectives

The **Sales Report** semantic model is designed to support sales performance analysis across salespeople, products, and time periods. It enables reporting on:

- Revenue generated vs. budget targets (attainment analysis).
- Order volume and product-category mix (Drink vs. Food).
- Salesperson ranking, top-3 performers, and individual revenue share.
- Period-over-period comparisons using a dedicated Date dimension.
- Visual enrichment with salesperson and product-group photos via URL columns.

### Key Characteristics

| Property | Value |
|---|---|
| Display culture | `en-US` |
| Source query culture | `en-GB` |
| Load mode | Import (all tables in-memory; no DirectQuery) |
| Time intelligence | Enabled (`__PBI_TimeIntelligenceEnabled = 1`) |
| Development tooling | DevMode annotation present |

---

## 2. Data Sources

| File | Type | Sheet / Object | Feeds Table(s) | Description |
|---|---|---|---|---|
| `SalesData.xlsx` | Excel Workbook | Sheet1 | Sales, Salesperson | Transactional sales records including order dates, quantities, unit prices, and salesperson hierarchy (Salesperson → Supervisor → Manager). |
| `Product.xlsx` | Excel Workbook | Table1 | Product | Product master data: product ID, name, group, and category. |
| `Budget.xlsx` | Excel Workbook | All sheets (expanded) | Budget | Monthly budget amounts per salesperson, originally in a cross-tab (pivot) layout that is unpivoted during load. |
| `Photos.xlsx` | Excel Workbook | Group (table), Salesperson (table) | Product (GroupPhoto), Salesperson (SalespersonPhoto) | URL references for product-group and salesperson images. Merged into dimension tables during Power Query transformation. |

> **Note:** All files are loaded from a local path on the report author's machine (`C:\Users\amoreira\Desktop\…\Database\`). Connection strings will need to be updated when publishing to Power BI Service or moving to a shared location.

---

## 3. Data Model Structure

### 3.1 Tables

| Table | Role | Source | Columns |
|---|---|---|---|
| **Sales** *(Fact)* | Central fact table — one row per order line. | SalesData.xlsx – Sheet1 | OrderDate, OrderNumber, ProductKey, SalespersonKey, Channel, Quantity, UnitPrice, SalesAmount |
| **Budget** *(Fact)* | Monthly budget targets per salesperson. | Budget.xlsx | SalespersonID, Amount, BudgetDate |
| **Product** *(Dimension)* | Product catalogue with grouping and category. | Product.xlsx + Photos.xlsx | ID, ProductName, ProductGroup, ProductCategory, GroupPhoto |
| **Salesperson** *(Dimension)* | Sales team hierarchy (Salesperson → Supervisor → Manager). | SalesData.xlsx + Photos.xlsx | SalespersonKey, Salesperson, Supervisor, Manager, SalespersonPhoto |
| **Date** *(Calculated)* | Date dimension auto-generated from model date range. | `CALENDARAUTO()` | Date, Year, Quarter, Month Number, Month, Day |
| **Key Measures** *(Measures)* | Virtual table used solely to host all DAX measures. | Inline (empty table) | Column1 (hidden placeholder) + 16 measures |
| **DateTableTemplate…** *(Hidden)* | Auto-generated Power BI date template (internal). | System | — |
| **LocalDateTable…** *(Hidden)* | Supports the default date hierarchy on the Date column. | System | Date (with Date Hierarchy) |

### 3.2 Relationships

> The model follows a **star-schema** pattern: two fact tables (*Sales* and *Budget*) are connected to shared dimensions (*Salesperson* and *Date*), with *Product* exclusively serving *Sales*.

| From (Many side) | To (One side) | Type | Active | Notes |
|---|---|---|---|---|
| Sales[ProductKey] | Product[ID] | Many-to-One | Yes | Links each sale to a product. |
| Sales[SalespersonKey] | Salesperson[SalespersonKey] | Many-to-One | Yes | Links each sale to the salesperson responsible. |
| Budget[SalespersonID] | Salesperson[SalespersonKey] | Many-to-One | Yes | Links budget entries to the corresponding salesperson, enabling Revenue vs Budget comparisons. |
| Sales[OrderDate] | Date[Date] | Many-to-One | Yes | Primary date relationship for filtering sales over time. |
| Budget[BudgetDate] | Date[Date] | Many-to-One | Yes | Connects monthly budget amounts to the Date dimension. |
| Date[Date] | LocalDateTable[Date] | Many-to-One | Yes | Internal Power BI relationship enabling the default Date Hierarchy (Year / Quarter / Month / Day). |

### 3.3 Hierarchies

| Table | Hierarchy | Levels (top → bottom) | Purpose |
|---|---|---|---|
| Date (LocalDateTable) | Date Hierarchy *(auto)* | Year → Quarter → Month → Day | Standard drill-down for any date/time axis in visuals. Automatically available when the Date column is used. |
| Salesperson | Organisational hierarchy *(implicit)* | Manager → Supervisor → Salesperson | Enables roll-up from individual salesperson to supervisor and management level. No formal Power BI hierarchy object is defined; columns can be used independently in slicers/rows. |
| Product | Product hierarchy *(implicit)* | ProductCategory → ProductGroup → ProductName | Allows drill-down from high-level category (Drink / Food) to individual products. No formal Power BI hierarchy object is defined. |

---

## 4. Calculated Columns & Measures

### 4.1 Calculated Columns

| Table | Column | Expression / Origin | Data Type | Significance |
|---|---|---|---|---|
| Sales | **SalesAmount** | `Quantity × UnitPrice` (Power Query) | Decimal | The primary revenue metric. Computed row-by-row during data load, avoiding the performance overhead of a DAX calculated column on a large fact table. |
| Date | **Year** | `YEAR('Date'[Date])` | Integer | Used to filter specific calendar years (e.g., *Revenue 2020* measure). |
| Date | **Quarter** | `"Qtr " & QUARTER('Date'[Date])` | Text | Provides a formatted label (Qtr 1 … Qtr 4) for quarterly grouping in visuals. |
| Date | **Month Number** | `MONTH('Date'[Date])` | Integer | Numeric month (1–12) used for correct chronological sorting of the Month text column. |
| Date | **Month** | `FORMAT('Date'[Date], "mmmm")` | Text | Full month name for display on axes and slicers (e.g., January, February). |
| Date | **Day** | `DAY('Date'[Date])` | Integer | Day-of-month number enabling daily granularity in time-series visuals. |

### 4.2 Measures (Key Measures Table)

| Measure | Format | Purpose |
|---|---|---|
| **Revenue** | $#,0 | Total sales revenue — the foundation for most financial KPIs. |
| **Orders** | #,0 | Count of distinct order numbers — measures transaction volume. |
| **Budget** | $#,0 | Sum of budgeted amounts per salesperson/period. |
| **ATP** | $#,0.00 | Average Transaction Price = Revenue ÷ Orders. Indicates average deal value. |
| **Revenue vs Budget** | 0.0% | Ratio of actual revenue to budget. >100% means target exceeded. |
| **Revenue 2020** | $#,0 | Revenue filtered to calendar year 2020 — used for year-specific KPI cards. |
| **Countrows Sales** | 0 | Row count of the Sales table — useful for data validation and load monitoring. |
| **Orders by Drink** | 0 | Order count filtered to ProductCategory = "Drink". |
| **Orders by Drink %** | 0% | Drink orders as a share of total orders — product-mix indicator. |
| **Orders by Food** | 0 | Order count filtered to ProductCategory = "Food". |
| **Orders by Food %** | 0% | Food orders as a share of total orders — product-mix indicator. |
| **% Revenue** | #,0.0% | Each salesperson's revenue as a percentage of total company revenue. |
| **Rank SP** | 0 | Salesperson rank by revenue (1 = highest). Ignores any filter on the Salesperson table so the ranking is always relative to all salespeople. |
| **Revenue Top 3** | $#,0 | Revenue attributable to the top-3 salespeople by revenue. |
| **Orders Top 3** | #,0 | Order count for the top-3 salespeople by revenue. |
| **ATP Top 3** | $#,0.00 | Average Transaction Price for the top-3 salespeople by revenue. |

---

## 5. DAX Formulas

### 5.1 Base Aggregation Measures

These are the atomic building blocks; all other measures reference them.

**Revenue**
```dax
Revenue = SUM(Sales[SalesAmount])
```
Simple column sum. Because `SalesAmount` is pre-calculated in Power Query, this measure is extremely fast even on large row sets.

---

**Orders**
```dax
Orders = DISTINCTCOUNT(Sales[OrderNumber])
```
Uses `DISTINCTCOUNT` rather than `COUNTROWS` to handle cases where a single order spans multiple lines (different products on the same order), avoiding double-counting.

---

**Budget**
```dax
Budget = SUM(Budget[Amount])
```
Aggregates the Budget fact table. Becomes meaningful when combined with Date or Salesperson filters.

---

### 5.2 Ratio & Variance Measures

**ATP (Average Transaction Price)**
```dax
ATP = DIVIDE([Revenue], [Orders], 0)
```
The `DIVIDE` function is used throughout the model instead of the division operator (`/`) to safely return 0 — rather than an error — when the denominator is blank or zero. This is critical when slicing by a salesperson or period with no orders.

---

**Revenue vs Budget**
```dax
'Revenue vs Budget' = DIVIDE([Revenue], [Budget])
```
Tracks budget attainment. A value of 1.0 (100%) means exactly on target. Values above 1 indicate over-performance; below 1 indicate under-performance.

---

**% Revenue**
```dax
'% Revenue' =
DIVIDE(
    [Revenue],
    CALCULATE(
        [Revenue],
        ALL(Salesperson)
    )
)
```
`ALL(Salesperson)` removes any filter on the Salesperson table so the denominator is always the grand total revenue — regardless of which salesperson is currently selected in the report. This correctly expresses each salesperson's share of the total.

---

### 5.3 Category-Mix Measures

**Orders by Drink**
```dax
'Orders by Drink' =
CALCULATE(
    [Orders],
    'Product'[ProductCategory] == "Drink"
)
```

**Orders by Drink %**
```dax
'Orders by Drink %' = DIVIDE([Orders by Drink], [Orders], 0)
```

**Orders by Food**
```dax
'Orders by Food' =
CALCULATE(
    [Orders],
    'Product'[ProductCategory] == "Food"
)
```

**Orders by Food %**
```dax
'Orders by Food %' = DIVIDE([Orders by Food], [Orders], 0)
```

The `CALCULATE` function overrides the current filter context, applying a hard filter on `ProductCategory`. The percentage variants then express category volume as a fraction of overall orders — useful for product-mix pie/donut charts.

---

### 5.4 Time Intelligence

**Revenue 2020**
```dax
'Revenue 2020' =
CALCULATE(
    [Revenue],
    'Date'[Year] = 2020
)
```
Hardcodes the year 2020 to provide a static benchmark figure. Commonly displayed on KPI cards where the report covers a fixed historical period. The calculated column `Date[Year]` (created via `YEAR()`) is the filter target.

---

### 5.5 Ranking & Top-N Measures

**Rank SP**
```dax
'Rank SP' = RANKX(ALL(Salesperson), [Revenue])
```
`RANKX` iterates over all salespeople (regardless of current filter) and assigns rank 1 to the highest revenue earner. `ALL(Salesperson)` ensures rankings are stable even when the visual is filtered to a subset of salespeople.

---

**Revenue Top 3**
```dax
'Revenue Top 3' =
CALCULATE(
    [Revenue],
    TOPN(3, ALL(Salesperson), [Revenue])
)
```

**Orders Top 3**
```dax
'Orders Top 3' =
CALCULATE(
    [Orders],
    TOPN(3, ALL(Salesperson), [Revenue])
)
```

**ATP Top 3**
```dax
'ATP Top 3' =
CALCULATE(
    [ATP],
    TOPN(3, ALL(Salesperson), [Revenue])
)
```

All three *Top 3* measures share the same filter: `TOPN(3, ALL(Salesperson), [Revenue])`. This expression produces a virtual table containing the 3 salespeople with the highest revenue across the whole model, then `CALCULATE` restricts the base measure to that subset. All three metrics (revenue, orders, ATP) are therefore always computed for the same three salespeople — ensuring consistency across KPI cards and comparison visuals.

---

### 5.6 Date Table Calculated Columns (DAX)

```dax
-- Date table is seeded automatically
Date[Date]         = CALENDARAUTO()        -- generates one row per day in model range

Date[Year]         = YEAR('Date'[Date])
Date[Quarter]      = "Qtr " & QUARTER('Date'[Date])
Date[Month Number] = MONTH('Date'[Date])
Date[Month]        = FORMAT('Date'[Date], "mmmm")
Date[Day]          = DAY('Date'[Date])
```

`CALENDARAUTO()` inspects all date columns in the model and creates a continuous date range spanning from the earliest to the latest date found — no manual start/end dates are required. The derived columns extend the base date into the attributes needed by visuals and slicers.

---

*Documentation generated from Power BI Semantic Model definition files (TMDL format) · PowerBIWeek Report.SemanticModel*
