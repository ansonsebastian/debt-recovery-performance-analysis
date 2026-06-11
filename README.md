# Debt Recovery Performance Analysis

**An end-to-end data pipeline and executive dashboard that identifies what drives loan default — and where a collections team should focus its recovery effort.**

Built on Azure Databricks (PySpark + SQL) using a Bronze → Silver → Gold medallion architecture, with Delta tables, and visualised in a two-page Power BI executive dashboard.

---

## Business problem

A lender's collections team has limited resources and a large book of loans. The question this project answers is practical: **which loans are most likely to default, where are the recoverable dollars concentrated, and where should collectors be pointed first?**

Rather than report every possible metric, the analysis is built to support a single decision — how to prioritise recovery effort — and is honest about which parts of the data can and cannot be trusted.

## Dataset

Kaggle **Loan Default** dataset — **148,670 loan records across 34 columns**, covering:

- **Borrower attributes:** income, credit score, region, gender, age
- **Loan terms:** loan amount, interest rate, loan-to-value ratio, term, property value
- **Risk ratios:** debt-to-income ratio
- **Target:** `Status` (1 = defaulted, 0 = repaid)

## Architecture

The project follows a **medallion architecture**, the standard layered approach for building reliable analytics on a lakehouse:

| Layer | Purpose | Output |
|-------|---------|--------|
| **Bronze** | Raw ingestion of the source file, untouched | Raw Delta table |
| **Silver** | Cleaned, typed, enriched, one row per loan | `loans_silver` (148,670 rows, 0 nulls) |
| **Gold** | Business-ready aggregates feeding the dashboard | 3 Gold tables |

## Pipeline

**Bronze → Silver (PySpark):**
- Profiled nulls across all 34 columns
- Imputed missing numeric values with the column median, categorical values with the mode
- Engineered three derived columns: `risk_tier`, `ltv_bucket`, `dti_bucket`
- Result: 148,670 rows, 37 columns, zero nulls

**Silver → Gold (SQL):**
- `gold_regional_breakdown` — loans, defaults, default rate and value-at-risk per region
- `gold_risk_summary` — default rate by region and debt-to-income band
- `gold_default_by_segment` — default rate by credit band and debt-to-income band

## Key findings

1. **Portfolio default rate is 24.64%** — roughly one loan in four.

2. **Debt-to-income above 30% is the single strongest predictor.** In every region, borrowers above the 30% debt-to-income mark default at a materially higher rate (24–32%) than those below it (15–20%). The relationship is a threshold effect, not a smooth gradient.

3. **Region is a secondary signal — and rate is not the same as exposure.** North-East has the highest default *rate* (~30%) but holds under 1% of the book. The recoverable *dollars* sit in South and North (~$5bn each of ~$12bn total at risk). **Effort should follow exposure, not rate.**

4. **Credit score has no predictive power in this portfolio.** Low, Medium, and High credit bands all default within a fraction of a percent of each other (~24%). This is reported deliberately — a tested-and-rejected variable is as useful to know as a confirmed one.

5. **Two variables were found to be unreliable and handled accordingly** (see below).

## Data quality — and why it matters

Two diagnostics surfaced imputation artifacts that would otherwise have produced misleading conclusions:

- **Loan-to-value ratio** was excluded. A frequency check showed ~10% of all records (15,203 rows) shared one identical imputed value, creating a false cluster that made any loan-to-value segmentation untrustworthy.
- **Debt-to-income "Over 30" rates are reported as directional, not exact.** About one in five records (28,661 rows, ~19%) had a missing debt-to-income value imputed to the same figure (~39), which sits in the "Over 30" band and slightly inflates its rates. The threshold *direction* is robust across all regions; the exact magnitudes are treated as indicative.

Catching and documenting these — rather than charting them blindly — is a core part of the analysis, not an afterthought.

## Dashboard

A two-page Power BI executive dashboard:

- **Page 1 — Executive Summary:** headline KPIs (total loans, value at risk, overall default rate), a combined dollars-at-risk-vs-default-rate chart by region, and a recommendations panel.
- **Page 2 — Risk Drivers:** debt-to-income threshold heat grid by region, the credit-score null result, and the data-quality notes.

*(Dashboard screenshots in `/images`.)*

## Recommendations delivered

1. **Prioritise South and North** — they hold the large majority of dollars at risk, despite mid-range default rates.
2. **Triage on the 30% debt-to-income threshold** — the strongest single predictor.
3. **Do not triage on credit score** — no predictive power in this portfolio.
4. **Monitor North-East, but do not over-resource it** — worst rate, smallest exposure.

## Limitations and next steps

- Several columns are heavily imputed and were intentionally not segmented on. A production version would add an `is_imputed` flag during cleaning to separate genuine from filled values.
- Future exploration with the most value: loan-characteristic cuts (purpose, type, occupancy, risky loan features), a value-at-risk concentration (Pareto) view, and a composite risk-targeting segment with measured lift.

## Tech stack

Azure Databricks · PySpark · Spark SQL · Delta Lake · Power BI · GitHub

## Repository structure

```
.
├── notebooks/      # PySpark: Bronze -> Silver cleaning, Silver -> Gold aggregation
├── sql/            # Analysis queries (commented by business question)
├── powerbi/        # .pbix dashboard + theme JSON
├── images/         # Dashboard screenshots
└── README.md       # This report
```
