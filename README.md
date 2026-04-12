# 🏦 FinanceCore SA — Bank Data Pipeline

A data cleaning and feature engineering pipeline for banking transaction data.

## 📁 Project Structure

```
├── bank.csv                  # Raw input data
├── breif_bank.ipynb          # Main analysis notebook
├── financecore_clean.csv     # Cleaned output dataset
├── DECISIONS.md              # Data decisions log
└── README.md                 # This file
```

## 🔄 Pipeline Overview

### 1. Importation & Exploration
- Loads `bank.csv` and normalizes column names (lowercase, stripped)
- Prints data types, shape, and descriptive statistics
- Visualizes missing value rates per column
- Identifies duplicate `transaction_id` entries and measures time gaps between them

### 2. Data Cleaning
- Removes duplicates on `transaction_id`, keeping the first occurrence
- Parses `date_transaction` into datetime (day-first format)
- Fixes `montant`: replaces comma separators, converts to float
- Fixes `solde_avant`: strips ` EUR` suffix, converts to float
- Standardizes text fields: `devise` → uppercase, `segment_client` → capitalized, `agence` → stripped
- Imputes missing values:
  - `score_credit_client` → median
  - `agence`, `segment_client`, `date_transaction` → mode
- Drops the `taux_interet` column

### 3. Outlier Detection
- Applies IQR method to `montant` and `score_credit_client`
- Applies business rule: credit scores must be in range [0, 850]
- Flags anomalies in a new boolean column `is_anomalie`

### 4. Feature Engineering

| Feature | Description |
|---|---|
| `annee`, `mois`, `trimestre`, `semaine-jour` | Date components |
| `montant_eur_verifie` | Recalculated EUR amount via exchange rate |
| `categorie_risque` | Risk tier: Low (≥700) / Medium (≥580) / High (<580) |
| `total_credit`, `total_debit` | Per-client aggregated amounts |
| `solde_net` | Net balance per client (credit − debit) |
| `nb_transaction` | Transaction count per client |
| `montant_moyen` | Average transaction amount per client |
| `nb_produit` | Number of distinct products per client |
| `taux_rejet` | Rejection rate (%) per agency |

### 5. Export
- Saves cleaned dataset to `financecore_clean.csv`
- Writes documentation to `DECISIONS.md`

## ▶️ How to Run

```bash
pip install pandas matplotlib
jupyter notebook breif_bank.ipynb
```

## 📊 Key Columns

| Column | Description |
|---|---|
| `transaction_id` | Unique transaction identifier |
| `client_id` | Client identifier |
| `date_transaction` | Transaction timestamp |
| `montant` | Transaction amount |
| `montant_eur` | Amount in EUR |
| `taux_change_eur` | Exchange rate to EUR |
| `solde_avant` | Account balance before transaction |
| `type_operation` | Credit or Debit |
| `statut` | Transaction status (e.g., Rejete) |
| `devise` | Currency code |
| `segment_client` | Client segment |
| `agence` | Branch name |
| `produit` | Banking product |
| `score_credit_client` | Client credit score (0–850) |
