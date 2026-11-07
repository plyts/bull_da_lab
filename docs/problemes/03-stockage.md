# 03 · Stockage – PostgreSQL / DuckDB

[← Retour à la synthèse](../../README.md#4-synthèse-des-problèmes-par-fonction) · Lab : [étape 5](../../README.md#parcours-du-lab)

Deux moteurs, deux rôles : **PostgreSQL 16 est la source de vérité** (transactionnel, contraintes, sécurité),
**DuckDB est un moteur de calcul analytique** en lecture. Cette séparation, actée dans un ADR, évite d'avoir
deux versions contradictoires de la même donnée.

| ID | Problème |
|---|---|
| [STO-01](#sto-01) | Intégrité et source de vérité |
| [STO-02](#sto-02) | Performance |
| [STO-03](#sto-03) | Partitionnement, indexation, rétention, archivage |
| [STO-04](#sto-04) | Sécurité |
| [STO-05](#sto-05) | Résilience |

---

<a id="sto-01"></a>

## STO-01 · Intégrité : doublons, versions contradictoires, absence de source de vérité

**Symptômes en production**
- Le backlog diffère entre l'API (Postgres) et un rapport calculé sur un fichier DuckDB de la veille.
- Deux processus écrivent dans le même fichier DuckDB : erreur de verrou ou fichier corrompu.

**Causes racines**
- Écritures possibles dans les deux moteurs.
- DuckDB n'accepte **qu'un seul processus écrivain** par fichier de base.
- Contraintes absentes (« on dédoublonne en aval »).

**Prévention (design)**
- **Postgres seul en écriture** pour les données de référence ; DuckDB est reconstruit à partir de Postgres/Parquet (jetable, reconstructible).
- Contraintes en base : `PRIMARY KEY`, `UNIQUE` sur clés naturelles, `FOREIGN KEY` dans `core`, `CHECK` (quantités ≥ 0).
- DuckDB : un seul écrivain (le job dbt-duckdb orchestré par Airflow), les lecteurs ouvrent en `read_only` ; publication par **écriture dans un nouveau fichier puis renommage atomique**.
- Chaque table analytique porte `_built_at` et la version source utilisée.

**Détection**
- Contrôle de réconciliation Postgres ↔ DuckDB après chaque build (comptages et sommes par mart).
- Alerte si l'âge du fichier DuckDB dépasse le SLA.

**Remédiation en production**
1. Écart constaté : Postgres fait foi ; reconstruire DuckDB (`make duckdb-rebuild`).
2. Fichier verrouillé : identifier le processus écrivain, l'arrêter, republier le fichier précédent.

**Dans le lab** — étape 5 · `make chaos-concurrent-duckdb-write` · test de réconciliation automatique.

---

<a id="sto-02"></a>

## STO-02 · Performance : requêtes coûteuses, jointures, scans complets, saturation

**Symptômes en production**
- Le endpoint `/v1/kpi/otd` passe de 200 ms à 12 s après six mois de données.
- Le CPU de Postgres est à 100 % pendant le `dbt build`, l'API ne répond plus.

**Causes racines**
- Requêtes analytiques lourdes (agrégations sur des millions de lignes) sur le moteur transactionnel.
- Index manquants sur les filtres et jointures, statistiques obsolètes.
- Pas de séparation des charges.

**Prévention (design)**
- Analytique lourde déportée sur **DuckDB** (moteur colonnes) ; Postgres sert les lectures ciblées et les marts pré-agrégés.
- `pg_stat_statements` activé ; toute requête de l'API est vérifiée par `EXPLAIN (ANALYZE, BUFFERS)` dans une revue.
- Index justifiés par une requête réelle ; marts pré-agrégés au grain demandé par l'API.
- `statement_timeout` par rôle (API : 5 s), `work_mem` adapté, pool de connexions (SQLAlchemy `pool_size`, PgBouncer en option).
- Fenêtre de build dbt hors des heures de forte consultation, `threads` limités.

**Détection**
- Top requêtes par temps total (`pg_stat_statements`), latence p95 de l'API, connexions actives.
- Alertes : p95 > 1 s, connexions > 80 % de `max_connections`, `seq_scan` en hausse sur une grosse table.

**Remédiation en production**
1. Identifier la requête : `SELECT query, calls, mean_exec_time FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;`
2. Requête bloquante : `pg_cancel_backend(pid)`, puis `pg_terminate_backend(pid)` en dernier recours.
3. Correctif : index (`CREATE INDEX CONCURRENTLY`, sans verrouiller la table), pré-agrégation ou bascule vers DuckDB.
4. `ANALYZE` après un gros chargement.

**Dans le lab** — étape 5 · `make load-test` (volume x100) · comparaison avant/après index et Postgres vs DuckDB.

---

<a id="sto-03"></a>

## STO-03 · Partitionnement, indexation, rétention et archivage

**Symptômes en production**
- Une purge `DELETE` de données anciennes verrouille la table et gonfle le disque (bloat).
- Les tables raw grossissent indéfiniment.

**Prévention (design)**
- **Partitionnement déclaratif mensuel** (`PARTITION BY RANGE (event_date)`) sur les grosses tables raw et faits ; création anticipée des partitions (job Airflow ou `pg_partman`).
- **Rétention par partition** : `ALTER TABLE … DETACH PARTITION … CONCURRENTLY`, export Parquet vers l'archive, puis `DROP` → pas de `DELETE` massif.
- Index locaux aux partitions ; on limite le nombre d'index sur les tables à fort taux d'écriture.
- Politique de rétention documentée par dataset (raw : 13 mois en ligne, archives : 5 ans, logs d'audit : selon obligation).

**Détection**
- Taille des tables et des index, ratio de bloat, partition future manquante (alerte J−7).

**Remédiation en production**
1. Partition manquante → les insertions échouent : créer la partition et rejouer les lots (idempotents).
2. Bloat : `VACUUM` / `REINDEX CONCURRENTLY` ; revoir l'autovacuum de la table.
3. Restaurer une archive : lecture du Parquet par DuckDB ou ré-attachement de la partition.

**Dans le lab** — étape 5 · `make retention-run` · test : la partition la plus ancienne est archivée et les requêtes courantes ne sont pas impactées.

---

<a id="sto-04"></a>

## STO-04 · Sécurité : chiffrement, contrôle des accès, données sensibles

**Symptômes en production**
- Un compte applicatif unique avec droits `superuser` utilisé par tous les composants.
- Des emails de contacts Salesforce visibles par tous les analystes.

**Prévention (design)**
- **Un rôle par composant** : `ingestion_writer` (raw), `dbt_runner` (staging→marts), `api_reader` (marts uniquement, lecture), `admin_ui`. Aucun accès `superuser` applicatif.
- `REVOKE ALL ON SCHEMA public FROM PUBLIC` ; `GRANT` explicites et `ALTER DEFAULT PRIVILEGES`.
- **Row-Level Security** sur les marts multi-organismes ([API-05](04-exposition.md#api-05)).
- Colonnes sensibles chiffrées (`pgcrypto`) ou pseudonymisées dès le staging ; vues masquées pour les rôles non habilités.
- TLS entre l'API et Postgres ; chiffrement du disque/volume en prod ; mots de passe via secrets Docker, jamais dans le dépôt.

**Détection**
- Audit des droits (requête sur `information_schema.role_table_grants`) en CI : échec si un rôle dépasse son périmètre.
- Journalisation des connexions (`log_connections`) et des modifications de rôles.

**Remédiation en production**
1. Fuite d'un mot de passe : rotation immédiate (`ALTER ROLE … PASSWORD`), redéploiement des secrets, revue des connexions.
2. Droit excessif constaté : `REVOKE`, puis correction de la migration Alembic qui gère les droits.

**Dans le lab** — étape 5 · test : `api_reader` ne peut ni écrire ni lire `raw` ; un analyste ne voit pas les colonnes PII.

---

<a id="sto-05"></a>

## STO-05 · Résilience : sauvegardes, restauration, réplication

**Symptômes en production**
- « On a des sauvegardes » — mais la dernière restauration testée date d'un an, et elle échoue.
- Suppression accidentelle d'une table à 14 h ; la sauvegarde date de 2 h du matin.

**Causes racines**
- Sauvegardes jamais testées.
- Sauvegarde logique quotidienne seule → RPO de 24 h.

**Prévention (design)**
- **Archivage WAL + sauvegarde physique** (pgBackRest ou `pg_basebackup`) → **restauration à un instant précis (PITR)**, RPO de quelques minutes.
- `pg_dump` quotidien en complément (restauration ciblée d'une table).
- Sauvegardes chiffrées, copiées hors de la machine (règle 3-2-1).
- Le schéma `snapshots` (historique SCD2, non reconstructible) est sauvegardé avec une rétention plus longue.
- Réplication en streaming vers un standby (option prod) pour la haute disponibilité.

**Détection**
- Alerte si la dernière sauvegarde réussie a plus de 26 h ou si l'archivage WAL échoue (`pg_stat_archiver`).
- **Test de restauration automatisé** hebdomadaire : restauration dans un conteneur jetable + requêtes de contrôle, durée mesurée (RTO).

**Remédiation en production**
1. Erreur humaine : PITR sur une instance temporaire à l'instant T − 1 min, extraction de la table, réinjection.
2. Perte du serveur : restauration complète depuis la dernière sauvegarde physique + WAL (runbook [INF-04](05-infrastructure.md#inf-04)).
3. Après restauration : relancer les ingestions depuis les watermarks (les sources sont rejouées grâce à l'idempotence).

**Dans le lab** — étapes 5 et 10 · `make drill-restore` · RPO et RTO réels consignés dans le post-mortem de l'exercice.
