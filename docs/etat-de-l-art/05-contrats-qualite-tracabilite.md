# Axe 5 — Contrats de données, qualité et traçabilité

> État vérifié en août 2026. Les versions et dates ci-dessous proviennent de PyPI, des dépôts GitHub et des sites officiels consultés (voir Sources). Ce qui n'a pas pu être vérifié est signalé explicitement.

## Périmètre et enjeu pour le projet

Le cadre extrait de la donnée depuis des sources hétérogènes (SQL/NoSQL, REST/SOAP, fichiers structurés, documents non structurés) via des agents LLM exécutés en production. Cet axe couvre trois fonctions distinctes, souvent confondues :

1. **Le contrat** : la définition déclarative, versionnée en Git, de ce que la donnée doit être (schéma, types, cardinalités, règles métier, SLA).
2. **La qualité** : l'exécution effective de ces règles contre des données réelles, avec un verdict bloquant ou non.
3. **La traçabilité** : la capacité à reconstituer a posteriori d'où vient une valeur, par quel traitement, avec quel modèle et quelle version de prompt.

Ces trois fonctions sont assurées par des outils différents. Un point structurant relevé dans la littérature 2026 (article zircote, avril 2026) : la majorité des outils dits « data contract » sont **définitionnels** (ils stockent un YAML dans un catalogue) et non **applicatifs** (ils ne bloquent rien au runtime). La question à poser pour chaque outil n'est pas « supporte-t-il les contrats ? » mais « quel composant refuse concrètement la donnée non conforme, et à quel moment ? ».

## Pourquoi le contrat de données est le garde-fou du runtime agentique

Un pipeline ETL classique est déterministe : le même code sur la même entrée produit la même sortie. On peut donc raisonner par tests unitaires sur le code. Un runtime agentique casse cette propriété : la sortie dépend du modèle, de sa version, de la température, du prompt, et du contenu du document source. Trois conséquences opérationnelles :

- **Le test du code ne suffit plus.** On ne peut pas garantir la conformité par revue de code ; on ne peut la garantir que par **validation de la sortie à chaque exécution**. La validation cesse d'être un filet de sécurité périodique pour devenir un composant du chemin d'exécution nominal.
- **Le contrat devient l'interface de l'agent.** Le schéma cible n'est plus seulement une contrainte de chargement : c'est ce qu'on injecte dans le prompt (ou dans le décodage contraint), ce qui gouverne le retry, et ce qui décide de l'acceptation. Un même artefact déclaratif sert donc trois usages : générer, valider, documenter. C'est l'argument le plus fort pour un format unique versionné en Git plutôt que des règles éparpillées dans le code.
- **Les modes de défaillance changent de nature.** Un ETL classique échoue bruyamment (exception, type invalide). Un agent échoue silencieusement et plausiblement : il hallucine une valeur bien typée, il confond deux colonnes, il invente un montant cohérent mais faux. Un contrat qui ne valide que le **schéma** (types, présence) ne détecte rien de tout cela. Il faut des règles sur les **valeurs** (plages, référentiels, cohérences croisées, réconciliation avec la source) et une **traçabilité par valeur** permettant l'audit humain.

Corollaire de conception : le seul contrat qui a du sens ici est un contrat **exécutable et bloquant**, placé entre l'agent et la cible, avec une voie de sortie explicite (quarantaine) pour ce qu'il refuse. Un contrat purement documentaire déplace le risque sans le réduire.

## Panorama des solutions

### Open Data Contract Standard (ODCS) — Bitol / LF AI & Data

Spécification YAML de contrat de données, Apache 2.0, projet en incubation LF AI & Data. Version courante **v3.1.0 (8 décembre 2025)** ; le dépôt reste actif en 2026 mais aucune v3.2/v4 n'est annoncée sur la page consultée. Onze sections : fundamentals, schema, references, data quality, support, pricing, team, roles, SLA, servers, custom properties. La section qualité distingue quatre types de règles : `text` (descriptive), `library` (métriques normalisées : `nullValues`, `missingValues`, `invalidValues`, `duplicateValues`, `rowCount`…), `sql` (requête renvoyant un scalaire, avec substitution `{object}`/`{property}`), `custom` (payload propre à un moteur : Soda, GX, dbt…). Sept dimensions de qualité et huit opérateurs (`mustBe`, `mustBeBetween`, `mustBeGreaterThan`…). Fait décisif de consolidation : la **Data Contract Specification** de datacontract.com a été **dépréciée** avec l'arrivée d'ODCS v3.1.0 et contribuée à Bitol ; sa prise en charge est maintenue jusqu'à fin 2026. Il n'existe donc plus de standard concurrent crédible.

### Data Contract CLI (datacontract-cli)

Outil Python, **licence MIT**, version **1.1.2 (26 août 2026)**, ~1,4 M téléchargements/mois. Commandes : `init`, `lint`, `test`, `changelog`, `export`, `import`, `dbt`, `ci`, `catalog`, `publish`, `api`. Export vers 25+ formats (JSON Schema, Avro, dbt models/sources, SQL, SodaCL, RDF, Terraform, HTML). ODCS v3.1 est le format par défaut depuis les 1.x. Utilisable comme **bibliothèque Python** (`DataContract(...).test()`), ce qui permet de l'appeler depuis le runtime agentique et pas seulement en CI. Point important vérifié dans le `pyproject.toml` de la branche `main` : les 1.x **ne dépendent plus ni de soda-core ni de great-expectations** ; les tests s'exécutent nativement via `duckdb`, `sqlglot`, `ibis-framework` et `fastjsonschema`. La v1.1.0 a également retiré PySpark des dépendances de compilation (image Docker 777 Mo → 277 Mo). Détection des changements de schéma via `changelog`.

### Great Expectations (GX Core)

Framework de validation Python, **Apache 2.0** (confirmé sur la branche `develop`), version **1.21.0 (19 août 2026)**. Changement de gouvernance majeur : **Fivetran est devenu steward du projet le 13 mai 2026** ; le dépôt est désormais `github.com/fivetran/great_expectations`. L'annonce promet un maintien en open source communautaire mais ne détaille pas d'engagement formel de licence — à surveiller. La v1.21.0 introduit un « SQL Harness Backend Framework », Trino et ClickHouse, et des « agent-skill guidance » pour la configuration. Modèle mental centré sur les *Expectation Suites* et les *Data Docs*. Riche en assertions, mais lourd en configuration et principalement orienté dataframes/tables ; ce n'est pas un format de contrat au sens ODCS, et il n'est pas trivialement versionnable comme un artefact déclaratif unique.

### Soda Core (v4)

Moteur de qualité YAML (SodaCL), se présentant en v4 comme un « data contracts engine ». Version PyPI **4.22.0 (25 août 2026)**. **Alerte licence, vérifiée sur trois sources concordantes** : le fichier `LICENSE` de la branche `main` est l'**Elastic License 2.0**, tandis que celui de la branche `v3` est Apache 2.0, et la fiche PyPI de `soda-core` 4.22.0 déclare la licence « Proprietary ». Autrement dit, **Soda Core n'est plus open source au sens OSI à partir de la v4** ; seule la lignée v3 l'est. ELv2 interdit notamment de fournir le logiciel comme service hébergé/managé. Par ailleurs Soda pousse une couche IA (Soda AI CLI, Contract Copilot, serveur MCP, février 2026) qui pointe vers Soda Cloud. Pour un cadre réutilisable revendu ou hébergé pour des tiers, ce point de licence est disqualifiant tant qu'il n'est pas arbitré juridiquement.

### dbt (tests) et l'écosystème dbt-natif

`dbt-core` **1.12.3 (21 août 2026)**, **Apache 2.0**, avec une **2.0.0b2** en pré-version. Les tests dbt (`not_null`, `unique`, `accepted_values`, `relationships`, plus dbt-expectations) sont un mécanisme de qualité déclaratif éprouvé, versionné en Git, mais **post-chargement** : ils s'exécutent sur des tables déjà écrites dans l'entrepôt. Le moteur **dbt Fusion** (réécriture Rust) est sous **Elastic License v2** pour l'essentiel et reste marqué **Beta** ; le dépôt `dbt-labs/dbt-fusion` est archivé pour le code source, le suivi ayant migré vers `dbt-core`. **Elementary** (Apache 2.0, actif) ajoute par-dessus dbt la détection d'anomalies (fraîcheur, volume, schéma), le lignage enrichi des résultats de tests et l'alerting. Pertinent seulement si une couche dbt existe dans la cible — ce qui n'est pas acquis dans le projet.

### Pydantic et Pandera (couche de contrat en Python)

**Pydantic 2.13.4 (6 mai 2026)** : validation par annotations de types, cœur en Rust, standard de fait pour la sortie structurée de LLM. **Pandera 0.32.1 (29 juin 2026, MIT, maintenu par unionai-oss)** : validation de dataframes, avec un éventail de backends inhabituellement large (pandas, Polars, PySpark, Ibis, Dask, Modin, Narwhals, GeoPandas). **Instructor 1.15.4 (28 juin 2026, MIT)** : couche au-dessus des SDK LLM qui valide la sortie contre un modèle Pydantic et **relance automatiquement** l'appel avec l'erreur de validation en contexte — c'est exactement la boucle de retry dont un runtime agentique a besoin. Ces trois briques opèrent au niveau objet/dataframe en mémoire, avant tout accès à la cible. Elles ne sont pas des contrats partageables entre producteur et consommateur : ce sont des implémentations. Le pont existe dans les deux sens (datacontract-cli sait importer un modèle Pydantic depuis la v1.1.1, et exporter du JSON Schema).

### Schémas de sérialisation et registries (Avro, Protobuf, JSON Schema)

**JSON Schema** : le draft **2020-12** reste la version courante en 2026, aucune version ultérieure publiée sur la page consultée. C'est le pivot d'interopérabilité naturel (ODCS et datacontract-cli exportent vers lui, tous les SDK LLM l'acceptent pour la sortie structurée). Côté registries : **Confluent Schema Registry** est sous **double licence** — Confluent Community License pour le serveur, Apache 2.0 pour les clients/serdes — et la validation *broker-side* (la seule réellement contraignante) est une fonction Enterprise payante ; la validation côté client relève de la bonne volonté du producteur. Alternatives réellement libres : **Apicurio Registry** (Apache 2.0, projet **CNCF Sandbox**, Avro/Protobuf/JSON Schema, règles de compatibilité, opérateur Kubernetes) et **Karapace** (Apache 2.0, Aiven, compatible API Schema Registry 6.1.1, FastAPI, OpenTelemetry). Pertinent seulement si un bus Kafka entre dans l'architecture.

### OpenLineage et Marquez (traçabilité)

**OpenLineage** : spécification et clients pour l'émission d'événements de lignage, **Apache 2.0**, projet **graduated LF AI & Data**, client Python **1.52.0 (23 juillet 2026)** — cadence de release soutenue (1.50.0 en juin, 1.51.0 en juillet). Modèle à facettes : *run facets*, *job facets*, *dataset facets*, dont `schema`, `dataSource`, `dataQualityMetrics`, `dataQualityAssertions`, `columnLineage`, `ownership`, `tags`. La facette `columnLineage` décrit, par colonne de sortie, les `inputFields` sources et le type de transformation (`DIRECT`/`INDIRECT`, sous-types `IDENTITY`, `TRANSFORMATION`, `AGGREGATION`, `JOIN`, `FILTER`…) plus un indicateur de masquage. **Les facettes custom sont supportées** (convention `{prefix}_{name}`), ce qui est le point d'accroche pour le cas agentique. Intégrations listées : Airflow, Spark, Flink, dbt, Trino, Great Expectations. **Aucune convention GenAI/agent n'est mentionnée dans la documentation OpenLineage consultée** : c'est le manque principal. **Marquez**, implémentation de référence (Apache 2.0, graduated LF AI & Data), présente un signal ambigu : dernière release **0.50.0 en octobre 2024**, mais commits toujours présents sur `main` (le plus récent consulté : **12 avril 2026**). À traiter comme un projet en maintenance lente, pas comme une brique de production sans reprise.

### DataHub et OpenMetadata (catalogue)

**DataHub** : `acryl-datahub` **1.7.0.7 (26 août 2026)**, **Apache 2.0**, 50+ connecteurs. La v1.7.0 ajoute le support des contrats **ODCS** chargés depuis S3, GCS, HTTP et **Git**, des entités `metric`/`semanticModel` de première classe, et une gestion plus fine de la sévérité des assertions. Réserve importante : DataHub OSS **stocke** les objets contrat mais n'exécute pas les assertions ; il faut brancher un runner externe. **OpenMetadata** : **2.0.0 (24 août 2026)**, 130+ connecteurs, échantillonnage dynamique par défaut dans le profiler, Context Center, **serveur MCP activé par défaut** exposant le catalogue aux assistants/agents, et support annoncé des standards ouverts DCAT, PROV-O, OpenLineage et ODCS. **Point de licence à revérifier** : le dépôt GitHub principal affiche Apache 2.0, mais le paquet PyPI `openmetadata-ingestion` **2.0.0.0** déclare une « Collate Community License Agreement ». La divergence est réelle sur les sources consultées et doit être tranchée avant tout engagement.

## Grille comparative

| Solution | Licence | Auto-hébergeable | Déclaratif versionnable | Portée (schéma / valeurs / lignage) | Effort d'intégration | Maturité | Adapté au cadre ? |
|---|---|---|---|---|---|---|---|
| ODCS (spéc. Bitol) | Apache 2.0 | s.o. (spécification) | Oui — YAML, cœur de la démarche | Schéma + valeurs (décl.) ; pas de lignage | Faible (rédaction) | v3.1.0, LF AI & Data incubation | **Oui — socle** |
| Data Contract CLI | MIT | Oui (CLI, lib Python, Docker) | Oui (consomme ODCS) | Schéma + valeurs + détection de rupture | Faible à moyen | v1.1.2, ~1,4 M dl/mois | **Oui — moteur** |
| Great Expectations (GX Core) | Apache 2.0 | Oui | Partiellement (config lourde, pas un contrat portable) | Valeurs (assertions riches) | Moyen à élevé | v1.21.0 ; steward Fivetran depuis 05/2026 | Complément optionnel |
| Soda Core v4 | **Elastic License 2.0** (v3 : Apache 2.0) | Oui, mais ELv2 interdit l'offre managée | Oui (SodaCL YAML) | Schéma + valeurs | Moyen | v4.22.0, actif | **Non** (licence) |
| dbt tests + Elementary | Apache 2.0 (dbt Fusion : ELv2, beta) | Oui | Oui (YAML dbt) | Valeurs, post-chargement ; anomalies | Élevé (impose dbt) | dbt-core 1.12.3 ; 2.0 en beta | Conditionnel (si dbt en cible) |
| Pydantic / Pandera / Instructor | MIT | Oui (bibliothèques) | Non (code, pas contrat portable) | Schéma + valeurs, en mémoire | Faible | Très mature, très actif | **Oui — exécution** |
| JSON Schema + Apicurio / Karapace | JSON Schema : spéc. ouverte ; registries : Apache 2.0 | Oui | Oui | Schéma + compatibilité de versions | Moyen (registry à opérer) | 2020-12 ; Apicurio CNCF Sandbox | Si bus/streaming |
| Confluent Schema Registry | Confluent Community (serveur) / Apache 2.0 (clients) | Oui, mais enforcement broker = Enterprise | Oui | Schéma | Moyen | Mature | Non (licence + enforcement payant) |
| OpenLineage | Apache 2.0 | Oui (client + collecteur) | Partiellement (événements, pas config) | Lignage (dont colonne) + facettes qualité | Moyen | v1.52.0, LF graduated | **Oui — traçabilité** |
| Marquez | Apache 2.0 | Oui | Non | Stockage/visualisation du lignage | Moyen | Pas de release depuis 10/2024 ; commits jusqu'à 04/2026 | Réserve — maintenance lente |
| DataHub | Apache 2.0 | Oui | Consomme ODCS depuis Git | Catalogue + lignage colonne + assertions (runner externe) | Élevé | v1.7.x, actif | Phase 2 |
| OpenMetadata | Dépôt : Apache 2.0 ; paquet ingestion 2.0 : **licence à revérifier** | Oui | Consomme ODCS | Catalogue + qualité + lignage colonne + MCP | Élevé | v2.0.0 (08/2026) | Phase 2, sous réserve licence |

## Traçabilité de bout en bout pour une extraction agentique

Aucun outil consulté ne couvre nativement la chaîne complète pour le cas agentique. La proposition est un **enregistrement de provenance par valeur extraite**, porté par une facette OpenLineage custom (mécanisme officiellement prévu, préfixe `{prefix}_{name}`), et persisté à côté de la donnée métier :

| Étape | Métadonnées à capturer | Avec quoi |
|---|---|---|
| Acquisition | URI de la source, hash du document/enregistrement, horodatage, version du connecteur | Événement OpenLineage `START`, facette `dataSource` |
| Ancrage (documents) | Page, offset ou bbox du texte source, extrait littéral (*evidence span*) | Facette custom — **rien de standard ne couvre cela** |
| Inférence | Identifiant et version du modèle, hash du prompt et sa référence Git, température, tokens, latence, identifiant de trace | Conventions sémantiques **OpenTelemetry GenAI** (dépôt `semantic-conventions-genai`, Apache 2.0, **statut « development »**, non stable) — à corréler par `trace_id` avec l'événement OpenLineage |
| Validation | Version du contrat ODCS appliquée, règles évaluées, verdict, écarts | Facettes `dataQualityAssertions` / `dataQualityMetrics` d'OpenLineage, alimentées par le résultat de `datacontract test` |
| Transformation | Lignage colonne-à-colonne, type de transformation | Facette `columnLineage` (`DIRECT`/`INDIRECT` + sous-types) |
| Chargement | Cible, clé technique, décision (accepté / quarantaine), identifiant de rejeu | Événement OpenLineage `COMPLETE`/`FAIL` + table de quarantaine |

Ce qui manque au standard, et qu'il faut donc spécifier soi-même : **l'ancrage source→valeur** (page/offset), **la version de prompt comme entité versionnée de premier rang**, et **le score de confiance par champ**. La corrélation OpenLineage ↔ OpenTelemetry GenAI n'est pas normalisée : elle repose sur un identifiant de corrélation propagé manuellement. Le fait que les conventions GenAI d'OTel soient encore en statut *development* impose de considérer ce point comme instable sur 12 à 18 mois.

## Ce que ça implique pour un runtime agentique

- **Validation en trois points, pas un.** (1) *À l'extraction* : contrainte de la sortie du LLM par JSON Schema dérivé du contrat, puis validation Pydantic avec retry automatique borné (motif Instructor). (2) *Après transformation* : règles métier, référentiels, cohérences croisées, via `datacontract test`. (3) *Au chargement* : contrainte structurelle en base (types, clés, contraintes d'intégrité), qui est la seule barrière qu'un bug applicatif ne peut pas contourner. Les trois sont nécessaires et ne détectent pas les mêmes fautes.
- **Quarantaine plutôt que rejet.** Tout enregistrement non conforme part dans une table de quarantaine avec sa charge utile brute, le verdict détaillé, le contexte d'inférence complet et un statut. Le rejeu doit être **idempotent** (clé déterministe dérivée de source + hash) et permettre trois issues : rejeu automatique (modèle ou prompt corrigé), correction humaine, abandon documenté. Ne jamais écraser en place : versionner les tentatives.
- **Un seul artefact déclaratif.** Le contrat ODCS en Git est la source unique dont on dérive le JSON Schema du prompt, les tests de qualité, le DDL de la cible et la documentation. Toute divergence entre ces artefacts est un bug de génération, pas une décision.
- **Dérive : deux phénomènes distincts.** Dérive *de source* (le schéma amont change) — détectable par `datacontract changelog`/`lint` sur un schéma réimporté périodiquement, en tâche planifiée. Dérive *de modèle* (même entrée, sortie différente entre deux versions) — non couverte par les outils data classiques ; nécessite un **jeu de régression figé** (documents de référence + sorties attendues) rejoué en CI à chaque changement de modèle ou de prompt, plus un suivi statistique en production (taux de rejet, taux de champs nuls, distribution des valeurs par version de modèle).
- **Réconciliation source↔cible.** Comptages par lot, sommes de contrôle sur les colonnes numériques, échantillonnage aléatoire relu par un humain, et unicité des clés. `data-diff` de Datafold, souvent cité pour cet usage, est **archivé depuis le 17 mai 2024** : il ne faut pas bâtir dessus. À réimplémenter par requêtes de contrôle exprimées comme règles `sql` du contrat ODCS.

## Recommandation

**Option 1 (recommandée) — ODCS + Data Contract CLI + Pydantic/Instructor + OpenLineage.** Le contrat ODCS versionné en Git est l'artefact pivot ; `datacontract-cli` (MIT, appelable comme bibliothèque Python) l'exporte en JSON Schema pour contraindre l'agent et l'exécute comme suite de tests ; Pydantic + Instructor assurent la validation et le retry en mémoire, avant tout accès à la cible ; OpenLineage émet le lignage avec une facette custom portant les métadonnées d'inférence. Toute la chaîne est MIT/Apache 2.0, auto-hébergeable, sans dépendance à un SaaS, et cohérente avec Python et GitOps. Coût principal : la facette de provenance agentique et la table de quarantaine sont à écrire.

**Option 2 — Option 1 + catalogue (DataHub) en phase 2.** À ouvrir seulement quand plusieurs connecteurs et plusieurs consommateurs coexistent. DataHub 1.7 lit les contrats ODCS directement depuis Git, ce qui préserve le modèle GitOps, et consomme OpenLineage. Réserve assumée : DataHub OSS n'exécute pas les assertions ; c'est un plan de visualisation et de gouvernance, pas un point d'application. OpenMetadata 2.0 est fonctionnellement plus riche (qualité intégrée, MCP natif, PROV-O) mais **son statut de licence pour la partie ingestion doit être tranché** avant tout engagement.

**Option 3 — Adossement à dbt.** Uniquement si la cible est un entrepôt et qu'une couche dbt est de toute façon prévue : tests dbt + Elementary + intégration OpenLineage native. Écartée par défaut ici car elle impose un modèle d'exécution (SQL post-chargement dans un entrepôt) que le projet n'a pas arbitré, et parce que le moteur Fusion est en beta sous ELv2.

**Ce que j'écarte explicitement :**
- **Soda Core v4** — l'Elastic License 2.0 sur la branche `main`, la mention « Proprietary » sur PyPI et l'interdiction d'offre managée sont incompatibles avec un cadre réutilisable potentiellement exploité pour des tiers. Rester en v3 (Apache 2.0) revient à s'adosser à une lignée qui n'est plus la principale. Signal corroborant : `datacontract-cli` a retiré `soda-core` de ses dépendances dans les 1.x.
- **Data Contract Specification (datacontract.com)** — dépréciée au profit d'ODCS, support jusqu'à fin 2026. Aucune raison de démarrer dessus.
- **Confluent Schema Registry** — licence Confluent Community sur le serveur et validation broker-side réservée à l'offre Enterprise ; Apicurio (CNCF Sandbox) ou Karapace si un besoin de registry apparaît.
- **`data-diff`** — archivé depuis mai 2024.
- **Great Expectations comme socle** — pas écarté comme complément, mais écarté comme pivot : ce n'est pas un format de contrat portable, la configuration est lourde, et le changement de steward vers Fivetran (mai 2026) est trop récent pour être considéré comme stabilisé.
- **Marquez comme brique de production** — pas de release depuis octobre 2024 malgré des commits en 2026. Émettre du OpenLineage reste pertinent ; le consommateur peut être un simple collecteur maison puis DataHub, sans engager Marquez.

**Points de décision restant ouverts :** (a) statut de licence exact d'OpenMetadata 2.0 côté ingestion ; (b) où stocker la provenance par valeur — table dédiée dans la cible, ou uniquement dans le backend de lignage ; (c) le seuil de rejet en quarantaine (strict, ou dégradé avec score de confiance) est un arbitrage métier, pas technique ; (d) la dépendance aux conventions OpenTelemetry GenAI, encore en statut *development*.

## Sources

- https://bitol.io/ — site du projet Bitol, statut LF AI & Data (incubation), ODCS v3 et ODPS v1.0.0.
- https://github.com/bitol-io/open-data-contract-standard — dépôt ODCS : Apache 2.0, v3.1.0, activité 2026, pas de v3.2/v4 annoncée.
- https://github.com/bitol-io/open-data-contract-standard/blob/main/CHANGELOG.md — historique des versions : v3.1.0 (2025-12-08), v3.0.2, v3.0.1, v3.0.0.
- https://bitol-io.github.io/open-data-contract-standard/latest/ — les 11 sections d'un contrat ODCS.
- https://bitol-io.github.io/open-data-contract-standard/latest/data-quality/ — types de règles (text/library/sql/custom), métriques normalisées, dimensions, opérateurs.
- https://github.com/bitol-io/open-data-contract-standard/blob/main/vendors.md — liste des outils déclarant un support ODCS (DataHub, IBM watsonx, Databricks Ontos, ContractGate, DataVow, vowl…).
- https://pypi.org/project/datacontract-cli/ — v1.1.2 (26/08/2026), MIT, ~1,4 M téléchargements/mois, capacités.
- https://github.com/datacontract/datacontract-cli/releases — historique 2026 : v1.1.0 (retrait PySpark), v1.1.1 (import Pydantic), v1.1.2 (DuckDB).
- https://raw.githubusercontent.com/datacontract/datacontract-cli/main/pyproject.toml — dépendances réelles : duckdb, sqlglot, ibis, fastjsonschema, open-data-contract-standard ; **ni soda-core ni great-expectations**.
- https://cli.datacontract.com/ — commandes disponibles, 25+ formats d'export, usage comme bibliothèque Python, licence MIT.
- https://pypi.org/project/open-data-contract-standard/ — modèle Pydantic d'ODCS, v3.1.2 (17/12/2025), MIT.
- https://zircote.com/blog/2026/04/most-data-contract-tools-dont-enforce-contracts/ — analyse critique (avril 2026) : distinction contrat définitionnel vs applicatif, limites de DataHub OSS et de Confluent.
- https://markets.financialcontent.com/stocks/article/bizwire-2026-5-13-fivetran-to-become-steward-of-the-great-expectations-open-source-community-and-gx-core-project — annonce du 13/05/2026 : Fivetran steward de GX Core.
- https://github.com/great-expectations/great_expectations/releases — redirige vers `fivetran/great_expectations` ; v1.21.0 (19/08/2026).
- https://raw.githubusercontent.com/fivetran/great_expectations/develop/LICENSE — GX Core toujours Apache 2.0 après le transfert.
- https://github.com/sodadata/soda-core — Soda Core v4 présenté comme « data contracts engine ».
- https://raw.githubusercontent.com/sodadata/soda-core/main/LICENSE — **Elastic License 2.0** sur la branche principale.
- https://raw.githubusercontent.com/sodadata/soda-core/v3/LICENSE — Apache 2.0 sur la branche v3 (preuve du changement de licence).
- https://pypi.org/project/soda-core/ — v4.22.0 (25/08/2026), licence déclarée « Proprietary ».
- https://soda.io/blog/introducing-soda-ai-cli — Soda AI CLI, Contract Copilot, serveur MCP (février 2026).
- https://pypi.org/project/dbt-core/ — v1.12.3 (21/08/2026), Apache 2.0, pré-version 2.0.0b2.
- https://github.com/dbt-labs/dbt-fusion — moteur Rust, majoritairement ELv2, **statut beta**, dépôt archivé pour le code source.
- https://github.com/elementary-data/elementary — observabilité dbt-native, Apache 2.0, anomalies fraîcheur/volume/schéma.
- https://pypi.org/project/pandera/ — v0.32.1 (29/06/2026), MIT, unionai-oss, backends pandas/Polars/PySpark/Ibis/Narwhals.
- https://pypi.org/project/pydantic/ — v2.13.4 (06/05/2026).
- https://pypi.org/project/instructor/ — v1.15.4 (28/06/2026), MIT, validation Pydantic et retry automatique de la sortie LLM.
- https://github.com/guardrails-ai/guardrails — Apache 2.0, actif ; migration des validators vers PyPI et arrêt du service d'inférence hébergé le 25/08/2026.
- https://json-schema.org/specification — draft 2020-12 toujours la version courante en 2026.
- https://github.com/confluentinc/schema-registry — double licence Confluent Community / Apache 2.0 ; Avro, Protobuf, JSON Schema.
- https://github.com/Apicurio/apicurio-registry — Apache 2.0, projet CNCF Sandbox, opérateur Kubernetes.
- https://github.com/Aiven-Open/karapace — Apache 2.0, compatible API Schema Registry 6.1.1, OpenTelemetry.
- https://pypi.org/project/openlineage-python/ — client v1.52.0 (23/07/2026), Apache 2.0.
- https://github.com/OpenLineage/OpenLineage/releases — cadence 2026 : 1.50.0 (18/06), 1.51.0 (09/07), 1.52.0 (23/07).
- https://openlineage.io/docs/ — statut « graduated » LF AI & Data ; intégrations listées ; aucune mention GenAI/agents.
- https://openlineage.io/docs/spec/facets/ — modèle à facettes et convention de nommage des facettes custom.
- https://openlineage.io/docs/spec/facets/dataset-facets/ — facettes standard dont dataQualityMetrics, dataQualityAssertions, columnLineage.
- https://openlineage.io/docs/spec/facets/dataset-facets/column_lineage_facet — structure de columnLineage : inputFields, type DIRECT/INDIRECT, sous-types, masking.
- https://github.com/MarquezProject/marquez — Apache 2.0, LF AI & Data graduated, aucun avis d'archivage.
- https://github.com/MarquezProject/marquez/releases — dernière release 0.50.0 (24/10/2024).
- https://github.com/MarquezProject/marquez/commits/main — commits jusqu'au 12/04/2026 malgré l'absence de release.
- https://pypi.org/project/acryl-datahub/ — v1.7.0.7 (26/08/2026), Apache 2.0, 50+ connecteurs.
- https://github.com/datahub-project/datahub/releases — v1.7.0 : support ODCS depuis S3/GCS/HTTP/Git, entités metric/semanticModel, sévérité des assertions.
- https://github.com/open-metadata/OpenMetadata — dépôt affiché en Apache 2.0 ; support annoncé de DCAT, PROV-O, OpenLineage, ODCS.
- https://pypi.org/project/openmetadata-ingestion/ — v2.0.0.0 (24/08/2026), licence déclarée « Collate Community License Agreement » (divergence avec le dépôt, à trancher).
- https://github.com/open-metadata/OpenMetadata/releases/tag/2.0.0-release — contenu de la 2.0 : échantillonnage dynamique, Context Center, serveur MCP activé par défaut.
- https://github.com/open-telemetry/semantic-conventions-genai — conventions GenAI d'OpenTelemetry, Apache 2.0, **statut « development »** (spans agent/tool, métriques tokens, contenu des prompts).
- https://github.com/datafold/data-diff — **archivé le 17/05/2024**, MIT, plus de développement.
