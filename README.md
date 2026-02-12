# 📊 Sales Performance & Delivery Risk — Power BI Dashboard

> **Integrating Sales, Logistics, and Operational Risk for Executive Decision-Making**

This project is part of my professional portfolio as a **Data Analyst**, with a strong focus on **Business Intelligence and Data Storytelling applied to executive decision-making**.

---

## 🎯 Executive Summary

Between **2019 and 2022**, **Bf.lubs** — a fictitious Brazilian company in the lubricants and additives sector — achieved **37% revenue growth** while maintaining stable margins (~55%). However, integrated data analysis revealed a critical operational risk:

| Metric | Value | Target |
|---|---|---|
| Revenue Growth (2019–2022) | **+37%** | — |
| Gross Margin | **~55%** | — |
| Delivery SLA | **40%** | **75%** |
| Revenue Exposed to Delays | **~60%** | — |

> ⚠️ Commercial growth was not matched by proportional logistics capacity expansion, creating significant revenue exposure to delivery risk.

The objective of this project was to develop an **analytical Power BI dashboard** that integrates **sales, cost, and delivery data**, providing executive visibility, identifying **revenue at risk**, and supporting **data-driven strategic decisions**.

---

## 🏭 Business Context

**Bf.lubs** is a fictitious Brazilian company operating in the **lubricants and additives sector**, with multiple industrial plants and nationwide operations.

Between 2019 and 2022, the company experienced consistent revenue growth and increasing order volumes. Alongside this growth, **operational challenges related to Delivery SLA** were identified — SLA performance below expectations directly increased **revenue exposure to operational risk**, affecting revenue predictability, customer satisfaction, and contractual commitments.

**Project was justified by the need to:**
- Consolidate sales, cost, and delivery data into a single analytical model
- Monitor revenue growth without losing visibility into **operational risks**
- Identify plants, divisions, and products with the **highest exposure to delivery delays**
- Support strategic decisions related to **operational prioritization, risk mitigation, and continuous SLA improvement**

---

## 🛠️ Tech Stack

| Tool | Usage |
|---|---|
| **Power BI Desktop** | Dashboard development and data visualization |
| **Power Query (M)** | ETL processes — data cleaning, transformation, and enrichment |
| **DAX** | KPI calculation, time intelligence, and risk measures |
| **Figma** | Dashboard layout and visual design |

**Data Sources:** Structured CSV and Excel files with historical sales and delivery data (2019–2022)

---

## 🗂️ Data Architecture

The model follows a **Star Schema**, with `fat_sales` as the central fact table connected to five dimension tables via foreign keys.
<img width="1178" height="737" alt="modelo de relacionamento" src="https://github.com/user-attachments/assets/f2117e6f-2134-4d80-aff8-2dd1cac96ee8" />



**All relationships are N:1 with single-direction filtering** — no bidirectional relationships, avoiding filter ambiguity and performance degradation.

### Tables Overview

| Table | Type | Description |
|---|---|---|
| `fat_sales` | Fact | Core table with orders, revenue, margin, cost, and delivery data |
| `dim_calendar` | Dimension | Dynamic date dimension with temporal attributes |
| `dim_customer` | Dimension | Customer master data enriched with geographic attributes |
| `dim_plant` | Dimension | Plant/factory structural and geographic information |
| `dim_material` | Dimension | Product and material descriptive data |
| `delivery_status_detail` | Support | Standardized delivery status classification |

---

## ⚙️ ETL Process

All ETL processes were performed in **Power Query using M language**, including data cleaning, standardization, missing value treatment, derived column creation, and relational integrity validation.

### Dimension Tables

**`dim_plant`**
- Connection to tab-delimited TXT file
- Conversion of Latitude and Longitude to numeric (locale: en-US)
- Standardization of text fields and column renaming to uppercase naming

**`dim_location`** *(not loaded to model — used for enrichment only)*
- Connection to Excel file
- Data type standardization (identifiers as text, coordinates as numeric, dates as datetime)
- Retention of control fields: ACTIVE status and division

**`dim_material`**
- Connection to Excel file
- Renaming `MATERIAL_ID` → `PRODUCT_ID` for consistency with the fact table

**`dim_customer`** *(enriched via merge)*
- Left Outer Join with `dim_location` using `LOCATION_ID`
- Expansion of country, state, city, latitude, longitude, geozone, and active status
- Column standardization: CUSTOMER_ID, DIVISION, DIVISION_MANAGER

### Operational Tables

**`deliveries`**
- Conversion of date fields to Date type (locale: en-US)
- Temporal consistency validation (Actual ≥ Expected)
- Creation of calculated column: `DELIVERY_DELAY_DAYS`

**`fat_sales`** *(central fact table)*
- Consolidation of multiple CSV files via custom function
- Data type standardization: IDs → Text | Revenue, costs, volume → Numeric | SALES_DATE → Date
- Critical transformation: extracting the first 4 characters of `CUSTOMER_ID` for consistency with `dim_customer`
- Strategic merge with `deliveries` using Left Outer Join on `ORDER_ID`
- Creation of calculated column: `DELIVERY_STATUS_DETAIL`

### Supporting Tables

**`dim_calendar`** *(created dynamically in Power Query)*
- Continuous date range from MIN(SALES_DATE) to MAX(SALES_DATE)
- Derived attributes: Year, Month, Day, Quarter, Week, Weekday, YearMonth, YearMonthDesc

**`delivery_status_detail`** *(manually created)*

| Status | Range |
|---|---|
| Early | Delivered before expected date |
| On Time | Delivered on expected date |
| 0–2 Days Late | Up to 2 days past expected date |
| 3–4 Days Late | 3–4 days past expected date |
| 5–7 Days Late | 5–7 days past expected date |

---

## 📐 Data Modeling

### Key Relationships

| From | To | Key | Cardinality |
|---|---|---|---|
| `fat_sales` | `dim_customer` | CUSTOMER_ID | N:1 |
| `fat_sales` | `dim_plant` | PLANT_ID | N:1 |
| `fat_sales` | `dim_material` | PRODUCT_ID | N:1 |
| `fat_sales` | `dim_calendar` | SALES_DATE | N:1 |
| `fat_sales` | `delivery_status_detail` | DELIVERY_STATUS_DETAIL | N:1 |

> `fat_sales[ACTUAL_DELIVERY_DATE]` → `dim_calendar[DATE]`: **inactive relationship**, activated via `USERELATIONSHIP()` in specific DAX measures.

---

## 📏 DAX Measures

Measures are organized by business category for readability and maintenance.

### 💰 Financial

```dax
Revenue = SUM(fat_sales[REVENUE])

Total Cost = SUM(fat_sales[TOTAL_COST])

Gross Profit = [Revenue] - [Total Cost]

Gross Margin % = DIVIDE([Gross Profit], [Revenue])
```

### 📦 Operational

```dax
Total Orders = COUNT(fat_sales[ORDER_ID])

Total Deliveries =
CALCULATE(
    DISTINCTCOUNT(fat_sales[ORDER_ID]),
    USERELATIONSHIP(fat_sales[ACTUAL_DELIVERY_DATE], dim_calendar[DATE])
)

AOV = DIVIDE([Revenue], [Total Orders], 0)
```

### 🚚 Logistics & SLA

```dax
Deliveries On Time =
CALCULATE(
    [Total Deliveries],
    delivery_status_detail[StatusDetail] IN {"Early", "On Time"}
)

Deliveries Overdue =
CALCULATE(
    [Total Deliveries],
    delivery_status_detail[StatusDetail] <> "Early"
        && delivery_status_detail[StatusDetail] <> "On Time"
)

Delivery SLA % = DIVIDE([Deliveries On Time], [Total Deliveries], 0)

Avg Delivery Delay = AVERAGE(fat_sales[DELIVERY_DELAY_DAYS])
```

### ⚠️ Risk

```dax
Sales at Risk =
CALCULATE(
    [Revenue],
    delivery_status_detail[StatusDetail] <> "Early"
        && delivery_status_detail[StatusDetail] <> "On Time"
)

Revenue at Risk % = DIVIDE([Sales at Risk], [Revenue], 0)
```

### 📅 Time Intelligence

```dax
Revenue LY =
CALCULATE(
    [Revenue],
    SAMEPERIODLASTYEAR(dim_calendar[DATE])
)

Revenue vs LY = [Revenue] - [Revenue LY]

Revenue vs LY % = DIVIDE([Revenue vs LY], [Revenue LY], 0)
```

---

## 📊 Dashboard Structure

The dashboard is composed of **5 strategically structured pages** + **2 custom tooltips**.

---

### Page 1 · Executive Overview | Business Performance

**Target audience:** CEOs, Commercial Directors, VPs of Operations

**KPI Cards:** Revenue · Total Orders · Margin · Margin % · Total Cost · Delivery SLA *(with visual alert when below target)*

**Visuals:**
- **Revenue & Margin Trend** — Combined line/column chart showing revenue growth with stable ~55% margin
- **Revenue & Cost Trend** — Stacked area chart identifying margin compression/expansion periods
- **Revenue Concentration Trend** — Line chart showing ~81.5% concentration in top 3 plants
- **Order Demand Trend** — Column chart for capacity planning and forecasting
<img width="1456" height="812" alt="Executive Overview pagina 1" src="https://github.com/user-attachments/assets/79df72fc-390b-467f-8474-4b92094563c3" />

---

### Page 2 · Sales Performance | Revenue, Margin & Orders

**Target audience:** Sales Managers, Commercial Coordinators, BI Analysts

**KPI Cards:** Revenue · Total Orders · Margin · Margin % · Total Cost · AOV

**Visuals:**
- **Revenue by Business Line** — Donut chart: Lubricants 88.14% ($796M) / Additives 11.86% ($107M)
- **Sales by Month** — Clustered column chart with current year vs. previous year (LY) comparison
- **Sales Year-to-Date** — Accumulated line chart for annual forecast validation
- **Plant Performance Matrix** — Detailed table with State, Sales, Sales vs LY, Orders, AOV, Margin, Margin %, Cost, Customers — with conditional formatting
- **Sales by Division / Plant** — Horizontal bar chart: Midwest/Southeast $601M · South $246M · North/Northeast $56M
<img width="1445" height="809" alt="pagina2 " src="https://github.com/user-attachments/assets/b92be100-1276-4f31-ad46-e2886e90c496" />

---

### Page 3 · Deliveries | Operational Performance

**Target audience:** Logistics Managers, Supply Chain Managers, Operations Directors

**KPI Cards:** Total Deliveries · SLA Compliance · Deliveries Overdue · Sales at Risk · Avg Delivery Delay

**Visuals:**
- **Delivery SLA % Trend vs Target** — Line chart with 75% reference line and gap shading
- **Delivery Status Distribution** — Stacked bar: Early/On Time 372.3K · 1–2 Days Late 185.8K · 3–4 Days Late 185.9K · 5–7 Days Late 185.3K
- **Delivery Performance by Plant/Division** — Horizontal bar: Midwest/Southeast $646M · South $235M · North/Northeast $58M
<img width="1451" height="812" alt="deliveriew overview" src="https://github.com/user-attachments/assets/83a28e48-5070-4bd2-b4e2-62fa04374de7" />

---

### Page 4 · Revenue at Risk | Financial Impact of Delivery Delays

**Target audience:** Supply Chain Managers, Operations Directors, Risk Analysts

**KPI Cards:** Sales at Risk · Revenue at Risk % · Avg Delivery Delay · High Risk Plant

**Visuals:**
- **Sales at Risk Trend** — Monthly line chart revealing seasonal risk patterns
- **Sales at Risk by Plant** — Horizontal bar ranking critical plants
- **Sales at Risk by Division** — Vertical bar for regional risk comparison
- **Top Drivers of Sales at Risk** — Product category ranking for targeted intervention
- **Sales vs SLA Scatter Plot** — Bubble chart with quadrant analysis (bubble size = Sales at Risk):
  - ⚠️ *High Risk*: Low SLA + High Sales
  - ✅ *Strong Performance*: High SLA + High Sales
<img width="1448" height="812" alt="Revenue at risk" src="https://github.com/user-attachments/assets/75c8c0b9-aeb1-440a-895a-02678a21eaed" />

---

### Page 5 · Delivery Drill-Down

**Target audience:** Plant Managers, Local Logistics Coordinators, Dispatch Supervisors

**KPI Cards (dynamic by selected plant):** Total Deliveries · On-Time Delivery % · Deliveries Overdue · Avg Delivery Delay

**Visuals:**
- **Main Driver of Delay by Product** — Detailed table with product-level ranking
- **Main Driver of Delay by Category** — Bar chart with delay volume and percentage
- **Main Driver of Delay by Product and Category** — Combined analysis for targeted interventions
- **Delivery Trend Over Time** — Stacked area chart: Early / On Time / Overdue over time
<img width="1454" height="813" alt="Delivery detail" src="https://github.com/user-attachments/assets/4039b276-9238-481c-9349-262a5a856e50" />

---

### Custom Tooltips

**Tooltip 1 — Product Detail · Delivery & Delay Analysis**
Fast access to delivery performance by product, including delay trends and key drivers.

<img width="453" height="513" alt="delivery" src="https://github.com/user-attachments/assets/dc507587-07a0-478b-8e6c-7914e09af25a" />

**Tooltip 2 — Sales at Risk · Financial Impact Overview**
Immediate view of the financial impact of delivery delays by plant — combining KPIs, trends, and rankings for executive analysis.

<img width="450" height="384" alt="Sales risk" src="https://github.com/user-attachments/assets/ab20c582-234f-4556-89e1-88bed90cbd0e" />

---

## 💡 Key Insights & Recommendations

### Current Diagnosis

- ✅ Strong revenue growth (+37%) with stable margins (~55%) — indicates financial efficiency
- ⚠️ ~60% of total revenue exposed to delivery delays
- 🔴 Delivery SLA of **40%** — significantly below the 75% target
- 🔴 Delays scale proportionally with sales growth → structural logistics capacity bottleneck

### Priority Recommendations

**1. Immediate Investment in Logistics Capacity** — focus on highest-risk plants:

| Plant | Revenue at Risk |
|---|---|
| Sorocaba | $231M |
| Blumenau | $147M |
| Vitória | $64M |

**2. Process Review & Standardization**
- Integrate demand forecasting with logistics planning
- Optimize distribution routes
- Implement differentiated SLAs by customer, product, or category

**3. Continuous Monitoring & Governance**
- Weekly SLA review rituals
- Automated alerts for high-risk orders
- Near real-time monitoring for plant managers

**4. Risk-Adjusted Profitability Analysis**
- Incorporate delay opportunity costs
- Reassess pricing and product mix
- Prioritize customers and products with balanced margin, volume, and SLA

---

## 📁 Project Structure

```
powerbi-Sales-2019-2022-bflubs-analysis/
│
├── 📂 data/
│   ├── fat_sales/                  # CSV files (consolidated via Power Query custom function)
│   ├── deliveries.xlsx
│   ├── dim_plant.txt
│   ├── dim_customer.xlsx
│   ├── dim_location.xlsx
│   └── dim_material.xlsx
│
├── 📂 pbix/
│   └── Sales_Performance_Delivery_Risk.pbix
│
└── README.md
```

🔗 [**View Interactive Dashboard**](https://app.powerbi.com/view?r=eyJrIjoiZGM3N2NjZmEtZThmMy00NjY4LTg4MjgtOWUyNDExOGYxZDY2IiwidCI6ImJlMzIyOTBmLTRkNTgtNGY1Yy05ODY2LWJiZmQxNzMwZGU3OCIsImMiOjN9)
---


*Analysis Period: 2019–2022 · Last Update: January 2026*
