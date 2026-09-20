 E-Commerce Cancellation & Refund Driver Analysis

**Business question:** What's driving the platform's ~12.65% cancellation/refund rate, and how much revenue is it costing?

**Bottom line:** Out of six candidate drivers tested with proper chi-square significance testing, only **province (state)** showed a statistically significant relationship (p=0.009). Payment method, discounts, age, missing-state status, and subscription status all failed significance, despite some looking meaningful when eyeballed as raw percentages. Two unexplained transaction-volume shifts were also found and flagged (Oct 2024 jump, Apr 2025 dip). Full reasoning in `EXECUTIVE_SUMMARY.md`.

## Project structure
```
project/
├── README.md                        <- you are here
├── EXECUTIVE_SUMMARY.md             <- findings + recommendation (start here for results)
├── CLEANING_LOG.md                  <- every cleaning decision, with reasoning, plus full
│                                        chi-square significance testing results
├── notebooks/
│   ├── 01_explore.ipynb             <- initial exploration of raw data
│   ├── 02_clean.ipynb               <- cleaning steps, one decision per section
│   └── 03_analysis.ipynb            <- chi-square tests, revenue trends, charts
├── data/
│   ├── customers_clean.csv
│   └── transactions_clean.csv
├── charts/
│   ├── 01_discount_vs_cancel_rate.png
│   ├── 04_state_cancel_rate.png       <- the one significant finding
│   ├── 05_monthly_lost_revenue.png
│   ├── 06_monthly_volume.png          <- the two volume anomalies
│   └── 07_significance_summary.png    <- all 6 tests, one chart
└── ecommerce_clean.db                <- SQLite DB with cleaned tables loaded
```

## Data source
Raw data: `customers.csv` (1,010 rows) and `transactions.csv` (8,199 rows), joined on `customer_id`. Source is a Kaggle "messy e-commerce" practice dataset with intentionally injected data quality issues (missing values, duplicates, inconsistent fields).

## Workflow
1. **Explore** — loaded raw CSVs into SQLite, profiled nulls/duplicates/dtypes/value ranges before touching anything.
2. **Clean** — handled missing values in `age`, `discount_applied`, `state`, and `review_text`, each decision backed by first checking whether the missingness itself correlated with the outcome (`transaction_status`) rather than defaulting to the same fill strategy for every column. Dropped 10 duplicate customer rows. Converted date columns to proper datetime types. Full reasoning, including one corrected mistake (an initial "untestable" conclusion on `state` that turned out to be based on stale in-memory data), is in `CLEANING_LOG.md`.
3. **Analyze** — tested six candidate drivers of the cancellation/refund rate using chi-square tests of independence, not just eyeballed percentage gaps. This caught two cases (payment method, discount missingness) where a raw percentage gap looked meaningful but wasn't statistically significant, and confirmed one real finding (province).
4. **Visualize** — five charts, including one summary chart showing all six test results together so the full methodology is visible at a glance, not just the one positive finding.
5. **Document** — `EXECUTIVE_SUMMARY.md` states findings with explicit confidence levels, what was ruled out, and named limitations (no product/category field exists in this data; the province finding doesn't identify which specific provinces differ without further pairwise testing).

## How to reproduce
```bash
pip install pandas matplotlib scipy
jupyter lab
# run notebooks in order: 01_explore -> 02_clean -> 03_analysis
```

## Honest limitations
- No product/category field in the source data — product IDs are ~unique per row, so "which product drives returns" cannot be answered with this dataset.
- The province finding is statistically significant overall but does not identify which specific provinces differ from one another without additional pairwise testing (outside this project's scope).
- The April 2025 volume dip is plausibly explained by Indonesia's Lebaran/Eid holiday timing, but this is an untested hypothesis based on external context, not confirmed within the transaction data.
- The October 2024 volume jump remains unexplained.
