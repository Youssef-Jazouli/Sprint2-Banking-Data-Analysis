# DECISIONS.md — FinanceCore SA

Documentation of all data processing decisions made during the pipeline.

---

## 🔁 Duplicates

- **Detection:** Rows sharing the same `transaction_id` were identified.
- **Decision:** Kept the **first occurrence**, removed all subsequent duplicates.
- **Rationale:** The first recorded entry is assumed to be the original transaction; later entries are likely re-submissions or system errors.

---

## 📅 Dates

- **Issue:** `date_transaction` contained inconsistent or unparseable formats.
- **Decision:** Converted to unified `datetime` format `YYYY-MM-DD HH:MM:SS` using `dayfirst=True`.
- **Missing dates:** Imputed using the **mode** (most frequent date).
- **Rationale:** Mode imputation is preferred over dropping rows to preserve transaction history.

---

## 💰 Amounts (`montant`)

- **Issue:** Some values used a comma (`,`) as the decimal separator.
- **Decision:** Replaced commas with periods and cast to `float`.
- **Invalid values:** Coerced to `NaN` using `errors='coerce'`.
- **Rationale:** Standardized numeric format required for all calculations.

---

## 💶 Balance (`solde_avant`)

- **Issue:** Values contained the suffix ` EUR` as a string.
- **Decision:** Stripped the ` EUR` text and converted to `float`.
- **Rationale:** Column must be numeric for any financial computation.

---

## 🔤 Text Fields

| Column | Transformation |
|---|---|
| `devise` | Uppercased and stripped |
| `segment_client` | Capitalized (first letter) |
| `agence` | Whitespace stripped |

- **Rationale:** Ensures consistent grouping and joins; prevents duplicates due to casing or spacing.

---

## 🩹 Missing Values

| Column | Method | Rationale |
|---|---|---|
| `score_credit_client` | **Median** | Robust to outliers in a skewed score distribution |
| `agence` | **Mode** | Most frequent branch is a safe default |
| `segment_client` | **Mode** | Most common segment is a safe default |
| `date_transaction` | **Mode** | Preserves rows while filling with the most typical date |

---

## 🗑️ Dropped Columns

- **`taux_interet`:** Dropped entirely.
- **Rationale:** Column had excessive missing values or was deemed irrelevant for downstream analysis.

---

## ⚠️ Outlier Detection

Three conditions flag a row as anomalous (`is_anomalie = True`):

1. **`montant` IQR outlier:** Value falls below `Q1 − 1.5×IQR` or above `Q3 + 1.5×IQR`.
2. **`score_credit_client` IQR outlier:** Same IQR rule applied to credit scores.
3. **`score_credit_client` business rule:** Score outside the valid range `[0, 850]`.

- **Decision:** Anomalies are **flagged, not removed**, to allow downstream teams to decide how to handle them.
- **Rationale:** Preserving flagged rows avoids silent data loss and supports audit trails.

---

## 🛠️ Feature Engineering

| Feature | Logic | Purpose |
|---|---|---|
| `annee`, `mois`, `trimestre`, `semaine-jour` | Extracted from `date_transaction` | Time-based trend analysis |
| `montant_eur_verifie` | `montant / taux_change_eur` | Cross-validate reported `montant_eur` |
| `categorie_risque` | Score ≥700 → Low; ≥580 → Medium; else → High | Risk segmentation |
| `total_credit` / `total_debit` | Per-client sum by operation type | Client financial profile |
| `solde_net` | `total_credit − total_debit` | Net position per client |
| `nb_transaction` | Count of transactions per client | Activity level |
| `montant_moyen` | Mean transaction amount per client | Average ticket size |
| `nb_produit` | Distinct product count per client | Product diversity |
| `taux_rejet` | % of rejected transactions per agency | Agency performance indicator |

---

## 📤 Export

- Cleaned data exported to: `financecore_clean.csv`
- Index column excluded (`index=False`) to avoid redundant row numbering.
