# 05 · Infrastructure – Docker / CI/CD / Git

[← Retour à la synthèse](../../README.md#4-synthèse-des-problèmes-par-fonction) · Lab : [étapes 1 et 10](../../README.md#parcours-du-lab)

L'infrastructure doit rendre les déploiements **ennuyeux** : reproductibles, réversibles et vérifiés. Le lab tourne
sous Docker Compose v2 sur Ubuntu ; les choix sont faits pour rester transposables à une production SLES.

| ID | Problème |
|---|---|
| [INF-01](#inf-01) | Déploiement maîtrisé |
| [INF-02](#inf-02) | Résilience des ressources |
| [INF-03](#inf-03) | Sécurité de la chaîne CI/CD |
| [INF-04](#inf-04) | Sauvegarde et reprise (PRA) |
| [INF-05](#inf-05) | Maîtrise des coûts |

---

<a id="inf-01"></a>

## INF-01 · Déploiement maîtrisé : dev/prod, migrations Alembic, tests, rollback

**Symptômes en production**
- « Ça marchait en dev » : versions de Python ou de dépendances différentes.
- Une migration Alembic supprime une colonne encore lue par l'ancienne version de l'API pendant le déploiement.
- Impossible de revenir en arrière : la migration n'a pas de `downgrade`.

**Prévention (design)**
- **Images immuables** construites une fois en CI, taguées par SHA Git, promues de dev à prod (pas de rebuild en prod).
- Configuration par environnement via `.env` / secrets ; un seul `compose.yaml` + fichiers de surcharge `compose.prod.yaml`.
- Dépendances verrouillées (`uv.lock` / `requirements.lock`), version Python figée dans l'image.
- Migrations Alembic en **expand/contract** : 1) ajouter (nouvelle colonne, nullable), 2) déployer le code qui écrit les deux, 3) migrer les données, 4) supprimer l'ancien dans une release ultérieure.
- CI : `alembic upgrade head` → `downgrade -1` → `upgrade head` sur un Postgres éphémère ; tests unitaires, d'intégration et `dbt build`.
- Branche protégée, PR obligatoire avec CI verte, tag de release.

**Détection**
- Vérification au démarrage : l'application refuse de démarrer si la révision Alembic de la base ne correspond pas à celle attendue.
- Smoke tests après déploiement (`/ready`, un appel métier).

**Remédiation en production**
1. Déploiement défectueux : redéployer le tag d'image précédent (`IMAGE_TAG=<sha> docker compose up -d`).
2. Migration défectueuse : `alembic downgrade` testé ; si destructrice, restauration PITR ([STO-05](03-stockage.md#sto-05)).

**Dans le lab** — étape 1 · workflow `ci.yml` · démonstration d'un rollback complet en une commande.

---

<a id="inf-02"></a>

## INF-02 · Résilience : saturation CPU/RAM/disque et redémarrage des services

**Symptômes en production**
- Le worker Airflow consomme toute la RAM pendant un gros chargement ; le noyau tue Postgres (OOM killer).
- Disque plein : les logs Docker et les WAL non archivés ont rempli le volume, Postgres passe en lecture seule.
- Un service redémarre en boucle sans alerte.

**Prévention (design)**
- **Limites de ressources** par service (`mem_limit`, `cpus`) ; Postgres protégé en priorité.
- **Healthchecks** sur chaque service et `depends_on: condition: service_healthy` ; `restart: unless-stopped`.
- Rotation des logs Docker (`max-size`, `max-file`) ; volumes séparés pour les données Postgres et les WAL.
- Ingestion en flux par lots (chunks), pas de chargement complet en mémoire.

**Détection**
- Alertes : disque > 80 % (warning) / > 90 % (critique), RAM des conteneurs, nombre de redémarrages d'un conteneur, archivage WAL en échec.

**Remédiation en production**
1. Disque plein : libérer (logs, artefacts), vérifier l'archivage WAL (cause fréquente) ; **ne jamais supprimer de fichiers WAL à la main**.
2. OOM : identifier le conteneur (`docker inspect` → `OOMKilled`), réduire la taille des lots ou augmenter la limite.
3. Redémarrages en boucle : `docker compose logs --tail 200 <service>`, vérifier les dépendances (base, Keycloak).

**Dans le lab** — étapes 1 et 10 · `make incident-disk-full`, `make incident-oom` · runbooks associés.

---

<a id="inf-03"></a>

## INF-03 · Sécurité : secrets, vulnérabilités, dépendances, chaîne CI/CD

**Symptômes en production**
- Un token commité dans le dépôt il y a six mois.
- Une image de base avec une CVE critique non corrigée.
- Une action GitHub tierce compromise exfiltre les secrets de la CI.

**Prévention (design)**
- **gitleaks** en pre-commit et en CI ; `.env` dans `.gitignore`, `.env.example` sans valeurs.
- **pip-audit** (dépendances Python), **trivy** (images et configuration), Dependabot.
- Images de base minimales et épinglées par digest, conteneurs en utilisateur non root.
- Actions GitHub épinglées par SHA, `permissions:` minimales par workflow, secrets par environnement GitHub avec approbation pour la prod.

**Détection**
- Échec de la CI sur toute CVE critique ou tout secret détecté ; scan hebdomadaire planifié des images en production.

**Remédiation en production**
1. Secret commité : **rotation d'abord** (le secret est considéré comme compromis), puis nettoyage de l'historique si nécessaire.
2. CVE critique : mise à jour de l'image de base, rebuild, redéploiement par la chaîne standard.

**Dans le lab** — étape 1 · workflow `security.yml` · PR de démonstration contenant un faux secret, bloquée par la CI.

---

<a id="inf-04"></a>

## INF-04 · Sauvegarde et reprise : PRA, RPO/RTO, tests de restauration

**Symptômes en production**
- Le serveur de production est perdu ; le PRA existe sur papier mais n'a jamais été joué.
- Les RPO/RTO promis (1 h / 4 h) ne sont pas tenables avec le dispositif réel.

**Prévention (design)**
- Objectifs explicites par composant :

| Composant | RPO cible | RTO cible | Moyen |
|---|---|---|---|
| PostgreSQL | 15 min | 2 h | Sauvegarde physique + archivage WAL (PITR) |
| Realm Keycloak | 24 h | 1 h | Export du realm versionné + base Keycloak sauvegardée |
| Configuration / code | 0 | 30 min | Git + images taguées dans le registre |
| DuckDB / marts | n/a | 30 min | Reconstruits depuis Postgres |

- Infrastructure décrite dans le dépôt (Compose, configuration Keycloak, dashboards Grafana) : un environnement complet se reconstruit depuis Git.

**Détection**
- Exercice de PRA **planifié et chronométré** ; la mesure réelle est comparée aux objectifs.

**Remédiation en production (runbook PRA)**
1. Provisionner un hôte, cloner le dépôt, récupérer les secrets.
2. Restaurer Postgres (sauvegarde + WAL jusqu'au dernier point disponible).
3. Démarrer les services (`docker compose --profile core --profile security up -d`), vérifier `/ready`.
4. Reprendre les ingestions depuis les watermarks ; rebuild dbt et DuckDB.
5. Consigner le RPO/RTO réels et les écarts dans un post-mortem.

**Dans le lab** — étape 10 · `make drill-dr` · rapport d'exercice dans `docs/runbooks/`.

---

<a id="inf-05"></a>

## INF-05 · Maîtrise des coûts : ressources, stockage, optimisation des traitements

**Symptômes en production**
- Le volume de stockage double chaque trimestre sans que les usages augmentent.
- Le `dbt build` complet tourne toutes les heures alors que les sources changent une fois par jour.

**Prévention (design)**
- **Profiles Docker Compose** : on ne démarre que les services utiles.
- Modèles **incrémentaux** pour les gros faits ; planification alignée sur la fréquence réelle des sources (déclenchement par assets Airflow plutôt que cron fixe).
- Rétention et archivage Parquet compressé ([STO-03](03-stockage.md#sto-03)) ; nettoyage des artefacts CI et images orphelines.
- Mesure systématique de la durée et des volumes par run ([GOV-06](06-gouvernance-exploitation.md#gov-06)).

**Détection**
- Croissance du stockage par schéma ; durée des builds dbt par modèle (`run_results.json`).

**Remédiation en production**
1. Identifier les 10 modèles les plus coûteux, passer en incrémental ou revoir la matérialisation (`view` vs `table`).
2. Supprimer les datasets sans consommateur (vérifié via `exposures` et le journal d'audit).

**Dans le lab** — étapes 1 et 9 · tableau de bord des coûts par pipeline.
