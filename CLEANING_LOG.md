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
- **Checked:** Missingness-vs-outcome test via inner join on `customer_id`, cross-checked two ways (pandas merge, then SQL join).
- **Correction note:** An initial in-memory pandas merge during cleaning appeared to show 0 transactions for customers with missing `state`, leading to a mistaken "untestable" conclusion. Re-verified later by reloading `customers_clean.csv`/`transactions_clean.csv` fresh from disk and re-running both the merge and an equivalent SQL join — both confirmed 447 transactions actually belong to customers with missing `state` (not 0). The original in-memory result was stale, likely due to notebook cells being re-run out of order during cleaning. Lesson: reload from saved files to verify a finding before treating an in-memory result as final.
- **Found (corrected):** Non-missing-state transactions: 971 bad / 7,752 total = 12.52%. Missing-state transactions: 66 bad / 447 total = 14.77%. Raw gap looks larger than the `age` result, but a chi-square test of independence returned **p = 0.19** — not statistically significant (threshold: p < 0.05). Cannot rule out random chance given the smaller missing-state group size (447 transactions).
- **Action:** Filled with `'Unknown'` for completeness. Despite the corrected comparison being testable after all, the result does not support treating `state` missingness as a driver of cancellations.
- **Verified:** `state.isnull().sum()` = 0 after fill.

## 6. `signup_date` and `transaction_date` (datatype conversion)

- **Action:** Converted both columns from string/object to `datetime64[ns]` using `pd.to_datetime()`.
- **Verified:** Null count after conversion = 0 for both (no unparseable dates), and `.dtype` confirmed as `datetime64[ns]` for both columns (not left as `object`).

---

## 7. Statistical significance testing (Phase 4 follow-up)

The Phase 3 comparisons above were based on eyeballing percentage gaps between groups. In Phase 4, each candidate driver was re-tested with a **chi-square test of independence** (`scipy.stats.chi2_contingency`) against `transaction_status`, to check whether the observed gaps are real patterns or plausibly explained by random chance given the group sizes involved. Threshold used: p < 0.05 = statistically significant.

| Feature | Raw gap (eyeballed) | Chi-square p-value | Verdict |
|---|---|---|---|
| `payment_method` | ~1.55 pts (11.73%–13.28% across 5 methods) | 0.62 | Not significant |
| `discount_missing_flag` | 1.78 pts (14.34% vs 12.56%) | 0.32 | Not significant |
| `age_missing` | <0.5 pts | 0.86 | Not significant |
| `state_missing` | 2.25 pts (14.77% vs 12.52%) | 0.19 | Not significant |
| `subscribe` | ~0.4 pts (12.77% vs 12.60%) | 0.86 | Not significant |
| `state` (actual province values, `Unknown` excluded) | 10.65 pts (7.26%–17.91% across 34 provinces) | **0.0087** (re-tested excluding `Unknown`: **0.0091**) | **Significant** |

**Key finding, revised:** Five of six tested candidates showed no statistically significant relationship to cancellation/refund rate — every gap that looked notable when eyeballing raw percentages turned out to be within the range explainable by random chance once sample sizes were properly accounted for. **The one exception is `state` (actual province value):** cancellation/refund rate varies meaningfully across Indonesia's provinces (p=0.009), and this result held up even after excluding the `Unknown`/missing-state placeholder group, confirming it isn't an artifact of the earlier `state_missing` test (which itself was not significant, p=0.19 — a different, narrower question).

**Important caveats on the `state` finding, to carry into the executive summary:**
- The overall chi-square test confirms provinces differ *as a group* — it does **not** identify which specific province(s) are reliably different from one another. With 34 groups compared, some individual differences in the sorted list (e.g., the highest vs. lowest province) may still reflect random noise even though the overall pattern is real. A rigorous next step would be pairwise testing with a multiple-comparisons correction (e.g., Bonferroni), which is outside this project's current scope.
- The dataset does not explain *why* province relates to cancellation rate. Plausible hypotheses (regional shipping/logistics differences, payment method adoption varying by region, local economic conditions) are not confirmed by this data and should be presented as directions for further investigation, not established causes.

---

## Summary table

| Column | Missing | Raw gap correlated with outcome? | Chi-square significant? | Treatment |
|---|---|---|---|---|
| `discount_applied` | 426 (5.2%) | Yes, eyeballed (+2.7 pts Cancelled) | **No** (p=0.32) | Flag added + filled with 0 |
| `age` | 56 (5.5%) | No (<0.5 pt) | No (p=0.86) | Median fill (42) |
| `review_text` | 4,915 (60%) | No (<0.6 pt) | Not tested (raw gap already negligible) | Flag added anyway + filled with `''` |
| `state` (missing/Unknown) | 53 (5.2%) | Yes, eyeballed (+2.25 pts) — initially miscategorized as untestable, corrected after stale in-memory result was caught | **No** (p=0.19) | Filled with `'Unknown'` |
| `state` (actual province value, excl. Unknown) | n/a | Yes, eyeballed (10.65 pt spread across 34 provinces) | **Yes** (p=0.009) — the only significant finding of the whole analysis | n/a — tested as a candidate driver |
| `payment_method` | n/a (no missing values) | Yes, eyeballed (~1.55 pt spread) | **No** (p=0.62) | n/a — tested as a candidate driver, not a missingness column |
| `subscribe` | n/a (no missing values) | Yes, eyeballed (~0.4 pt spread) | **No** (p=0.86) | n/a — tested as a candidate driver |
| `customer_id` (customers) | 10 duplicate rows | — | — | Dropped (confirmed full-row duplicates) |
| `signup_date`, `transaction_date` | 0 parse failures | — | — | Converted to `datetime64[ns]` |

## Files produced

- `customers_clean.csv`
- `transactions_clean.csv`
