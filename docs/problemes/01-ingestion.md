# 01 · Ingestion – ETL/ELT Python

[← Retour à la synthèse](../../README.md#4-synthèse-des-problèmes-par-fonction) · Lab : [étapes 2, 3 et 4](../../README.md#parcours-du-lab)

L'ingestion est la couche où se créent la plupart des erreurs silencieuses : une donnée perdue ou dupliquée ici
fausse tous les indicateurs en aval. Principe directeur du lab : **tout chargement doit pouvoir être rejoué
sans effet de bord** (idempotence) et **tout écart doit laisser une trace** (audit).

| ID | Problème |
|---|---|
| [ING-01](#ing-01) | Doublons, pertes de données et non-idempotence |
| [ING-02](#ing-02) | Erreurs des sources : timeouts, indisponibilité, pagination, 429 |
| [ING-03](#ing-03) | CDC, suppressions, événements désordonnés et données tardives |
| [ING-04](#ing-04) | Reprises : backfill, retries et collisions |
| [ING-05](#ing-05) | Traçabilité et changements de schéma |

---

<a id="ing-01"></a>

## ING-01 · Doublons, pertes de données et non-idempotence

**Symptômes en production**
- Le backlog commandes double après une relance manuelle du pipeline.
- Un IDoc SAP renvoyé par l'émetteur (même `DOCNUM`) apparaît deux fois.
- Écart entre le nombre de lignes de l'export source et le nombre chargé, sans erreur visible.

**Causes racines**
- `INSERT` simple au lieu d'un `UPSERT` : un retry après échec partiel réinsère ce qui était déjà passé.
- Pas de clé naturelle identifiée par source.
- Commit au fil de l'eau : un crash au milieu laisse un lot à moitié chargé.
- Fichier déplacé/supprimé avant la confirmation du chargement (perte).

**Prévention (design)**
- Une **clé naturelle** par entité (`sap_docnum + segment`, `salesforce_id`, `cqfd_nc_id`…) et un **hash du contenu** (`_hash`).
- Chargement en transaction unique par lot, avec `INSERT … ON CONFLICT (clé) DO UPDATE … WHERE raw._hash <> EXCLUDED._hash`
  (pas de mise à jour inutile si la ligne n'a pas changé).
- Table `audit.ingestion_runs` : `batch_id`, source, fenêtre, `rows_read`, `rows_inserted`, `rows_updated`, `rows_rejected`, statut.
- Registre des fichiers traités (`audit.processed_files` avec hash du fichier) : un fichier déjà traité est ignoré.
- Le fichier source n'est archivé **qu'après** le commit.

```sql
INSERT INTO raw.sap_orders AS t (docnum, payload, _hash, _batch_id, _ingested_at)
VALUES (:docnum, :payload, :hash, :batch_id, now())
ON CONFLICT (docnum) DO UPDATE
   SET payload = EXCLUDED.payload, _hash = EXCLUDED._hash,
       _batch_id = EXCLUDED._batch_id, _ingested_at = EXCLUDED._ingested_at
 WHERE t._hash IS DISTINCT FROM EXCLUDED._hash;
```

**Détection**
- Rapprochement automatique `rows_read = rows_inserted + rows_updated + rows_unchanged + rows_rejected` ; sinon le run est en échec.
- Test dbt `unique` sur la clé naturelle en staging.
- Comparaison périodique des comptages avec la source (contrôle de complétude J+1).

**Remédiation en production**
1. Identifier les `batch_id` concernés dans `audit.ingestion_runs`.
2. Si doublons : la clé naturelle permet de dédoublonner (`DELETE … USING` en gardant le `_ingested_at` le plus récent), puis ajouter la contrainte manquante.
3. Si perte : rejouer la fenêtre via un backfill (voir [ING-04](#ing-04)) — sans risque grâce à l'idempotence.
4. Relancer `dbt build --select source:sap+` pour recalculer l'aval.

**Dans le lab** — étape 3 · scénario `make chaos-duplicates` (le faux SAP renvoie 10 % des IDocs) · test : deux exécutions du même lot donnent un état identique.

---

<a id="ing-02"></a>

## ING-02 · Erreurs des sources : timeouts, indisponibilité, pagination incomplète, 429

**Symptômes en production**
- Le DAG Salesforce échoue chaque matin à 8 h (pic d'usage de l'API).
- Chargement « réussi » mais seulement 2 000 comptes sur 15 000 : la pagination s'est arrêtée sur une erreur avalée.
- Bannissement temporaire de l'API après des retries trop agressifs.

**Causes racines**
- Retries immédiats et sans limite qui aggravent la surcharge.
- En-tête `Retry-After` ignoré.
- Exception attrapée trop largement (`except Exception: pass`).
- Pagination par offset sur des données qui bougent pendant la lecture.

**Prévention (design)**
- Client HTTP commun à tous les connecteurs :
  - timeouts explicites (connexion/lecture) ;
  - **retry avec backoff exponentiel + jitter** (tenacity), uniquement sur les erreurs transitoires (429, 502, 503, 504, timeouts) ;
  - respect de `Retry-After`.
- **Circuit breaker** : après N échecs consécutifs, on arrête d'appeler la source pendant une durée donnée.
- **Pagination par curseur** (`nextRecordsUrl` côté Salesforce) avec **checkpoint** persistant : reprise au dernier curseur validé.
- Débit plafonné (pool Airflow `salesforce_api` à 2 slots) pour rester sous le quota.
- Contrôle de fin : le total annoncé par l'API (`totalSize`) doit correspondre au total lu.

**Détection**
- Métriques `source_http_errors_total{source,status}` et `source_retry_total`.
- Alerte si le taux de 429 dépasse 5 % sur 15 minutes.
- Alerte si `rows_read < 0.8 × moyenne 7 jours` (voir [GOV-02](06-gouvernance-exploitation.md#gov-02)).

**Remédiation en production**
1. Source indisponible : le circuit breaker passe le run en `up_for_retry` ; aucune action tant que le SLA n'est pas menacé.
2. Quota épuisé : réduire le pool, décaler la planification hors des heures de pointe.
3. Pagination interrompue : relancer le run, qui repart du dernier checkpoint (pas de relecture complète).
4. Prévenir les consommateurs si la fraîcheur dépasse le seuil (`dbt source freshness` en `error`).

**Dans le lab** — étapes 2-3 · `make chaos-429-storm`, `make chaos-timeouts`, `make chaos-broken-pagination` · test : le chargement complet aboutit malgré 30 % d'erreurs transitoires.

---

<a id="ing-03"></a>

## ING-03 · CDC, suppressions, événements désordonnés et données tardives

**Symptômes en production**
- Une opportunité supprimée dans Salesforce reste visible dans les indicateurs.
- Une livraison fournisseur « annulée » écrase la livraison « confirmée » arrivée après elle mais émise avant.
- L'OTD du mois dernier change sans explication : des livraisons sont arrivées avec trois jours de retard.

**Causes racines**
- Extraction incrémentale basée sur `updated_at > dernier_run` : perte des lignes modifiées pendant l'extraction, ou d'horloges décalées.
- Les suppressions physiques ne sont pas visibles dans un export incrémental.
- Ordre d'arrivée ≠ ordre métier.

**Prévention (design)**
- **Watermark avec chevauchement** : on relit `[dernier_watermark − 15 min ; maintenant]`. Les doublons créés par le chevauchement sont absorbés par l'idempotence ([ING-01](#ing-01)).
- Watermark stocké dans `audit.watermarks` et avancé **dans la même transaction** que le chargement.
- Suppressions :
  - Salesforce : lecture de `IsDeleted` via `queryAll` ;
  - autres sources : diff de snapshot périodique ;
  - dans les deux cas → `_is_deleted = true`, `_deleted_at` (soft delete, jamais de suppression physique en raw).
- Deux temps distincts : `event_time` (métier) et `_ingested_at` (technique). La version retenue est celle qui a le **plus grand `event_time` / numéro de séquence**, pas la dernière arrivée.
- Côté dbt, modèles incrémentaux avec **fenêtre de rattrapage** (voir [TRF-03](02-transformation.md#trf-03)).

**Détection**
- Indicateur `late_arrival_ratio` = part des lignes avec `_ingested_at − event_time > 24 h`.
- Contrôle de cohérence : nombre d'enregistrements actifs vs source (snapshot hebdomadaire).

**Remédiation en production**
1. Données tardives au-delà de la fenêtre : lancer un rebuild ciblé (`dbt build --select fct_supplier_deliveries --full-refresh` sur la partition concernée).
2. Communiquer la **politique de clôture** aux métiers : un mois est « provisoire » jusqu'à J+5, puis figé.
3. Suppressions non captées : forcer un snapshot complet de la source.

**Dans le lab** — étape 3 · `make chaos-late-events`, `make chaos-out-of-order`, `make chaos-deletes` · test : le résultat final est identique quel que soit l'ordre d'arrivée.

---

<a id="ing-04"></a>

## ING-04 · Reprises : backfill, retries et collisions avec les chargements courants

**Symptômes en production**
- Un backfill de 6 mois lancé en journée bloque le chargement quotidien (verrous, saturation).
- Le backfill et le run planifié écrivent la même partition : résultat incohérent.
- Le watermark recule après un backfill, et le run suivant recharge tout.

**Causes racines**
- Backfill exécuté avec le même code et le même état (watermark) que le run courant.
- Pas d'exclusion mutuelle par périmètre de données.

**Prévention (design)**
- Chaque run est **paramétré par sa fenêtre** (`data_interval_start/end` d'Airflow), jamais par « maintenant ».
- Le backfill **ne touche pas au watermark** du flux courant.
- **Verrou consultatif Postgres** par (source, partition) : `pg_try_advisory_xact_lock(hashtext('sap_orders:2026-10'))`. Un second run sur le même périmètre attend ou échoue proprement.
- Airflow 3 :
  - backfill natif avec `max_active_runs` limité ;
  - pool dédié `backfill` à faible priorité, pour ne pas affamer les runs planifiés.
- Retries Airflow uniquement si la tâche est idempotente (c'est le cas grâce à [ING-01](#ing-01)).

**Détection**
- Durée d'attente des verrous (`pg_locks`, `pg_stat_activity` avec `wait_event_type = 'Lock'`).
- Alerte si un run planifié dépasse son délai (Deadline Alerts d'Airflow 3.1+).

**Remédiation en production**
1. Mettre en pause le backfill (`max_active_runs = 0` ou pause du DAG), laisser passer le run courant.
2. Reprendre le backfill hors des heures de production, par tranches mensuelles.
3. Vérifier le rapprochement des volumes par partition dans `audit.ingestion_runs`.

**Dans le lab** — étapes 3-4 · `make backfill FROM=2026-01-01 TO=2026-06-30` pendant un run planifié · test : aucun doublon, aucune perte, watermark courant inchangé.

---

<a id="ing-05"></a>

## ING-05 · Traçabilité des sources et détection des changements de schéma

**Symptômes en production**
- CQFD ajoute une colonne et en renomme une autre : le pipeline plante, ou pire, charge des `NULL` en silence.
- Impossible de dire de quel fichier vient une valeur aberrante.

**Causes racines**
- Lecture permissive (`pd.read_csv` puis insertion de « ce qui correspond »).
- Pas de métadonnées techniques sur les lignes.

**Prévention (design)**
- **Contrat de schéma** par source (modèle Pydantic + empreinte du schéma attendu).
  - Colonne ajoutée → *warning*, la colonne est ignorée et signalée.
  - Colonne supprimée, renommée ou type incompatible → le lot entier part en **quarantaine** (`quarantine.batches` + fichier original conservé).
- Validation au niveau de la ligne : les lignes invalides vont dans `quarantine.rows` avec le motif du rejet ; le reste du lot passe si le taux de rejet est sous le seuil (ex. 1 %).
- Colonnes techniques sur chaque ligne raw : `_source`, `_source_file`, `_file_hash`, `_batch_id`, `_ingested_at`, `_schema_version`.

**Détection**
- Alerte à chaque lot mis en quarantaine.
- Métrique `rows_rejected_ratio{source}`.
- Historique `audit.schema_versions` : date du changement d'empreinte.

**Remédiation en production**
1. Ouvrir la page *Quarantaine* de l'UI admin : voir le diff entre schéma attendu et schéma reçu.
2. Changement légitime : adapter le contrat (PR + version de schéma), puis **rejouer** le lot depuis l'UI.
3. Changement non annoncé : remonter au propriétaire de la source ([GOV-03](06-gouvernance-exploitation.md#gov-03)), le lot reste isolé.
4. Aucun indicateur n'est calculé sur un lot partiel.

**Dans le lab** — étapes 3 et 8 · `make chaos-schema-drift` · test : un fichier au schéma modifié n'altère aucune table raw et apparaît en quarantaine.
