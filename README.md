# Modern Data Lakehouse Platform — Databricks

Pipeline de données e-commerce en architecture médaillon (Bronze → Silver → Gold) sur **Databricks**, avec **Delta Lake**, **Unity Catalog** et un **job Databricks** à 4 tâches. Les données sources sont volontairement sales : le pipeline les nettoie, garde chaque ligne rejetée dans une table dédiée et produit des tables d'analyse fiables.

Ce projet est la version « plateforme managée » de [data_lakehouse](https://github.com/khadijafair/data_lakehouse) (version locale open source : Spark, Apache Iceberg, Trino, dbt, Kafka, Airflow). **Les deux versions appliquent la même logique métier et donnent exactement les mêmes comptages** (vérifié le 13/11/2026).

## Stack

| Rôle | Outil |
|---|---|
| Plateforme | Databricks Free Edition (calcul serverless) |
| Traitement | Apache Spark (PySpark) et Spark SQL |
| Format des tables | Delta Lake |
| Gouvernance et catalogue | Unity Catalog : catalogue `workspace`, schéma `lakehouse_demo`, volume `raw_files` |
| Orchestration | Databricks Jobs (job `Lakehouse_Databricks_Batch`) |
| Ingestion de fichiers en continu | Auto Loader (notebook `08_auto_loader`, en cours) |
| Versionnement | Dossier Git Databricks relié à ce dépôt GitHub |

## Architecture

```
Volume Unity Catalog                Bronze                 Silver                          Gold
/Volumes/workspace/lakehouse_demo   (données brutes)       (nettoyées, validées)           (agrégats métier)
  /raw_files
  customers_raw.csv  ───────────►  bronze_customers ──►  silver_customers  ─┐
  products_raw.csv   ───────────►  bronze_products  ──►  silver_products   ─┼─► silver_orders_enriched ─► gold_sales_daily
  orders_raw.csv     ───────────►  bronze_orders    ──►  silver_orders     ─┘   (jointure)               gold_sales_by_product
                                                          │
                                                          └─► reject_customers, reject_products, reject_orders,
                                                              reject_order_references, audit_customer_duplicates
```

Toutes les tables sont des tables Delta du schéma `workspace.lakehouse_demo` (18 objets au total). Le schéma contient aussi une vue d'analyse, `v_customers_by_location`, et 3 tables d'exercices issues des notebooks 04 et 07 : `delta_orders_practice`, `orders_incremental_practice` et `reject_incremental_conflicts_practice`.

## Résultats (comptages vérifiés le 13/11/2026)

| Table | Lignes | Commentaire |
|---|---:|---|
| bronze_customers | 50 000 | données brutes |
| bronze_products | 500 | données brutes |
| bronze_orders | 300 000 | données brutes |
| silver_customers | 47 200 | après dédoublonnage et validation |
| audit_customer_duplicates | 1 800 | doublons de clients détectés |
| reject_customers | 1 000 | clients invalides |
| reject_products | 95 | produits invalides (405 gardés) |
| silver_orders | 150 137 | commandes valides |
| reject_orders | 149 863 | commandes invalides (dates, quantités, montants, moyens de paiement) |
| reject_order_references | 31 228 | commandes valides mais liées à un client ou un produit rejeté |
| gold_sales_daily | 730 | ventes par jour |
| gold_sales_by_product | 405 | ventes par produit |

Le chiffre d'affaires ne compte que les commandes au statut `success`. Aucune ligne n'est perdue sans trace : chaque ligne écartée se retrouve dans une table de rejet ou d'audit.

## Notebooks (`notebooks/`)

| Notebook | Rôle |
|---|---|
| `00_setup` | Préparation de l'environnement |
| `01_bronze` | Ingestion brute des 3 CSV depuis le volume, contrôle des colonnes attendues |
| `02_silver` | Nettoyage, validation, dédoublonnage, tables de rejet et d'audit |
| `03_gold` | Agrégats métier (ventes par jour, par produit) |
| `04_delta_practice` | Fonctionnalités Delta Lake (exercices) |
| `05_quality_check` | Contrôles qualité, 4e tâche du job |
| `06_spark_performance` | Optimisation des traitements Spark (exercices) |
| `07_incremental_orders` | Chargement incrémental des commandes |
| `08_auto_loader` | Ingestion de fichiers avec Auto Loader (en cours) |

## Orchestration

Le job Databricks `Lakehouse_Databricks_Batch` enchaîne 4 tâches dépendantes : **Bronze → Silver → Gold → quality_check**. Une tâche ne démarre que si la précédente a réussi. Il a été exécuté 5 fois avec succès les 3 et 4 octobre 2026.

## Données

Jeu de données Kaggle [Customer 360](https://www.kaggle.com/datasets/vinaykandimalla/customer-360) (données synthétiques, volontairement sales ; licence non indiquée sur la page). Les CSV ne sont pas versionnés dans ce dépôt.

## Reproduire

1. Créer un compte [Databricks Free Edition](https://www.databricks.com/learn/free-edition).
2. Relier son compte GitHub (Settings → Linked accounts), puis créer un dossier Git à partir de ce dépôt.
3. Créer le schéma `workspace.lakehouse_demo` et le volume `raw_files`, y déposer `customers_raw.csv`, `products_raw.csv` et `orders_raw.csv`.
4. Exécuter les notebooks `00_setup` à `03_gold`, puis `05_quality_check` (ou créer un job avec ces 4 tâches).

## Limites et suite

- Données synthétiques : les montants et le chiffre d'affaires ne reflètent pas un vrai commerce.
- Databricks Free Edition : calcul serverless uniquement, usage non commercial.
- À venir : tables Gold `sales_by_region` et `customer_metrics` ; ingestion du clickstream avec Auto Loader et entonnoir de conversion (vues → paniers → commandes).
