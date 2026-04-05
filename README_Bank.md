# BriefBank — Banking Transaction Data Pipeline

A end-to-end data preparation pipeline for banking transaction data, covering ingestion, cleaning, anomaly detection, feature engineering, and export.

---

## Project Structure

```
briefbank/
├── bank.csv                  # Raw transaction dataset (source)
├── financecore_clean.csv     # Cleaned, enriched output dataset
└── breif_bank.ipynb          # Main pipeline notebook
```

---

## Dataset Overview

### Raw Data — `bank.csv`

| Property | Value |
|---|---|
| Rows | 2,060 |
| Columns | 16 |
| Duplicates (by `transaction_id`) | 60 |

**Columns:**

| Column | Type | Description |
|---|---|---|
| `transaction_id` | string | Unique transaction identifier |
| `client_id` | string | Client identifier |
| `date_transaction` | string → datetime | Transaction date |
| `montant` | string → float | Transaction amount (original currency) |
| `devise` | string | Currency code |
| `taux_change_eur` | float | Exchange rate to EUR |
| `montant_eur` | float | Amount in EUR |
| `categorie` | string | Transaction category |
| `produit` | string | Banking product |
| `agence` | string | Branch name (64 missing) |
| `type_operation` | string | Operation type |
| `statut` | string | Transaction status |
| `score_credit_client` | float | Client credit score (167 missing) |
| `segment_client` | string | Client segment (105 missing) |
| `solde_avant` | string → float | Balance before transaction |
| `taux_interet` | float | Interest rate (entirely empty) |

### Clean Data — `financecore_clean.csv`

The cleaned dataset adds 9 new columns on top of the original 16:

| New Column | Description |
|---|---|
| `is_anomaly` | Boolean flag — IQR outlier on `montant` or invalid credit score |
| `annee` | Year extracted from `date_transaction` |
| `mois` | Month extracted from `date_transaction` |
| `trimestre` | Quarter extracted from `date_transaction` |
| `jour_semaine` | Day of week name |
| `montant_eur_verifie` | Cross-check: `montant / taux_change_eur` |
| `categorie_risque` | Risk tier: `Low` (≥700) / `Medium` (≥580) / `High` (<580) |
| `nb_transactions` | Number of transactions per client |
| `montant_moyen` | Average EUR amount per client |
| `nb_produits` | Number of distinct products per client |

---

## Pipeline Steps

### 1. Importation & Exploration
Load `bank.csv` and audit shape, dtypes, missing values, and duplicate transaction IDs.

### 2. Data Cleaning
- Remove 60 duplicate transactions (keyed on `transaction_id`)
- Parse `date_transaction` to datetime
- Normalize `montant` (comma → dot) and cast to float
- Strip ` EUR` suffix from `solde_avant` and cast to float
- Uppercase `devise`, title-case `segment_client`, strip whitespace from `agence`
- Impute missing `score_credit_client` with the column median
- Impute missing `segment_client` with the mode
- Fill missing `agence` with `"Inconnue"`

### 3. Anomaly Detection
Flag transactions where `montant` falls outside `[Q1 − 1.5×IQR, Q3 + 1.5×IQR]`, or where `score_credit_client` is outside `[0, 850]`. **112 anomalies detected.**

### 4. Feature Engineering
- Extract temporal features from `date_transaction`
- Compute `montant_eur_verifie` as an independent EUR cross-check
- Assign credit risk tier via `categorie_risque`
- Aggregate per-client statistics (`nb_transactions`, `montant_moyen`, `nb_produits`) and merge back onto the transaction table

### 5. Export
Write the enriched DataFrame to `financecore_clean.csv` (no index).

---

## Quick Start

```bash
pip install pandas seaborn
jupyter notebook breif_bank.ipynb
```

The notebook is self-contained. Run cells top to bottom; `financecore_clean.csv` is generated automatically in the last cell.

---

## Notes

- `taux_interet` is entirely null in the raw data and is preserved as-is for future use.
- Anomalies are **flagged**, not removed — downstream users decide how to handle them.
- All monetary cross-checks use `montant / taux_change_eur`; small floating-point discrepancies vs. `montant_eur` are expected.
