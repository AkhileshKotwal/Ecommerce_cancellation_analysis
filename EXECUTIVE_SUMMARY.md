# Executive Summary: Cancellation & Refund Drivers

**Business question:** What's driving the platform's cancellation/refund rate, and how much revenue does it cost?

## Headline finding
Out of six commonly-assumed drivers tested with a proper statistical significance test (chi-square, not just eyeballed percentages), **only one — the customer's province — showed a statistically significant relationship** with cancellation/refund rate (p = 0.009). Payment method, discount level (as a missing-data flag), customer age (as a missing-data flag), missing-state status, and subscription status all failed to reach significance, despite some of them showing raw percentage gaps that looked meaningful at first glance.

This matters: it means the intuitive assumptions a team might reach for first (payment friction, discount-seeking customers, non-subscribers being less committed) are **not supported by this data**, and resources aimed at those levers would likely be wasted.

## What was tested, and what survived
| Candidate driver | Raw gap (eyeballed) | Chi-square p-value | Verdict |
|---|---|---|---|
| **Province (state)** | 7.26%-17.91% across 34 provinces | **0.0087** | **Significant** |
| State missing/unknown | 14.77% vs 12.52% | 0.19 | Not significant |
| Discount missing flag | 14.34% vs 12.56% | 0.32 | Not significant |
| Payment method | 11.73%-13.28% across 5 methods | 0.62 | Not significant |
| Age missing flag | <0.5 pt gap | 0.86 | Not significant |
| Subscription status | 12.60% vs 12.77% | 0.86 | Not significant |

**Important caveat on the province finding:** the test confirms provinces differ *as a group* — it does not identify which specific province(s) are reliably different from each other, since comparing 34 groups carries a real risk that some of the spread is due to chance even though the overall pattern is real (the "multiple comparisons" problem). A rigorous next step would be pairwise testing with a correction (e.g., Bonferroni) — outside this analysis's current scope. The data also doesn't explain *why* province matters; plausible hypotheses (regional shipping/logistics infrastructure, payment method adoption varying by region, local economic conditions) are untested and should be framed as directions for further investigation, not established causes.

## Revenue impact
- Overall cancellation/refund rate: **12.65%** of all 8,199 transactions.
- Monthly revenue lost to cancellations/refunds has settled into a fairly stable **10-18%** band since transaction volume stabilized (Oct 2024 onward). Earlier months (Oct 2023-Sep 2024) showed a much wider, noisier range (0%-27%) — a predictable effect of very small transaction counts in those months, not a real trend, and should not be read as meaningful volatility.

## Two unexplained volume shifts, flagged rather than smoothed over
1. **October 2024:** Monthly transaction volume jumps from 103 to 789 — roughly 8x — in a single month, with no gradual ramp-up, and stays elevated afterward. This does not fit a typical seasonal/holiday pattern (which would spike and revert); it looks more like a platform-level change (marketing push, catalog expansion, or the point the business/data collection became consistent). Cause not confirmed by this dataset.
2. **April 2025:** Volume drops from 808 to 196 transactions, then recovers over subsequent months. This timing plausibly aligns with Indonesia's Lebaran/Eid al-Fitr holiday period (national holidays and "mudik" travel ran March 28-April 3, 2025), which could suppress e-commerce activity. This is a **plausible, untested hypothesis** based on external context, not something confirmed within the transaction data itself — a day-level breakdown of April would strengthen or weaken this explanation.

## Limitations
- No product/category field exists in this dataset (product IDs are effectively unique per transaction), so product-level questions are out of scope.
- The province finding, while statistically significant, doesn't establish causation or identify the specific province(s) responsible without further testing.
- Several fields (`review_text` at 60% missing) were checked and found unrelated to outcome but weren't included in the chi-square testing table above since the raw gap was already negligible.
- This dataset originates from a Kaggle practice dataset with intentionally-injected data quality issues, so findings should be read as a demonstration of methodology rather than a live business result.

## Recommendation
1. Investigate province-level differences further — start with a pairwise comparison (with a multiple-comparisons correction) to identify which specific provinces differ meaningfully, and look into regional shipping/logistics data if available, since that's the most plausible mechanism.
2. Do not pursue payment-method restrictions, discount-policy changes, or subscription-based interventions as cancellation-reduction levers — the data does not support any of them.
3. Investigate what changed operationally around October 2024 (the volume jump) — this is a larger, unexplained shift than anything found in the cancellation-driver analysis and may be more consequential to understand.
4. If day-level transaction data is available, check whether the April 2025 dip is concentrated around the Lebaran holiday dates specifically, to confirm or rule out that explanation.

**Confidence level:** High confidence in the five "not significant" findings, given consistent results across reasonably large, evenly-sized groups. Moderate-to-high confidence in the province finding — the overall test is solid, but which specific provinces differ and why remains unconfirmed. Low confidence in the Lebaran explanation for the April dip — plausible and well-timed, but not verified against the data itself.
