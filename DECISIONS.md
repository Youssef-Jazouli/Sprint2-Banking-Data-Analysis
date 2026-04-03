# DECISIONS.md — Design & Technical Choices

This document records the key decisions made during the BriefBank data pipeline, along with the rationale and trade-offs for each.

---

## 1. Deduplication Strategy

**Decision:** Drop duplicate rows using `transaction_id` as the sole key, keeping the first occurrence.

**Rationale:** `transaction_id` is intended to be a unique business key. Among the 60 duplicates found, no business rule exists to prefer one occurrence over another, so the first-seen record is retained as the canonical one.

**Trade-off:** If a re-submitted transaction was legitimately corrected (e.g., an amended amount), the correction is silently discarded. A production system would need a `updated_at` timestamp to resolve this properly.

---

## 2. Missing Value Imputation

**Decision:**
- `score_credit_client` (167 nulls, 8.1%) → imputed with the **column median**
- `segment_client` (105 nulls, 5.1%) → imputed with the **mode**
- `agence` (64 nulls, 3.1%) → filled with the literal string `"Inconnue"`

**Rationale:**
- The median is robust to outliers for a bounded credit score variable (0–850).
- The mode is the appropriate central tendency for a categorical segment variable.
- `"Inconnue"` is preferred over dropping or forward-filling `agence` because branch assignment is a lookup attribute, not a sequential value; forward-filling would propagate a wrong branch.

**Trade-off:** Imputation introduces artificial data points. The `is_anomaly` flag does not account for imputed rows, so imputed credit scores near the boundary (580/700) may be miscategorized into the wrong risk tier.

---

## 3. `taux_interet` Column

**Decision:** Retain the column as-is (entirely null).

**Rationale:** Dropping it entirely would lose the schema intent. Marking it as a known gap makes it visible to downstream consumers and signals that this feature is planned but not yet populated.

**Trade-off:** Any model or aggregation that inadvertently uses this column will receive all nulls without warning. A schema validation step (e.g., Great Expectations) would make this safer.

---

## 4. Anomaly Detection Method

**Decision:** Use the **IQR fence** (`Q1 − 1.5×IQR`, `Q3 + 1.5×IQR`) on `montant`, combined with a hard domain bound on `score_credit_client` (0–850).

**Rationale:** IQR is distribution-agnostic and interpretable. The credit score bound is a known business rule (FICO scale), not a statistical estimate.

**Trade-off:**
- IQR flags ~5.4% of transactions (112/2060) as anomalies. In banking, high-value legitimate transactions (e.g., property purchases, large transfers) are common and may be over-flagged.
- No anomaly detection was applied to `montant_eur` or `solde_avant`.
- Anomalies are **flagged, not removed** — the pipeline preserves all rows, leaving the decision to downstream analysts or models.

---

## 5. EUR Cross-Check Column (`montant_eur_verifie`)

**Decision:** Add `montant_eur_verifie = montant / taux_change_eur` as a verification column separate from the existing `montant_eur`.

**Rationale:** `montant_eur` was pre-computed in the source system. An independent recalculation allows downstream auditors to detect discrepancies caused by rounding, rate snapshot differences, or ETL bugs.

**Trade-off:** The column is additive only — the pipeline does not automatically correct `montant_eur` when the two diverge. A reconciliation rule would be needed in production.

---

## 6. Credit Risk Tiers (`categorie_risque`)

**Decision:** Three tiers based on `score_credit_client`:
- **Low** — score ≥ 700
- **Medium** — score ≥ 580
- **High** — score < 580

**Rationale:** These thresholds approximate conventional FICO interpretations (Good/Fair/Poor) and are a reasonable default in the absence of a bank-specific policy document.

**Trade-off:** The thresholds are hard-coded constants. Any change to risk policy requires a code change. A configuration file or parameter table would be more maintainable.

---

## 7. Per-Client Aggregation Approach

**Decision:** Compute `nb_transactions`, `montant_moyen`, and `nb_produits` per `client_id` on the **cleaned, deduplicated** dataset, then left-join back onto the transaction table.

**Rationale:** Aggregating after deduplication ensures that duplicate transactions do not inflate client-level statistics.

**Trade-off:** Aggregation is computed over the entire dataset (no time window), so `nb_transactions` reflects all historical activity visible in the file, not a rolling window. For churn or behavioral models, a recency-weighted metric would be more useful.

---

## 8. Date Parsing

**Decision:** Use `pd.to_datetime(..., errors='coerce')` — invalid dates become `NaT` rather than raising an error.

**Rationale:** Coercion keeps the pipeline running on dirty data and makes bad dates visible (as NaT) rather than crashing the job.

**Trade-off:** Silently coerced dates will produce NaT values in the derived temporal columns (`annee`, `mois`, `trimestre`, `jour_semaine`), which may silently corrupt any time-based aggregation downstream. A post-parse assertion on the null count of `date_transaction` is advisable.

---

## 9. Output Format

**Decision:** Export to CSV (`financecore_clean.csv`) without the index.

**Rationale:** CSV is the most portable format for handoff to BI tools, SQL loaders, and other teams. Omitting the index avoids a spurious unnamed column on re-import.

**Trade-off:** CSV does not preserve dtypes (e.g., booleans become `True`/`False` strings, datetimes lose timezone info). A Parquet export would be more robust for a production pipeline.
