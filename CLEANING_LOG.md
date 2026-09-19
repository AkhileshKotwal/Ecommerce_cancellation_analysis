# Cleaning Log

Documents every cleaning decision made on `customers.csv` and `transactions.csv`, and the reasoning behind each one. Where relevant, missingness was tested against `transaction_status` before deciding how to handle it — not just filled by default.

---

## 1. `customer_id` duplicates (customers table)

- **Checked:** `df_customers.duplicated().sum()` (full-row duplicates) and `df_customers.duplicated(subset='customer_id').sum()` (duplicate ID regardless of other columns).
- **Found:** 10 duplicate rows — full-row duplicates and duplicate-ID counts matched, confirming these were true copies, not ID reuse with conflicting data.
- **Action:** Dropped with `drop_duplicates()`.
- **Verified:** Post-drop, both duplicate checks return 0.

## 2. `discount_applied` (transactions table)

- **Missing:** 426 rows (5.2%).
- **Checked:** Compared `transaction_status` distribution between missing and non-missing groups.
- **Found:** Missingness is **not random**. Missing-discount transactions have a Cancelled rate of 10.6% vs. 7.9% for non-missing — a 2.7-point gap, the largest difference in the comparison. Overall bad-rate (Cancelled+Refunded) is 14.34% for missing vs. 12.56% for non-missing.
- **Caveat:** Group size for missing (426) is smaller than non-missing (7,773), so this is a real but moderate-confidence signal — worth investigating further, not treated as proven.
- **Action:** Added `discount_missing_flag` (boolean) to preserve the signal, then filled `discount_applied` with 0.
- **Verified:** `discount_missing_flag.sum()` = 426 (matches original null count); `discount_applied.isnull().sum()` = 0 after fill.

## 3. `age` (customers table)

- **Missing:** 56 rows.
- **Checked:** Merged customers with transactions (inner join, 8,199 rows — confirms referential integrity held), then compared `transaction_status` distribution between missing-age and non-missing-age groups.
- **Found:** No meaningful difference — all four status categories differ by less than 0.5 percentage points between groups. Missingness appears random with respect to the outcome.
- **Action:** Filled with median age (42). No flag needed, since missingness carries no signal.
- **Verified:** `age.isnull().sum()` = 0 after fill.

## 4. `review_text` (transactions table)

- **Missing:** 60% of rows (4,915 of 8,199) — expected, since most customers don't leave a review; this is normal behavior, not a data quality defect.
- **Hypothesis tested:** Customers with a cancelled/refunded order might be more likely to leave a review explaining the issue.
- **Checked:** Compared `transaction_status` distribution between `has_review = True` and `has_review = False`.
- **Found:** Hypothesis not supported — differences across all four statuses are under 0.6 percentage points. Leaving a review is not meaningfully associated with order outcome in this dataset.
- **Action:** Added `has_review` (boolean) flag anyway, for completeness/potential future use, despite the weak result. Filled `review_text` with `''` (empty string) rather than leaving as null, so string operations (e.g. `.str.contains()`) don't silently fail on `NaN`.
- **Verified:** `review_text.isnull().sum()` = 0 after fill.

## 5. `state` (customers table)

- **Missing:** 53 rows (post-dedup; was 54 before the 10 duplicate rows were dropped, consistent with one duplicate having a null state).
- **Checked:** Attempted the same missingness-vs-outcome test as above via inner join on `customer_id`.
- **Found:** **Untestable, not "no correlation."** All 53 customers with missing `state` have zero transactions (`merged['state_missing'].value_counts()` shows no `True` rows at all) — they don't appear in the transactions data, so there's nothing to compare against. This is a distinct finding from the `age` result and should not be conflated with "checked and found no signal."
- **Action:** Filled with `'Unknown'` for completeness. Since these customers have no transactions, this field has no bearing on the cancellation/refund analysis.
- **Verified:** `state.isnull().sum()` = 0 after fill.

## 6. `signup_date` and `transaction_date` (datatype conversion)

- **Action:** Converted both columns from string/object to `datetime64[ns]` using `pd.to_datetime()`.
- **Verified:** Null count after conversion = 0 for both (no unparseable dates), and `.dtype` confirmed as `datetime64[ns]` for both columns (not left as `object`).

---

## Summary table

| Column | Missing | Correlated with outcome? | Treatment |
|---|---|---|---|
| `discount_applied` | 426 (5.2%) | **Yes** — +2.7 pts Cancelled rate | Flag added + filled with 0 |
| `age` | 56 (5.5%) | No — <0.5 pt difference | Median fill (42) |
| `review_text` | 4,915 (60%) | No — <0.6 pt difference | Flag added anyway + filled with `''` |
| `state` | 53 (5.2%) | **Untestable** — affected customers have 0 transactions | Filled with `'Unknown'` |
| `customer_id` (customers) | 10 duplicate rows | — | Dropped (confirmed full-row duplicates) |
| `signup_date`, `transaction_date` | 0 parse failures | — | Converted to `datetime64[ns]` |

## Files produced

- `customers_clean.csv`
- `transactions_clean.csv`
