# LendingClub Loan Score Engine

End-to-end data engineering pipeline built on PySpark and Hive that processes 2.26 million loan records and assigns each borrower a credit risk score.

The dataset can be downloaded at https://www.kaggle.com/datasets/panchammahto/lendingclub-dataset-full-2007-to-2018.
The dataset covers LendingClub loans from 2007 to 2018. The goal was to go beyond simple EDA and actually build something useful — a scoring model that weights payment history, default behaviour, and financial health to produce a final loan grade (A through F).

---

## Pipeline overview

```
Raw CSV (2.26M rows)
    │
    ├── Customers    → nulls, dedup, emp_length cleanup
    ├── Loans        → loan term, purpose standardisation
    ├── Repayments   → date fixes, total_payment correction
    └── Defaulters   → delinquency, public records, enquiries
    │
    ▼
Hive external tables  →  bad data isolation (duplicate member_ids)
    │
    ├── Payment history  (20%)   last_payment vs installment
    ├── Default history  (45%)   delinq, pub_rec, bankruptcies, enquiries
    └── Financial health (35%)   loan status, home ownership, credit limit, grade
    │
    ▼
loan_score_final  →  weighted score  →  grade A–F
```

---

## Notebooks

| Notebook | What it does |
|---|---|
| `Lending_Club_ingestion` | Reads raw CSV, generates `member_id` via SHA2 hash, splits into source tables |
| `LC_Customers` | Cleans customer data — nulls, deduplication, employment length |
| `LC_Loans` | Cleans loan data — term conversion, purpose standardisation |
| `LC_Loan_repayments` | Fixes payment totals and date columns |
| `LC_Loan_defaulters` | Cleans delinquency and public records data |
| `LC_processing` | Creates Hive external tables, identifies and isolates bad member_ids |
| `LC_score_calculation` | Calculates loan score across three weighted criteria |

---

## Scoring model

The score is built in three stages, each contributing a weighted percentage to the final number.

**Payment history — 20%**  
Compares `last_payment_amount` against the borrower's monthly installment. Someone consistently paying above their installment scores excellent (800 pts), below half scores very bad (100 pts).

**Defaulter history — 45%**  
The heaviest weight, because past default behaviour is the strongest predictor of future risk. Looks at delinquencies in the last 2 years, public records, bankruptcies, and credit enquiries in the last 6 months.

**Financial health — 35%**  
Assesses current loan status (fully paid / current / late / charged off), home ownership type, how much of the borrower's total credit limit the loan represents, and their original Lending Club grade with sub-grade precision (A1 through G).

Scoring thresholds:

| Grade | Score range |
|---|---|
| A | > 2500 |
| B | 2000 – 2500 |
| C | 1500 – 2000 |
| D | 1000 – 1500 |
| E | 750 – 1000 |
| F | ≤ 750 |

---

## Stack

- PySpark 3.x on YARN
- Hive (external tables, parquet storage)
- HDFS for raw / cleaned / processed data layers
- Jupyter notebooks

---

## Data layers

```
lendingclub/
├── raw/              source CSVs split by entity
├── cleaned/          parquet after cleaning
├── cleaned_new/      parquet after bad data removal
├── bad/              duplicate member_ids
└── processed/        final loan scores
```
---
## REsults

- Back-tested against 687K historical loans with known outcomes — confirmed monotonic default rate increase from grade A to grade F  with grade D at 77.7%, validating the model's risk stratification logic.
---

## Known limitations

- `member_id` is a SHA2 hash of customer attributes, not a true unique key — identical customer profiles across different loans produce the same hash, which is how duplicates are detected and isolated
- Scoring thresholds were set based on domain intuition rather than statistical calibration against actual default rates
- The pipeline is built for batch processing; there's no streaming or incremental load support
