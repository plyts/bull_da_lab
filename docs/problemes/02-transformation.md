# 02 · Transformation – dbt

[← Retour à la synthèse](../../README.md#4-synthèse-des-problèmes-par-fonction) · Lab : [étape 6](../../README.md#parcours-du-lab)

La transformation est l'endroit où une donnée techniquement correcte devient un **chiffre métier**. Les problèmes ici
sont rarement des plantages : ce sont des chiffres faux qui passent inaperçus. Principe directeur du lab :
**chaque modèle a un grain, un propriétaire et des tests**, et un indicateur n'est défini qu'**une seule fois**.

| ID | Problème |
|---|---|
| [TRF-01](#trf-01) | Qualité des données |
| [TRF-02](#trf-02) | Cohérence métier |
| [TRF-03](#trf-03) | Modélisation : grain, SCD2, données tardives |
| [TRF-04](#trf-04) | Tests et évolutivité |
| [TRF-05](#trf-05) | Traçabilité : lineage, documentation, impact |

---

<a id="trf-01"></a>

## TRF-01 · Qualité : complétude, unicité, validité, cohérence, fraîcheur

**Symptômes en production**
- Le taux de non-conformité est à 0 % une semaine : l'export CQFD était vide.
- Des commandes sans client (clé étrangère orpheline) disparaissent des jointures.
- Le tableau de bord affiche des données de la veille sans que personne ne le sache.

**Causes racines**
- Tests absents ou uniquement en staging.
- Tests sans sévérité : soit tout bloque, soit rien.
- Pas de contrôle de fraîcheur des sources.

**Prévention (design)**

| Dimension | Contrôle dbt |
|---|---|
| Complétude | `not_null` sur les colonnes obligatoires, volume minimal attendu (test singulier) |
| Unicité | `unique` / `dbt_utils.unique_combination_of_columns` sur le grain |
| Validité | `accepted_values` (statuts SAP, codes défaut CQFD), plages de dates et de quantités |
| Cohérence | `relationships` (commande → client, ligne → article), sommes contrôlées entre couches |
| Fraîcheur | `sources: freshness: warn_after 6h, error_after 24h` sur `_ingested_at` |

- **Sévérités** : `error` bloque la publication des marts, `warn` notifie sans bloquer. Seuils `error_if: ">100"` / `warn_if: ">0"` sur les tests tolérants.
- `store_failures: true` : les lignes en échec sont conservées dans un schéma `dbt_test__audit` pour l'investigation.

**Détection**
- Résultats `run_results.json` chargés dans `audit.dbt_test_results` → tableau de bord qualité dans l'UI admin.
- Alerte sur tout test `error` et sur une hausse du nombre de `warn`.

**Remédiation en production**
1. Lire les lignes fautives dans `dbt_test__audit.<test>`.
2. Origine source → quarantaine / remontée au propriétaire. Origine transformation → correctif + PR.
3. Les marts restent sur la **dernière version valide** : `dbt build` s'arrête avant de publier les modèles en aval d'un test en échec.

**Dans le lab** — étape 6 · `make chaos-empty-export`, `make chaos-orphans` · test : le mart n'est pas publié et le test en échec est visible dans l'UI.

---

<a id="trf-02"></a>

## TRF-02 · Cohérence métier : règles, référentiels, mappings, définitions des indicateurs

**Symptômes en production**
- La Qualité et les Achats n'ont pas le même OTD fournisseur.
- Le même fournisseur porte un code différent dans SAP et dans le flux EDI.
- Une règle (« une livraison est à l'heure si elle arrive ≤ J+2 ») est codée différemment dans trois modèles.

**Causes racines**
- Définitions d'indicateurs dupliquées dans plusieurs requêtes ou outils.
- Mappings maintenus dans des fichiers Excel hors du dépôt.

**Prévention (design)**
- **Seeds versionnés** pour les mappings et référentiels (`seed_supplier_mapping.csv`, `seed_defect_codes.csv`) avec tests d'unicité.
- Règles métier dans des **macros** uniques (`is_on_time(promised, delivered, tolerance_days)`).
- Chaque indicateur est calculé **dans un seul mart** et documenté (définition, formule, owner, exemples) dans le glossaire ([GOV-03](06-gouvernance-exploitation.md#gov-03)).
- **Unit tests dbt** (dbt ≥ 1.8) sur les règles : entrées fictives → sortie attendue, validées avec le métier.

```yaml
unit_tests:
  - name: otd_tolerance_two_days
    model: int_supplier_deliveries
    given:
      - input: ref('stg_supplier_deliveries')
        rows:
          - {delivery_id: 1, promised_date: 2026-10-01, delivered_date: 2026-10-03}
          - {delivery_id: 2, promised_date: 2026-10-01, delivered_date: 2026-10-04}
    expect:
      rows:
        - {delivery_id: 1, is_on_time: true}
        - {delivery_id: 2, is_on_time: false}
```

**Détection**
- Test singulier de **réconciliation** : total des quantités livrées en mart = total en staging.
- Test `relationships` vers les seeds : tout code non mappé fait échouer le build.

**Remédiation en production**
1. Divergence d'indicateur : retrouver la définition de référence dans le glossaire et l'appliquer (une seule source).
2. Code non mappé : ajouter la ligne au seed via PR, validée par le data owner.
3. Changement de règle : nouvelle version du mart ([TRF-04](#trf-04)) et communication aux consommateurs.

**Dans le lab** — étape 6 · `make chaos-unmapped-supplier` · test : un fournisseur inconnu bloque le build avec un message explicite.

---

<a id="trf-03"></a>

## TRF-03 · Modélisation : grain, historisation SCD2, données tardives

**Symptômes en production**
- Un `SUM` de quantités double après une jointure avec la nomenclature (un à plusieurs non prévu).
- La non-conformité d'un serveur est rattachée à la révision **actuelle** du produit au lieu de celle en vigueur à la date du défaut.
- Le modèle incrémental ne prend jamais en compte les livraisons arrivées en retard.

**Causes racines**
- Grain non documenté, jointures « fan-out ».
- Dimensions écrasées (SCD1) alors que le métier a besoin de l'historique.
- Incrémental filtré sur `max(_ingested_at)` uniquement, sans fenêtre de rattrapage.

**Prévention (design)**
- Chaque modèle déclare son grain dans sa description **et** le teste (`unique_combination_of_columns`).
- **Snapshots dbt** (stratégie `check` ou `timestamp`) pour les révisions PLM et les fournisseurs → `dim_product_revision` avec `valid_from`, `valid_to`, `is_current`.
- Jointure temporelle : `nc.detected_at >= rev.valid_from AND nc.detected_at < coalesce(rev.valid_to, 'infinity')`.
- Modèles incrémentaux avec **fenêtre de rattrapage** (`lookback` de N jours) et `unique_key` pour un merge idempotent.

```sql
{{ config(materialized='incremental', unique_key='delivery_id', incremental_strategy='merge') }}
select * from {{ ref('int_supplier_deliveries') }}
{% if is_incremental() %}
where event_date >= (select max(event_date) from {{ this }}) - interval '{{ var("lookback_days", 7) }} days'
{% endif %}
```

**Détection**
- Test de grain sur chaque fait.
- Test SCD2 : pas de chevauchement de périodes pour une même clé, un seul `is_current` par clé.
- Écart entre un rebuild complet et l'incrémental (contrôle hebdomadaire).

**Remédiation en production**
1. Fan-out : corriger la jointure (pré-agrégation au bon grain), rebuild du mart.
2. Historique SCD2 corrompu : les snapshots ne se rejouent pas — d'où la **sauvegarde dédiée** du schéma `snapshots` ([STO-05](03-stockage.md#sto-05)).
3. Retards au-delà de la fenêtre : `--full-refresh` ciblé ou augmentation temporaire de `lookback_days`.

**Dans le lab** — étape 6 · `make chaos-plm-revision`, `make chaos-late-events` · test : un défaut est rattaché à la bonne révision, l'incrémental converge vers le full refresh.

---

<a id="trf-04"></a>

## TRF-04 · Tests et évolutivité : changements de schéma et dépendances

**Symptômes en production**
- Le renommage d'une colonne dans un mart casse l'API et un tableau de bord.
- Un `dbt run` complet prend 40 minutes en CI pour une modification d'une ligne.

**Causes racines**
- Pas de contrat entre le mart et ses consommateurs.
- CI qui reconstruit tout le projet.

**Prévention (design)**
- **Contrats de modèles** (`contract: enforced: true`) sur les marts exposés : noms et types de colonnes garantis.
- **Versions de modèles** (`versions:`) pour les changements cassants : `v1` et `v2` coexistent le temps de la migration des consommateurs.
- **Slim CI** : `dbt build --select state:modified+ --defer --state prod-artifacts/` → seuls les modèles modifiés et leur aval sont testés.
- Le même projet tourne sur **dbt-postgres et dbt-duckdb** ; les différences SQL sont isolées dans des macros `adapter.dispatch`. La CI teste les deux adapters.

**Détection**
- La CI échoue si un contrat est rompu ou si un modèle `v1` encore référencé (`exposures`) est supprimé.

**Remédiation en production**
1. Changement cassant déjà déployé : rollback de l'image dbt (tag précédent), puis réintroduction via une nouvelle version de modèle.
2. Publier une date de dépréciation (`deprecation_date`) pour l'ancienne version.

**Dans le lab** — étapes 1 et 6 · PR de démonstration qui renomme une colonne → la CI échoue sur le contrat.

---

<a id="trf-05"></a>

## TRF-05 · Traçabilité : lineage, documentation et analyse d'impact

**Symptômes en production**
- « Si je modifie `stg_sap_orders`, qu'est-ce qui casse ? » — personne ne sait répondre.
- Un indicateur faux est découvert, impossible de remonter à la source.

**Prévention (design)**
- `dbt docs generate` publié à chaque déploiement : description de chaque modèle et colonne, graphe de lineage.
- **Exposures** pour l'API FastAPI, l'UI Streamlit et les exports : le lineage va jusqu'au consommateur.
- Métadonnées `meta: {owner, domain, pii, classification}` sur chaque modèle, réutilisées par la gouvernance ([GOV-03](06-gouvernance-exploitation.md#gov-03)).
- Les colonnes techniques `_batch_id` / `_source_file` sont propagées jusqu'aux faits : on remonte d'un chiffre au fichier d'origine.

**Détection**
- Test CI qui échoue si un modèle n'a pas de description ou d'owner.

**Remédiation en production**
1. Analyse d'impact avant changement : `dbt ls --select stg_sap_orders+ --resource-type model exposure`.
2. Investigation d'un chiffre faux : du mart vers `_batch_id`, puis `audit.ingestion_runs`, puis le fichier archivé.

**Dans le lab** — étapes 6 et 9 · `make docs` · démonstration : remonter un chiffre de l'API jusqu'au fichier source.
