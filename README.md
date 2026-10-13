# Broking-Revenue-Client-Intelligence-Analytics
📈 Broking Intelligence Dashboard: Power BI model analyzing 210K+ trades across Indian discount brokers. Uncovers 64% F&amp;O revenue concentration, 8.87% peak-hour failure spikes, 49% dormancy risks &amp; expiry volume surges. Built using Python, DAX &amp; Star Schema.

# Broking Revenue & Client Intelligence — Power BI

An end-to-end Power BI project that looks at a discount stock-broking platform from two angles: **where the revenue really comes from** and **where the platform leaks it** (dormant clients and failed trades at market open).

Built independently as a portfolio project on a **fully synthetic dataset**. It is modeled on patterns common to Indian discount brokers and is not affiliated with any company.

**Workflow:** Python (synthetic data) → Power Query (cleaning) → Star-schema model → DAX → 2-page dashboard + drillthrough + tooltip

---

## Business Problem

Discount brokers earn a small flat fee per executed order. Account opening is free, so many clients sign up, place a few trades and go quiet. Meanwhile most revenue comes from a narrow set of products, and orders fail at the busiest time of the day.

Two questions this project answers:

1. Which client segments and products drive revenue, and how much revenue is lost to dormant clients?
2. When and why do orders fail, and how does trading behaviour change on F&O expiry days?

---

## Dataset (synthetic)

Generated with Python (Pandas, NumPy, Faker). No real user or market data and no personal information. Script: `scripts/generate_broking_dataset.py`.

| Table | Rows | Notes |
|---|---:|---|
| Dim_Customer | 10,000 | city, city tier, acquisition channel, age band, KYC status |
| Dim_Product | 15 | Equity Delivery, Intraday, F&O Futures/Options, Commodity, Currency, Mutual Fund |
| Date Table | 547 days | Jan 2024 – Jun 2025 |
| Fact_Trades | 209K (after cleaning) | one row per order, with status and failure reason |

**Behaviours built into the data** (so the analysis has something real to find):
- Flat per-order brokerage: ₹20 for Intraday, F&O, Commodity and Currency; ₹0 for Equity Delivery and Mutual Funds
- Higher F&O Options activity on expiry days (Tuesday/Thursday, simplified)
- Failures concentrated in the 9:15–10:00 AM market-open window
- A mix of client types: one-time traders who go dormant, casual investors, regular traders and heavy F&O traders

---

## Data Cleaning (Power Query)

The raw data was generated with deliberate flaws to practise ETL:

| Issue | Scale | Fix |
|---|---|---|
| Duplicate `trade_id` rows | 1,044 rows (0.5%) | Profiled and counted first, then removed |
| City names with wrong case and stray spaces | ~17.5% of customers | `Text.Trim` + `Text.Proper` (distinct cities collapsed to the true 16) |
| Data types | all tables | Dates, decimals and booleans validated |

---

## Data Model

Star schema with one fact table and three dimensions. All relationships are one-to-many with single-direction filtering. Foreign keys are hidden in report view.

```
Dim_Customer ──┐
Dim_Product  ──┼──< Fact_Trades
Date Table   ──┘
```

The date table was generated in Python and imported, then marked as the model's Date Table. Revenue stays as a column on `Fact_Trades` because it is a per-order value, so a separate revenue fact table would add nothing.

---

## Key DAX Logic

- **Active vs Dormant:** a client is *Dormant* if their last trade was 60 or more days before the latest date in the data (calculated per client using `ALLEXCEPT`, `ALL` and `DATEDIFF`).
- **Active Clients** and **Revenue per Active Client** use only clients with Active status:

```dax
Active Clients =
CALCULATE(DISTINCTCOUNT(Fact_Trades[customer_id]), Dim_Customer[Client Status] = "Active")

Revenue per Active Client =
DIVIDE(CALCULATE([Total Revenue], Dim_Customer[Client Status] = "Active"), [Active Clients])
```

- **Expiry Day Volume Multiplier** compares average F&O Options trades per *trading* day. Weekends are excluded, otherwise the ratio is inflated (an early version showed 3.47 and was corrected to 2.09):

```dax
Expiry Day Volume Multiplier =
VAR ExpiryDays = CALCULATE(DISTINCTCOUNT('Date Table'[date]), 'Date Table'[is_expiry_day] = TRUE)
VAR NonExpiryMarketDays = CALCULATE(DISTINCTCOUNT('Date Table'[date]),
    'Date Table'[is_expiry_day] = FALSE, 'Date Table'[is_market_day] = TRUE)
RETURN DIVIDE(
    DIVIDE([F&O Options Trades on Expiry Days], ExpiryDays),
    DIVIDE([F&O Options Trades on Non-Expiry Days], NonExpiryMarketDays))
```

Other measures: Total Revenue, Total Trades, Transaction Failure Rate, Revenue MoM %, Dormancy Rate, Peak Hour Failure Rate, Revenue Rank by Customer.

---

## Dashboard

| Page | What it shows |
|---|---|
| **Executive Overview** | Total Revenue, Total Trades, Active Clients, Failure Rate, MoM growth; revenue trend; revenue by segment; active vs dormant revenue share; top 5 customers. Slicers: Month-Year, Segment, City Tier, Client Status |
| **Client & Market Risk Intelligence** | Dormancy Rate, Revenue per Active Client, Expiry Day Multiplier, Peak Hour Failure Rate; revenue per client by channel; failures by reason and by hour; expiry vs non-expiry trading |
| **Customer Drillthrough** (hidden) | Right-click a customer in the Top 5 table to see that client's profile and full trade history |
| **Segment Tooltip** (hidden) | Hover a segment bar to see its monthly revenue trend |

Screenshots: `screenshots/` (add images here)

![Executive Overview](https://github.com/mahesh735-ai/Broking-Revenue-Client-Intelligence-Analytics/blob/main/Dashboard%201.png)
![Client and Market Risk Intelligence](https://github.com/mahesh735-ai/Broking-Revenue-Client-Intelligence-Analytics/blob/main/Dashboard%202.png)

---

## Key Insights

| Finding | Number |
|---|---|
| Revenue is concentrated in derivatives | F&O Options + Futures = ₹1.75M of ₹2.75M (about 64%) |
| Delivery and Mutual Funds earn nothing by design | ₹0 brokerage on both |
| Dormant clients are half the base but almost no revenue | 49.38% dormant, 1.65% of revenue; ≈₹9 per dormant client vs ₹533.51 per active client |
| Failures spike at market open | 8.87% in the 9:15–10:00 window vs 4.03% overall (about 1% for the rest of the day) |
| Server timeouts are the biggest cause | 4.2K of 8.4K failed trades (50%); margin shortfall 3.0K (36%); exchange connectivity 1.2K (14%) |
| Expiry days are far busier | 234 vs 112 F&O Options trades per day = 2.09x |
| Channel quality differs | App Store clients earn ₹496.73 per active client vs ₹562.62 for Referral (about 12% lower) |

---

## Recommendations

1. **Load-test and add capacity for the 9:15–10:00 window**, since server timeouts are half of all failures.
2. **Show a margin check before the order is placed** to cut margin-shortfall rejections.
3. **Re-engage clients before they cross 60 days of inactivity.** Dormant clients are about half the base, and delivery earns ₹0, so nudging a first paid product (for example intraday) has the most upside.
4. **Plan extra capacity on Tuesdays and Thursdays**, when F&O Options volume is about twice normal.
5. **Review App Store acquisition quality**, since those clients earn the least per active client.

---

## Assumptions and Limitations

- All numbers come from synthetic data, so the findings show the analysis method rather than real market facts.
- Brokerage is a simplified flat fee per order.
- The 60-day dormancy threshold is an assumption, and expiry days are simplified to Tuesday/Thursday.
- Failed orders are not retried in the data, so revenue lost to failures is not estimated.

---

## Repository Structure

```
Broking-Revenue-Client-Intelligence/
├── README.md
├── 01_BRD.md
├── Broking_Revenue_Client_Intelligence.pbix
├── data/            Dim_Customer.csv, Dim_Product.csv, Dim_Date.csv, Fact_Trades.csv
├── scripts/         generate_broking_dataset.py
├── screenshots/
└── docs/            project documentation, interview prep
```

To explore: open the `.pbix` in Power BI Desktop. If asked, point the data source to the `data/` folder.

---

## Tools

Python (Pandas, NumPy, Faker) · Power Query (M) · Power BI Desktop (data modeling, DAX, drillthrough, tooltip pages) · Custom background and KPI icons made with AI image tools

---

## Author

**Mahesh Thakare** — Final-year B.Tech CSE | Aspiring Data Analyst
[LinkedIn](https://www.linkedin.com/in/mahesh-thakare-75817b2a7) · [GitHub](https://github.com/mahesh735-ai)
