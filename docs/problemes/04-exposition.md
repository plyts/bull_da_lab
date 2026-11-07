# 04 · Exposition – FastAPI / SQLAlchemy / Keycloak

[← Retour à la synthèse](../../README.md#4-synthèse-des-problèmes-par-fonction) · Lab : [étape 7](../../README.md#parcours-du-lab)

L'API est la porte d'entrée vers les données : c'est là que se jouent la sécurité, la conformité et la confiance des
consommateurs. Principe directeur du lab : **défense en profondeur** — le jeton Keycloak, l'API et la base
(RLS) appliquent chacun le contrôle d'accès ; une faille dans une couche ne suffit pas à exposer les données.

| ID | Problème |
|---|---|
| [API-01](#api-01) | Authentification, autorisation RBAC, comptes |
| [API-02](#api-02) | Chiffrement, secrets, injections |
| [API-03](#api-03) | Versionnage, pagination, erreurs, disponibilité |
| [API-04](#api-04) | Rate limiting, timeouts, volumes sortants |
| [API-05](#api-05) | Traçabilité, isolation des organismes, droits |
| [API-06](#api-06) | Minimisation et RGPD |

---

<a id="api-01"></a>

## API-01 · Sécurité des accès : authentification, RBAC, gestion des comptes

**Symptômes en production**
- Un jeton émis pour l'UI Streamlit est accepté par l'API (mauvaise audience).
- Un ancien prestataire accède toujours aux données : son compte n'a jamais été désactivé.
- Après une rotation des clés Keycloak, toutes les requêtes sont rejetées pendant 10 minutes.

**Causes racines**
- Validation partielle du JWT (signature seule, sans `aud`, `iss` ni `exp`).
- Rôles vérifiés dans chaque route « à la main ».
- Comptes locaux hors SSO.

**Prévention (design)**
- Realm Keycloak `bull-lab`, un client par application (`api`, `admin-ui`), rôles `viewer`, `analyst`, `steward`, `admin`.
- Validation complète du JWT : signature via **JWKS mis en cache avec rafraîchissement sur `kid` inconnu**, `iss`, `aud`, `exp`, `nbf`.
- Autorisation déclarative par **dépendance FastAPI** : `Depends(require_roles("analyst"))` ; refus par défaut.
- Jetons d'accès courts (5 min), refresh tokens côté UI ; comptes de service (`client_credentials`) pour les intégrations.
- Revue périodique des comptes ; désactivation centralisée dans Keycloak (pas de comptes locaux).

**Détection**
- Métriques `auth_failures_total{reason}` (expiré, audience, signature) ; alerte sur un pic.
- Événements de connexion Keycloak exportés vers les logs centralisés.

**Remédiation en production**
1. Compte compromis : désactivation dans Keycloak + révocation des sessions (`logout all sessions`) ; les jetons d'accès expirent en 5 min.
2. Rotation de clés : publier la nouvelle clé avant de retirer l'ancienne ; l'API rafraîchit le JWKS automatiquement.

**Dans le lab** — étape 7 · tests : jeton expiré, mauvaise audience, rôle insuffisant → 401/403 ; rotation de clé sans interruption.

---

<a id="api-02"></a>

## API-02 · Sécurité des échanges : chiffrement, secrets, injections

**Symptômes en production**
- Le mot de passe de la base est visible dans `docker inspect` ou dans un log d'exception.
- Un filtre `?supplier=` construit par concaténation permet une injection SQL.

**Prévention (design)**
- TLS de bout en bout (reverse proxy devant l'API, TLS vers Postgres et Keycloak).
- Secrets injectés via **Docker secrets** / variables du gestionnaire de secrets, lus par `pydantic-settings` avec `SecretStr` (jamais logués).
- **SQLAlchemy 2.x uniquement avec des paramètres liés** (`select(...).where(Model.col == value)`) ; pas de `text()` construit par f-string.
- Validation stricte des entrées par Pydantic (types, longueurs, énumérations) ; tri et filtres limités à une **liste blanche** de colonnes.
- En-têtes de sécurité, CORS restreint aux origines connues.

**Détection**
- CI : `bandit` (sécurité Python) et gitleaks ; tests d'injection sur les paramètres de filtre.
- Filtre de logs qui masque les motifs de secrets.

**Remédiation en production**
1. Secret exposé : rotation, purge des logs concernés, analyse de l'historique d'accès.
2. Vulnérabilité d'injection : correctif + test de non-régression, recherche d'exploitation dans les logs d'audit.

**Dans le lab** — étape 7 · test : `?supplier=' OR 1=1 --` renvoie 422 et aucun accès anormal.

---

<a id="api-03"></a>

## API-03 · Fiabilité : versionnage, pagination, erreurs standardisées, disponibilité

**Symptômes en production**
- Un consommateur casse après l'ajout d'un champ obligatoire.
- La pagination par `offset` saute ou duplique des lignes pendant un chargement.
- Les erreurs ont un format différent selon l'endpoint ; le conteneur est « up » mais ne répond pas.

**Prévention (design)**
- Préfixe **`/v1`** ; les changements cassants vont dans `/v2`, avec une période de coexistence et un en-tête `Deprecation`.
- **Pagination par curseur** (clé de tri stable + `id`), `limit` plafonné, curseur opaque encodé.
- Format d'erreur unique **RFC 9457 (Problem Details)** : `type`, `title`, `status`, `detail`, `correlation_id`.
- `/health` (liveness : processus vivant) et `/ready` (readiness : base et JWKS joignables) utilisés par le healthcheck Docker.
- Schéma OpenAPI généré et vérifié en CI (détection de changement cassant par diff).

**Détection**
- Taux d'erreurs 5xx, latence p95 par route, disponibilité mesurée par une sonde externe.

**Remédiation en production**
1. `/ready` en échec : l'API est retirée du trafic ; vérifier la base et Keycloak avant de redémarrer.
2. Régression de contrat : rollback de l'image de l'API (tag précédent).

**Dans le lab** — étape 7 · test : pagination stable pendant un chargement concurrent ; diff OpenAPI en CI.

---

<a id="api-04"></a>

## API-04 · Protection des services : rate limiting, timeouts, volumes sortants

**Symptômes en production**
- Un script client boucle et appelle l'API 50 fois par seconde ; la base sature.
- Un export demande 5 millions de lignes en une seule requête.

**Prévention (design)**
- **Rate limiting par client** (identifiant du jeton) : par ex. 60 req/min pour `viewer`, réponse `429` avec `Retry-After`.
- Taille de page maximale (1 000) ; les exports volumineux passent par un job asynchrone (fichier Parquet à télécharger).
- Timeouts : `statement_timeout` sur le rôle `api_reader`, timeout côté serveur d'application, pool de connexions borné.
- Compression des réponses ; cache des indicateurs peu volatils.

**Détection**
- Métriques `http_requests_total{client,status}` ; alerte sur un client qui concentre plus de 30 % du trafic ; volume d'octets sortants par client.

**Remédiation en production**
1. Client abusif : baisser son quota ou suspendre son compte de service dans Keycloak, le contacter.
2. Saturation : activer temporairement un quota global plus strict, prioriser les rôles critiques.

**Dans le lab** — étape 7 · `make load-test-api` · test : au-delà du quota → 429, la base reste disponible.

---

<a id="api-05"></a>

## API-05 · Gouvernance des accès : traçabilité, isolation des organismes, droits sur les données

**Symptômes en production**
- Un utilisateur d'une filiale voit les commandes d'une autre filiale en modifiant un paramètre `org_id`.
- Impossible de dire qui a consulté les données d'un client donné le mois dernier.

**Causes racines**
- Isolation faite uniquement par un filtre `WHERE` dans le code applicatif.
- `org_id` pris dans la requête au lieu du jeton.

**Prévention (design)**
- L'organisme vient **uniquement du jeton** (claim `org_id` mappé dans Keycloak), jamais d'un paramètre.
- **Row-Level Security Postgres** : à chaque transaction, l'API exécute `SET LOCAL app.org_id = :org_id` ; la politique `USING (org_id = current_setting('app.org_id'))` filtre les lignes même si le code oublie le filtre.
- **Journal d'audit** (`audit.api_access_log`) : utilisateur, rôle, organisme, route, paramètres, nombre de lignes renvoyées, `correlation_id`, horodatage. Table en insertion seule.

**Détection**
- Requêtes d'analyse sur le journal d'audit : accès hors horaires, volume anormal par utilisateur ([GOV-02](06-gouvernance-exploitation.md#gov-02)).
- Test automatisé de non-régression : un utilisateur de l'organisme A ne peut obtenir aucune ligne de B, sur toutes les routes.

**Remédiation en production**
1. Suspicion de fuite entre organismes : extraire du journal d'audit les accès concernés, préserver les preuves ([GOV-05](06-gouvernance-exploitation.md#gov-05)), corriger, notifier selon la procédure.
2. Demande d'un auditeur : requête sur `audit.api_access_log` par personne concernée et période.

**Dans le lab** — étapes 5 et 7 · test : même avec un filtre retiré du code, la RLS empêche l'accès inter-organismes.

---

<a id="api-06"></a>

## API-06 · Conformité : minimisation des données exposées et RGPD

**Symptômes en production**
- L'API renvoie l'objet SQLAlchemy complet, avec des champs internes et des données personnelles inutiles.
- Une demande d'exercice de droits RGPD arrive : personne ne sait où sont les données de la personne.

**Prévention (design)**
- **Schémas de réponse Pydantic explicites** par rôle (`OrderPublic`, `OrderAnalyst`) : seuls les champs nécessaires sortent.
- Données personnelles (contacts Salesforce) pseudonymisées en amont ; champs en clair réservés au rôle `steward`, et journalisés.
- Registre des traitements : finalité, base légale, durée de conservation pour chaque endpoint exposant des données personnelles.
- Classification `pii` des colonnes dans dbt, réutilisée pour générer la liste des endpoints concernés ([GOV-04](06-gouvernance-exploitation.md#gov-04)).

**Détection**
- Test CI : aucun champ classé `pii` dans un schéma de réponse non habilité.

**Remédiation en production**
1. Champ sensible exposé par erreur : retrait immédiat (hotfix du schéma), évaluation via le journal d'audit de qui l'a reçu, escalade au DPO.
2. Demande d'accès ou d'effacement : procédure outillée décrite en [GOV-04](06-gouvernance-exploitation.md#gov-04).

**Dans le lab** — étape 7 · test : le rôle `viewer` ne reçoit jamais de champ classé `pii`.
