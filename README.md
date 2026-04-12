# 🏦 FinanceCore SA — Pipeline de Données Bancaires

Un pipeline de nettoyage et d'ingénierie des données pour les transactions bancaires.

## 📁 Structure du Projet

```
├── bank.csv                  # Données brutes en entrée
├── breif_bank.ipynb          # Notebook d'analyse principal
├── financecore_clean.csv     # Données nettoyées en sortie
├── DECISIONS.md              # Journal des décisions
└── README.md                 # Ce fichier
```

## 🔄 Aperçu du Pipeline

### 1. Importation & Exploration
- Chargement de `bank.csv` et normalisation des noms de colonnes (minuscules, espaces supprimés)
- Affichage des types de données, de la forme et des statistiques descriptives
- Visualisation du taux de valeurs manquantes par colonne
- Identification des `transaction_id` en double et calcul des écarts temporels entre eux

### 2. Nettoyage des Données
- Suppression des doublons sur `transaction_id`, en conservant la première occurrence
- Conversion de `date_transaction` en datetime (format jour en premier)
- Correction de `montant` : remplacement de la virgule par un point, conversion en float
- Correction de `solde_avant` : suppression du suffixe ` EUR`, conversion en float
- Standardisation des champs texte : `devise` → majuscules, `segment_client` → première lettre en majuscule, `agence` → espaces supprimés
- Imputation des valeurs manquantes :
  - `score_credit_client` → médiane
  - `agence`, `segment_client`, `date_transaction` → mode
- Suppression de la colonne `taux_interet`

### 3. Détection des Valeurs Aberrantes
- Application de la méthode IQR sur `montant` et `score_credit_client`
- Règle métier : les scores de crédit doivent être dans l'intervalle [0, 850]
- Les anomalies sont signalées dans une nouvelle colonne booléenne `is_anomalie`

### 4. Ingénierie des Variables

| Variable | Description |
|---|---|
| `annee`, `mois`, `trimestre`, `semaine-jour` | Composantes temporelles |
| `montant_eur_verifie` | Montant EUR recalculé via le taux de change |
| `categorie_risque` | Niveau de risque : Low (≥700) / Medium (≥580) / High (<580) |
| `total_credit`, `total_debit` | Montants agrégés par client |
| `solde_net` | Solde net par client (crédit − débit) |
| `nb_transaction` | Nombre de transactions par client |
| `montant_moyen` | Montant moyen des transactions par client |
| `nb_produit` | Nombre de produits distincts par client |
| `taux_rejet` | Taux de rejet (%) par agence |

### 5. Export
- Sauvegarde du jeu de données nettoyé dans `financecore_clean.csv`
- Écriture de la documentation dans `DECISIONS.md`

## ▶️ Comment Exécuter

```bash
pip install pandas matplotlib
jupyter notebook breif_bank.ipynb
```

## 📊 Colonnes Principales

| Colonne | Description |
|---|---|
| `transaction_id` | Identifiant unique de la transaction |
| `client_id` | Identifiant du client |
| `date_transaction` | Horodatage de la transaction |
| `montant` | Montant de la transaction |
| `montant_eur` | Montant en euros |
| `taux_change_eur` | Taux de change vers l'euro |
| `solde_avant` | Solde du compte avant la transaction |
| `type_operation` | Crédit ou Débit |
| `statut` | Statut de la transaction (ex. : Rejeté) |
| `devise` | Code devise |
| `segment_client` | Segment du client |
| `agence` | Nom de l'agence |
| `produit` | Produit bancaire |
| `score_credit_client` | Score de crédit du client (0–850) |
