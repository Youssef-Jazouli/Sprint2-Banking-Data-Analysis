# DECISIONS.md — FinanceCore SA

Documentation de toutes les décisions de traitement des données prises durant le pipeline.

---

## 🔁 Doublons

- **Détection :** Les lignes partageant le même `transaction_id` ont été identifiées.
- **Décision :** Conservation de la **première occurrence**, suppression de toutes les suivantes.
- **Justification :** La première entrée enregistrée est considérée comme la transaction d'origine ; les entrées ultérieures sont probablement des re-soumissions ou des erreurs système.

---

## 📅 Dates

- **Problème :** `date_transaction` contenait des formats incohérents ou non parsables.
- **Décision :** Conversion au format unifié `AAAA-MM-JJ HH:MM:SS` avec `dayfirst=True`.
- **Valeurs manquantes :** Imputées par le **mode** (date la plus fréquente).
- **Justification :** L'imputation par le mode est préférable à la suppression des lignes afin de préserver l'historique des transactions.

---

## 💰 Montants (`montant`)

- **Problème :** Certaines valeurs utilisaient une virgule (`,`) comme séparateur décimal.
- **Décision :** Remplacement des virgules par des points et conversion en `float`.
- **Valeurs invalides :** Converties en `NaN` via `errors='coerce'`.
- **Justification :** Un format numérique standardisé est nécessaire pour tous les calculs.

---

## 💶 Solde (`solde_avant`)

- **Problème :** Les valeurs contenaient le suffixe ` EUR` sous forme de chaîne de caractères.
- **Décision :** Suppression du texte ` EUR` et conversion en `float`.
- **Justification :** La colonne doit être numérique pour tout calcul financier.

---

## 🔤 Champs Texte

| Colonne | Transformation |
|---|---|
| `devise` | Mise en majuscules et suppression des espaces |
| `segment_client` | Première lettre en majuscule |
| `agence` | Suppression des espaces en début/fin |

- **Justification :** Garantit des regroupements et des jointures cohérents ; évite les doublons dus à la casse ou aux espaces.

---

## 🩹 Valeurs Manquantes

| Colonne | Méthode | Justification |
|---|---|---|
| `score_credit_client` | **Médiane** | Robuste aux valeurs aberrantes dans une distribution asymétrique |
| `agence` | **Mode** | L'agence la plus fréquente est une valeur par défaut sûre |
| `segment_client` | **Mode** | Le segment le plus courant est une valeur par défaut sûre |
| `date_transaction` | **Mode** | Préserve les lignes tout en remplissant avec la date la plus typique |

---

## 🗑️ Colonnes Supprimées

- **`taux_interet` :** Supprimée entièrement.
- **Justification :** Colonne avec trop de valeurs manquantes ou jugée non pertinente pour l'analyse en aval.

---

## ⚠️ Détection des Valeurs Aberrantes

Trois conditions marquent une ligne comme anormale (`is_anomalie = True`) :

1. **`montant` hors IQR :** Valeur en dessous de `Q1 − 1,5×IQR` ou au-dessus de `Q3 + 1,5×IQR`.
2. **`score_credit_client` hors IQR :** Même règle IQR appliquée aux scores de crédit.
3. **Règle métier sur `score_credit_client` :** Score hors de l'intervalle valide `[0, 850]`.

- **Décision :** Les anomalies sont **signalées, non supprimées**, afin de laisser les équipes en aval décider du traitement à appliquer.
- **Justification :** Préserver les lignes signalées évite la perte silencieuse de données et facilite les audits.

---

## 🛠️ Ingénierie des Variables

| Variable | Logique | Objectif |
|---|---|---|
| `annee`, `mois`, `trimestre`, `semaine-jour` | Extraites de `date_transaction` | Analyse des tendances temporelles |
| `montant_eur_verifie` | `montant / taux_change_eur` | Vérification croisée du `montant_eur` déclaré |
| `categorie_risque` | Score ≥700 → Low ; ≥580 → Medium ; sinon → High | Segmentation par niveau de risque |
| `total_credit` / `total_debit` | Somme par client et par type d'opération | Profil financier du client |
| `solde_net` | `total_credit − total_debit` | Position nette par client |
| `nb_transaction` | Nombre de transactions par client | Niveau d'activité |
| `montant_moyen` | Montant moyen des transactions par client | Ticket moyen |
| `nb_produit` | Nombre de produits distincts par client | Diversité produit |
| `taux_rejet` | % de transactions rejetées par agence | Indicateur de performance agence |

---

## 📤 Export

- Données nettoyées exportées dans : `financecore_clean.csv`
- Colonne d'index exclue (`index=False`) pour éviter une numérotation redondante des lignes.
