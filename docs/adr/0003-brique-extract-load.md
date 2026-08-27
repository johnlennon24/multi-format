# ADR 0003 — Brique d'Extract-Load

- **Statut** : Proposé
- **Date** : 2026-08-27
- **Décideurs** : Repreneur du dépôt ; en attente d'arbitrage du porteur du projet
- **Axes de l'état de l'art concernés** : `02-ingestion-extract-load`, `00-synthese`

## Contexte

Le cadre a besoin d'une brique qui va **réellement chercher la donnée** dans les sources
du SI : bases SQL/NoSQL, APIs REST/SOAP, fichiers structurés (CSV, XML, JSON, Excel,
Parquet) sur SFTP/S3. Aucun SI cible n'étant défini, **le connecteur pour une source
inconnue est le cas nominal**, pas l'exception.

L'ADR 0001 a fixé la frontière décision/exécution. Appliquée à l'ingestion, elle impose
un partage précis : l'agent produit une **configuration déclarative** (source REST,
liste de tables, glob de fichiers), et le moteur d'ingestion — déterministe — gère
l'état, les reprises, le typage et l'écriture. Laisser un LLM écrire à chaque exécution
le code de pagination, de curseur et de reprise serait non déterministe, non testable,
non rejouable — exactement ce que l'ADR 0001 proscrit.

Deux critères de l'axe 2 discriminent alors les candidats :

- **Bibliothèque ou plateforme.** Une bibliothèque Python appelée dans le processus de
  l'agent garde tout dans Git et donne à l'agent le retour d'exécution fin (exceptions,
  retries). Une plateforme ajoute un composant à héberger, une base de métadonnées à
  sauvegarder et à réconcilier avec Git, et réduit l'agent au pilotage par API.
- **Gestion d'état.** Curseurs incrémentaux, idempotence et reprise après échec sont ce
  qu'un agent non déterministe improvise le plus mal : l'état ne doit jamais vivre dans
  le contexte de l'agent.

## Options envisagées

### Option A — dlt, complété par Debezium au besoin

Bibliothèque Python pure (Apache 2.0), sans backend ni serveur, appelable in-process.
Couvre nativement SQL (réflexion SQLAlchemy), REST (déclaratif) et fichiers (S3, GCS,
Azure, SFTP). État incrémental persisté **côté destination** (`_dlt_pipeline_state`),
donc reprise à froid sur runtime éphémère. Contrats de schéma explicites
(`evolve`/`freeze`/`discard_row`) contre le schema drift. Coûte le comblement manuel du
SOAP (via `zeep`) et l'absence de CDC log-based natif. Détail : axe 2.

### Option B — Meltano + Singer SDK

Orchestrateur ELT déclaratif au-dessus de Singer, auto-hébergeable, format de connecteur
standardisé et parc communautaire important. Coûte un risque de gouvernance (projet repris
par Matatika après la fermeture d'Arch, opération récente) et une ergonomie agentique
inférieure : l'agent produit du YAML et pilote des processus séparés plutôt que du code
qu'il instrumente. Détail : axe 2.

Autres candidats (Airbyte, Sling, Estuary Flow, Fivetran, ingestr) : écartés par l'axe 2
pour composant stateful à héberger, licence restrictive (ELv2, BSL, GPL-3.0, FSL-1.1),
absence de cadre de développement de connecteur inédit, ou verrouillage SaaS. Non repris
ici.

## Décision

**Le cadre retient dlt comme moteur unique d'Extract-Load (option A).**

1. **Moteur.** `dlt` (Apache 2.0) est la brique d'extraction et de chargement. On
   n'adopte **pas** le paquet commercial `dlthub` ; les garde-fous que dltHub formalise
   dans son « AI Workbench » (le schéma « propose, verify, enforce ») sont réimplémentés
   dans notre propre cadre.

2. **Ce que l'agent produit.** L'agent produit une **configuration déclarative** de
   pipeline (source REST, tables SQL, glob de fichiers), versionnée en Git. Il ne produit
   du code Python borné que lorsque la source sort du cadre déclaratif. Dans tous les cas,
   `dlt` reste le moteur déterministe qui gère état, reprises, typage et écriture.

3. **État jamais dans l'agent.** L'état de curseur est écrit en destination et rechargé
   automatiquement par `dlt` (write disposition `merge` avec clé primaire). Aucune
   solution où l'agent porterait lui-même le curseur n'est admise.

4. **Schema drift.** Les contrats de schéma `dlt` sont utilisés : `evolve` en découverte,
   `freeze` ou `discard_row` en production, pour qu'une dérive lève une alerte au lieu de
   polluer silencieusement la cible.

5. **SOAP.** Le seul trou de couverture de `dlt` est le SOAP. Il se comble avec `zeep`
   (MIT) en amont d'un `@dlt.resource` — pattern à écrire une fois, réutilisable.

6. **CDC différé.** Debezium Server est **désigné comme le complément CDC prévu** (bases
   transactionnelles, faible latence, capture des DELETE), configurable par fichier
   versionné donc compatible GitOps. Il n'est **pas adopté** tant qu'un SI cible n'a pas
   de besoin CDC réel. Cette adoption sera actée par SI, le cas échéant, sans nouvel ADR
   de principe — l'orientation est fixée ici.

## Justification

`dlt` est retenu parce qu'il est la seule solution du panel qui réunit les critères
imposés par l'ADR 0001 et par le contexte : bibliothèque in-process (pas de composant à
héberger, tout dans Git), licence permissive sans porte dérobée commerciale sur les
fonctions cœur, gestion d'état survivant à un runtime éphémère, et contrôle explicite du
schema drift. Il s'appuie en outre sur des briques bas niveau déjà solides (SQLAlchemy,
connectorx, PyArrow) plutôt que de les réinventer.

Face à Meltano, l'écart décisif est double : l'ergonomie agentique (code instrumentable
contre YAML et processus séparés) et l'assise de gouvernance, plus faible après la
reprise récente du projet. Face aux plateformes (Airbyte, Estuary), l'écart est le
composant stateful à héberger et les licences restrictives, incompatibles avec un cadre
générique et une souveraineté non tranchée.

Debezium est différé parce que le besoin CDC dépend du SI cible, inconnu à ce stade :
l'adopter maintenant engagerait une brique JVM à opérer pour un besoin non constaté, à
rebours d'une V1 volontairement modeste.

## Conséquences

### Ce que ça nous apporte

- Une ingestion entièrement dans Git : configuration versionnée, pipelines Python, CI.
- Une reprise après plantage sans perte ni doublon, sans état dans l'agent.
- Une couverture native de trois des quatre familles de sources.
- Un verrouillage très faible (Apache 2.0, pas de plateforme imposée).

### Ce que ça nous coûte

- L'écriture et la maintenance du pont SOAP (`zeep` → `@dlt.resource`).
- L'absence de CDC tant que Debezium n'est pas adopté : l'extraction se fait par lots, ce
  qui exclut la faible latence et rend la capture des suppressions moins directe.
- La réimplémentation, à notre charge, des garde-fous « propose, verify, enforce » que
  l'offre commerciale `dlthub` fournirait clés en main.

### Ce que ça nous ferme

- Le pilotage par plateforme (Airbyte, Estuary) comme cœur du runtime. Réouvrable en rôle
  de satellite si un SI cible est dominé par des SaaS grand public déjà couverts par un
  catalogue, sans que le cadre cesse d'être piloté par `dlt`.

## Critères de réexamen

Cette décision devra être rediscutée si :

- **`dlt` change de licence** ou déplace des fonctions cœur (SQL, REST, filesystem, état
  incrémental) vers le paquet commercial `dlthub` (critère licence bloquant, cf. synthèse).
- **Un SI cible présente un besoin CDC réel** : déclenche l'adoption de Debezium Server,
  selon l'orientation déjà fixée au point 6.
- **Un SI cible est dominé par des sources SaaS** déjà couvertes par un catalogue de
  connecteurs : déclenche l'évaluation d'Airbyte en satellite.
- **Le modèle de distribution du cadre** (revente, opération pour des tiers) impose une
  validation juridique des licences des briques tactiques (GPL-3.0 de Sling si utilisé).
