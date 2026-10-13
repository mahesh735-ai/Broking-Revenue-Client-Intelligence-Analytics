# Business Requirements Document
## Broking Revenue & Client Intelligence Analytics

| | |
|---|---|
| **Version / Date** | v1.0 / October 2026 |
| **Author** | Mahesh Thakare, Final-Year B.Tech CSE (Data Analyst) |
| **Project type** | Independent portfolio project on a fully synthetic dataset. The business context is modeled on common patterns of Indian discount broking platforms. It is not affiliated with any company, and the stakeholders below are simulated. |

---

## 1. Business Context

A discount broking platform earns a small flat fee per executed order. Equity delivery and mutual funds carry zero brokerage, and account opening is free. This makes client acquisition easy, but it creates three problems the business cannot see in its standard reports:

1. **Dormant clients.** Many clients open an account, trade once or twice and go quiet. They cost money to acquire and earn almost nothing.
2. **Revenue concentration.** Income comes mainly from derivatives (F&O). Products that bring volume (delivery, mutual funds) earn nothing.
3. **Failures at peak load.** Orders fail most when the market opens and on F&O expiry days, which is exactly when clients trade most.

## 2. Business Objectives

| # | Objective |
|---|---|
| O1 | Show which products and client groups produce the revenue, and how much dormant clients contribute |
| O2 | Show when, why and how often orders fail, and how trading changes on expiry days |
| O3 | Give each team a clear, actionable view in a single two-page dashboard |

## 3. Stakeholders and What They Need

| Stakeholder (simulated) | Question they need answered | Where it is answered |
|---|---|---|
| Head of Product | Which products earn revenue? Which clients are slipping away? | Executive Overview |
| VP Marketing | Which acquisition channel brings better clients? How big is the dormant base? | Client & Market Risk page |
| Head of Engineering | When and why do orders fail? Is expiry-day load a problem? | Client & Market Risk page |
| Risk / Operations | What happened to one specific client's trades? | Customer Drillthrough page |

## 4. Scope

**In scope:** synthetic trade-level data generation (Python), data cleaning (Power Query), a star-schema model, DAX measures, a two-page dashboard, a drillthrough page and a hover tooltip page.

**Out of scope:** real or live market data, real client data, real-time refresh, clearing or settlement integrations, tax calculations, and any machine learning.

## 5. Business Requirements

| ID | Requirement | Delivered by |
|---|---|---|
| BR-01 | Show total revenue, trades, active clients, failure rate and month-over-month growth | Page 1 KPI cards |
| BR-02 | Show revenue over time and by product segment | Page 1 trend line, segment bar |
| BR-03 | Split revenue between active and dormant clients | Page 1 Active vs Dormant chart |
| BR-04 | Show where clients are located (city tier) and who the top 5 clients are | Page 1 donut, Top 5 table |
| BR-05 | Show dormancy rate and revenue per active client, and compare acquisition channels | Page 2 KPIs and channel bar |
| BR-06 | Show failed trades by reason and by hour of day, including the peak-hour failure rate | Page 2 charts and KPI |
| BR-07 | Compare F&O Options activity on expiry days against normal trading days | Page 2 chart and Expiry Day Multiplier |
| BR-08 | Let a user open one client's profile and full trade history | Customer Drillthrough page |
| BR-09 | Let a user hover over a product segment and see its monthly revenue trend | Segment Tooltip page |
| BR-10 | Allow filtering by month, segment, city tier, client status, acquisition channel and expiry day | Slicers on both pages |

## 6. Data Requirements

| Table | Purpose | Rows |
|---|---|---:|
| Dim_Customer | Client attributes: city, city tier, acquisition channel, age band, KYC status | 10,000 |
| Dim_Product | 15 products across 7 segments, with brokerage rate | 15 |
| Date Table | Calendar with weekend, market-day and expiry-day flags (Jan 2024 – Jun 2025) | 547 |
| Fact_Trades | One row per order: client, product, date, time, value, brokerage, status, failure reason | ~209K after cleaning |

**Data quality rules**
- Remove duplicate `trade_id` rows (about 0.5% of the raw data) and record how many were found.
- Standardise `city` using trim and proper case.
- Confirm all date fields are real dates, and that every dimension key is unique.
- All data must be synthetic, with no personal information.

## 7. KPI Definitions

| KPI | Definition | Value |
|---|---|---:|
| Total Revenue | Sum of brokerage earned on successful orders | ₹2.75M |
| Total Trades | Count of orders after removing duplicates | 209K |
| Active Clients | Clients whose last trade was within 60 days of the latest date in the data | 5K |
| Transaction Failure Rate | Failed orders ÷ all orders | 4.03% |
| Dormancy Rate | Clients with 60 or more days since their last trade ÷ all clients | 49.38% |
| Revenue per Active Client | Revenue earned from active clients ÷ active clients | ₹533.51 |
| Peak-Hour Failure Rate | Failure rate for orders placed between 9:15 and 10:00 AM | 8.87% |
| Expiry Day Volume Multiplier | Average F&O Options trades per expiry day ÷ per normal trading day (weekends excluded) | 2.09x |

## 8. Acceptance Criteria

- Every headline number on the dashboard matches the source data (checked independently).
- All relationships are one-to-many with single-direction filtering, and the Date Table is marked as the model's date table.
- Slicers, drillthrough and tooltip all work without errors.
- Each page answers its stakeholder questions without needing extra explanation.

## 9. Assumptions and Constraints

- Brokerage is a simplified flat fee per order (₹20 for intraday, F&O, commodity and currency; ₹0 for delivery and mutual funds).
- Dormancy uses a 60-day threshold, and expiry days are simplified to Tuesday and Thursday.
- Results come from synthetic data, so they demonstrate the analysis method and are not claims about any real market or company.
