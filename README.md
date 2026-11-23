<img width="1313" height="737" alt="Screenshot 2026-11-21 085020" src="https://github.com/user-attachments/assets/90a8ae55-16f8-47d5-b11c-33b9a5c84ed5" />
# Credit Risk & Portfolio Monitoring

**Author:** Aryan Huria
**Stack:** MySQL · SQL (CTEs, window functions, CASE logic, conditional aggregation) · Power BI Desktop · DAX

---

## Overview

This project puts me in the shoes of a Data Analyst on a credit-card issuer's Risk & Portfolio Management team. The bank has six months of billing, payment, and repayment data on 30,000 customers and needs to know where default risk is concentrated so it can prioritize monitoring.

I intentionally kept this out of ML-prediction territory — scoring a brand-new applicant is a different problem than this dataset can credibly answer. Instead I ran it as an analyst would: profile the raw data, engineer behavioural features from the repayment history, test which behaviours actually track with default, build a segmentation the evidence supports, and size up the bank's exposure in each segment.

**Headline numbers:**
- 22.12% overall default rate (30,000 customers)
- Persistent delinquency (delinquent 3+ of 6 months) is by far the strongest signal — default rate rises from 11.71% at zero delinquent months to 70.32% at six
- An equal-weight risk score looked promising, then failed a deeper check and was scrapped
- Final segments: Lower Risk (15.19% default) → Watchlist (19.79%) → High Risk (49.85%) → Critical Risk (62.10%)
- **Central finding: the Watchlist, not Critical Risk, holds the largest share of outstanding exposure — 57.3%, ~880M**

![Credit Risk & Portfolio Monitoring Dashboard](Credit%20Risk%20%26%20Portfolio%20Monitoring%20Dashboard.png)

---

## The Business Question

> Which customer behaviours track with higher default risk, where is credit exposure concentrated, and which segments need the closest monitoring?

The Credit Risk Manager needs three things from this: **where** default is most common, **what behaviour** drives it, and **how much money** is riding on the riskier customers.

Language matters here — I say a behaviour is *associated with* default, never that it *causes* it, since this is observational data. Demographics (age, gender, education, marital status) stay purely descriptive; the segmentation is built entirely from repayment behaviour, not who the customer is.

---

## Data

**Source:** [UCI ML Repository — Default of Credit Card Clients](https://archive.ics.uci.edu/dataset/350/default%2Bof%2Bcredit%2Bcard%2Bclients)
30,000 Taiwanese credit-card customers, 25 columns, six months of history (Apr–Sep 2005), target = default next month (1/0). No missing values per UCI. CC BY 4.0 licensed.

---

## Repository Structure

```
credit-risk-portfolio-analysis/
├── README.md
├── Credit_Risk_Portfolio_Project_Documentation.docx  ← full write-up
├── credit_risk_analysis_full.sql                     ← complete SQL workflow
├── Credit-Risk-Portfolio-Monitoring.pbix
└── Credit Risk & Portfolio Monitoring Dashboard.png
```

---

## Workflow

| Phase | Tool | What happens |
|---|---|---|
| Import | MySQL | Raw UCI export loaded untouched |
| Profile | SQL | Understand the data before changing anything |
| Clean | SQL | Readable, analysis-ready table, nothing discarded silently |
| Analyze | SQL | Test which behaviours predict default; build the segmentation |
| Report | Power BI + DAX | Turn the SQL findings into a dashboard |

All business logic — segmentation rules, feature definitions, exposure math — lives in SQL, in one place I could test and validate. Power BI reads a single final view and never redefines that logic in DAX.

---

## Data Profiling

Raw data went into `credit_card_clients_raw` and stayed untouched as an audit trail. Profiling column-by-column turned up undocumented category codes in `EDUCATION` (0, 5, 6 — ~1.15% of customers) and `MARRIAGE` (code 0 — 54 customers). Rather than guess or delete, I grouped these as `Other/Unknown` and kept every row.

Numeric checks surfaced negative bill balances (customers who overpaid) and 3,931 customers (~13%) who exceeded their limit in at least one month — different from the 637 customers who averaged 100%+ utilization across all six months. Both are real account behaviour, not errors, so both stayed in. Zero duplicate IDs, zero NULLs across 25 columns.

## Cleaning

Built `credit_card_clients_clean` as a separate table (same audit-trail principle). Recoded `SEX`/`EDUCATION`/`MARRIAGE` into readable labels, and renamed the cryptic `PAY_0`, `BILL_AMT1`, `PAY_AMT1`-style columns to `pay_status_sep`, `bill_amount_sep`, `payment_amount_sep`, etc. Nothing deleted; every transformation is defensible.

## Behavioural Analysis

**Recent repayment status:** default rate jumps from 12.81% (no delinquency) to 33.95% (1 month late) to 69.14% (2 months late) — strong enough to check across the full 6-month window.

**Multi-month delinquency (strongest signal):**

| Delinquent months | Customers | Default rate |
|---:|---:|---:|
| 0 | 19,931 | 11.71% |
| 1 | 4,426 | 29.82% |
| 2 | 1,899 | 38.76% |
| 3 | 1,154 | 50.87% |
| 4 | 951 | 57.31% |
| 5 | 298 | 57.38% |
| 6 | 1,341 | 70.32% |

Nearly a straight line, ~6× spread end to end. This became the backbone metric for everything after.

**Utilization:** banded average 6-month utilization against default rate — real effect, but only ~2× spread (17.86% at low utilization vs 34.22% at 100%+). Combined with delinquency, delinquency dominates: high-utilization-but-no-delinquency customers default at just 13.38%.

**Payment ratio:** (payments ÷ billed amount) doesn't move in a straight line — risk falls sharply from 26.98% (very low payment) to 14.43% (moderate), then flattens. Used as a modifier, not a standalone score, and it's a behavioural proxy, not a literal "% of debt repaid."

## The Rejected Hypothesis

Combined delinquency, utilization, and low-payment into a 0–3 point score. Looked clean on the surface (14.11% → 61.72% as flags increased), but breaking it into its 8 individual combinations showed two customers scoring "1" could sit at 48.55% or 13.49% risk depending on *which* flag they had. Equal weighting was masking that gap, so I dropped the score and built the segmentation from the empirical break instead — delinquency present vs. absent, not a point system.

## Final Segmentation

```sql
CASE
    WHEN delinquent_months >= 3 AND payment_ratio_pct < 10 THEN 'Critical Risk'
    WHEN delinquent_months >= 3                            THEN 'High Risk'
    WHEN delinquent_months < 3 AND avg_utilization_pct >= 50 THEN 'Watchlist'
    ELSE 'Lower Risk'
END
```

| Segment | Customers | Default rate |
|---|---:|---:|
| Lower Risk | 17,509 | 15.19% |
| Watchlist | 8,747 | 19.79% |
| High Risk | 646 | 49.85% |
| Critical Risk | 3,098 | 62.10% |

Frozen once defined — no threshold-tuning for a prettier chart later.

## Exposure Analysis

Outstanding balance = positive September statement balance per customer.

| Segment | Outstanding balance | Avg / customer |
|---|---:|---:|
| Lower Risk | 466,646,833 | 26,651.83 |
| Watchlist | 880,412,963 | 100,653.13 |
| High Risk | 12,848,281 | 19,888.98 |
| Critical Risk | 177,473,180 | 57,286.37 |

**Portfolio total: 1,537,381,257 (~1.54bn).** The Watchlist carries the most money at risk despite a mid-range default rate — the project's main takeaway. A monitoring process ranked purely by default rate would miss this.

---

## Power BI Dashboard

Connects to MySQL in **Import mode**, reading only the final `vw_credit_risk_analysis` view — Power BI never re-derives business logic in DAX. A small `Risk Segments` table controls display order.

Core measures: `Total Customers`, `Default Rate`, `Total Outstanding Balance`, `Critical Risk Customers`, `Average Delinquent Months`, `Average Payment Ratio`, `Average Utilization`. Utilization and payment-ratio fields are stored as raw percentage points in SQL, so native Power BI percentage formatting is applied only where genuinely needed (to avoid a 100x-inflated display).

**Dashboard elements:** 5 KPI cards · exposure-vs-default combo chart · a bubble "risk map" (default rate × avg balance × total exposure) · segment/default stacked bars · a behavioural scorecard · slicers on real fields only (segment, gender, education, marital status, age, default status).

---

## Recommendations

1. **Track exposure alongside default probability, not instead of it** — Critical Risk has the scariest rate, but the Watchlist holds more money.
2. **Screen on persistent delinquency first** — it's the dominant driver everywhere in the data.
3. **Use utilization as a secondary filter**, not a standalone trigger — weak signal on its own.
4. **Treat low payment ratio as an amplifier**, not an independent flag — it only matters paired with delinquency.

---

## Challenges

- Undocumented repayment-status codes (`-2`, `0`) and category codes in EDUCATION/MARRIAGE → profiled first, documented assumptions, never guessed
- Equal-weight score looked clean but hid very different risk levels → broke it into components, rebuilt from the empirical break
- One exposure query drifted from frozen logic; another produced a ~2x-inflated total → corrected the logic, cross-verified every number against an independent calculation before trusting it
- Power BI's default percentage formatting would have double-multiplied already-percentage fields → kept as decimals, built dedicated measures

---

## Limitations

- Dataset reflects 2005 Taiwanese customers — not a current benchmark
- Segmentation is descriptive/rule-based for monitoring, not a production underwriting model
- No LGD/EAD data, so no expected-loss figure
- Undocumented codes were preserved, not assigned invented meanings

---

## Full Documentation

See [`Credit_Risk_Portfolio_Project_Documentation.docx`](./Credit_Risk_Portfolio_Project_Documentation.docx) for the complete methodology, and [`credit_risk_analysis_full.sql`](./credit_risk_analysis_full.sql) for every query used above.

---

## Skills Demonstrated

MySQL · Advanced SQL (CTEs, window functions, CASE segmentation) · Data profiling & cleaning · Behavioural feature engineering · Hypothesis testing · Risk segmentation · Exposure analysis · Power BI · DAX · Data storytelling

---

## About

**Aryan Huria**
Data Analyst | SQL · Power BI · Python · Excel 
