# Axe 2 — Ingestion et Extract-Load

> État vérifié en août 2026. Les versions et dates ci-dessous proviennent des pages listées en fin de document (PyPI, GitHub, docs éditeurs). Quand une information n'a pas pu être confirmée à la source, c'est écrit explicitement.

## Périmètre et enjeu pour le projet

Cet axe couvre la brique qui va **réellement chercher la donnée** dans les sources du SI : bases SQL/NoSQL, APIs REST/SOAP, fichiers structurés (CSV/XML/JSON/Excel/Parquet) déposés sur SFTP/S3, documents non structurés. Le reste du cadre (orchestration, transformation, qualité) est traité dans les autres axes.

L'enjeu structurant est particulier : le client a acté un **runtime agentique**, où des agents LLM exécutent l'extraction en production. Cela déplace le critère de choix. La question n'est plus « quel outil a le plus de connecteurs » mais **« qu'est-ce que l'agent manipule concrètement à l'exécution »** :

- une **bibliothèque Python** que l'agent importe et appelle dans son propre processus (il écrit ou paramètre du code, il voit les exceptions, il peut réessayer) ;
- ou une **plateforme** avec serveur, base de métadonnées et UI, que l'agent doit piloter par API REST (il crée une source, une connexion, déclenche un job, sonde un statut).

La seconde option ajoute un composant stateful à exploiter, à sauvegarder et à faire vivre en GitOps, et elle rend l'agent dépendant d'un modèle d'objets externe. La première garde tout dans le code, donc dans Git. Les deux autres critères décisifs sont l'**effort de création d'un connecteur pour une source inconnue** (aucun SI cible n'est défini : le connecteur inédit est le cas nominal, pas l'exception) et la **gestion d'état** (curseurs incrémentaux, idempotence, reprise après échec), qui est ce qu'un agent non déterministe sait le moins bien improviser.

## Panorama des solutions

### dlt (dltHub / ScaleVector GmbH)

Bibliothèque Python pure, `pip install dlt`, sans backend ni serveur — version 1.30.0 du 11 août 2026, Apache 2.0, Python 3.10 à 3.14 (3.14 expérimental), ~5,8k étoiles GitHub. Trois sources « core » maintenues par l'éditeur couvrent l'essentiel du besoin : REST API (déclaratif : pagination, auth, ressources dépendantes), SQL database (réflexion SQLAlchemy, backends interchangeables `sqlalchemy` / `pyarrow` / `pandas` / `connectorx`) et filesystem (S3, GCS, Azure, **SFTP**, local ; CSV, Parquet, JSONL, avec exemples Excel et XML). L'état incrémental est persisté à la fois dans le répertoire de travail local et dans une table `_dlt_pipeline_state` **côté destination**, ce qui permet une reprise sur un système de fichiers vierge — propriété très utile en exécution éphémère (conteneur, Lambda, runner CI). Les contrats de schéma (`evolve` / `freeze` / `discard_row` / `discard_value`, applicables aux tables, colonnes et types) donnent un levier explicite sur le schema drift, en mode `evolve` par défaut. dltHub pousse depuis mars 2026 un « AI Workbench » (preview publique annoncée le 23 mars 2026) : serveur MCP `dlt-workspace-mcp` exposant l'inspection de données, toolkits de build/run/deploy, testé avec Claude Code, Cursor et Codex. À noter : le paquet `dlthub` (0.30.0, 11 août 2026) est une **extension commerciale sous licence payante** (YAML déclaratif, Iceberg, transformations) — la partie utile au cadre reste dans `dlt`, Apache 2.0.

### Airbyte

Plateforme complète (serveur, UI, base de métadonnées, workers Kubernetes/Docker), version 2.0 sortie le 15 octobre 2025 ; à partir de la 2.1 seul le Helm chart V2 est supporté, et `abctl` ≥ 0.30.2 est requis pour la montée de version. Licence **Elastic License 2.0 (ELv2)** pour la plateforme et les connecteurs des dépôts publics, MIT pour le protocole (le CDK est annoncé MIT par l'éditeur — non revérifié à la source primaire dans cette session). ELv2 n'est pas bloquante pour un usage interne, mais interdit d'offrir le produit en service managé ou d'exposer directement son UI/API à des tiers : point à valider si le cadre doit être revendu ou opéré pour des clients. Le Connector Builder est un no-code UI adossé à un manifeste **YAML low-code** exportable et donc versionnable ; il ne gère que des sources HTTP et aucune destination. Le CDK Python reste nécessaire pour tout le reste (SOAP, protocoles exotiques). `PyAirbyte` permet d'appeler des connecteurs Airbyte depuis Python sans la plateforme, mais le projet est nettement plus petit (~343 étoiles) et n'apporte ni orchestration, ni planification, ni supervision. Depuis mai 2026, Airbyte pousse « Airbyte Agents » (Context Store, SDK, endpoint MCP), une offre **commerciale** qui répond à un autre besoin que le nôtre : donner du contexte à des agents applicatifs, pas rendre l'ingestion pilotable par agent.

### Meltano et l'écosystème Singer

Meltano est un orchestrateur ELT déclaratif (`meltano.yml`) au-dessus de l'écosystème Singer, en Python, auto-hébergeable et sans serveur obligatoire. Versions actives : v4.2.2 du 22 juillet 2026, avec une branche 3.9.x maintenue en parallèle (v3.9.5, même date) — le projet est donc bien vivant. **Attention au contexte capitalistique** : la société Meltano avait pivoté vers Arch, Arch a fermé, et le projet open source a été repris par Matatika ; la date exacte de l'opération est incertaine (une source secondaire indique décembre 2025, le billet officiel « Looking for Arch » est daté de mars 2026). L'engagement affiché est le maintien en open source, mais l'assise industrielle est plus faible que celle de dlt ou d'Airbyte. Le Singer SDK (`singer-sdk`) est actif (v0.54.5 du 16 juin 2026) et constitue le vrai atout : écrire un tap est cadré et testable. Le point faible est la qualité très hétérogène du parc de taps communautaires (le Meltano Hub annonce 600+ connecteurs, chiffre non vérifié à la source primaire), et un protocole inter-processus JSON sur stdout qui coûte cher en CPU sur les gros volumes.

### Debezium (Change Data Capture)

Référence du CDC log-based, Apache 2.0, très actif : 3.6.1.Final le 4 août 2026, 3.7.0.Beta1 le 26 août 2026. Deux modes de déploiement pertinents ici : Kafka Connect (lourd) et surtout **Debezium Server**, application Quarkus autonome **sans Kafka obligatoire**, capable d'écrire vers Kinesis, Pub/Sub, Pulsar, RabbitMQ, NATS, Redis, HTTP, etc. C'est de la JVM : ce n'est pas une bibliothèque Python et un agent ne l'« appelle » pas, il la **configure** (fichier de propriétés versionnable en GitOps). Debezium ne couvre qu'une famille de sources — les bases transactionnelles — mais y répond mieux que tout le reste : capture des DELETE, latence faible, pas de scan de table, offsets gérés par le connecteur. Il est complémentaire, pas concurrent, d'un outil batch.

### Sling

CLI Go compilée (binaire unique), `sling-cli` sous **GPL-3.0**, ~892 étoiles, très haute fréquence de livraison : v1.5.26 le 17 août 2026, avec sur les seuls mois de mai à août 2026 l'ajout d'un connecteur ScyllaDB, d'un lecteur CDC MariaDB, du support ADBC et de corrections DuckDB. Couvre 34+ bases et 9+ systèmes de fichiers (S3, Azure, GCS), formats CSV/Parquet/JSON/Excel, réplication déclarative par YAML avec jokers (`schema.*`) et modes incrémentaux. Un wrapper Python existe (`pip install sling`), mais on pilote un binaire, pas une API objet. Excellent pour du base-à-base et du fichier-à-base à faible effort ; **aucun cadre pour construire un connecteur d'API métier inédite**, ce qui est précisément notre cas nominal. La GPL-3.0 est acceptable en invocation sous-processus, mais mérite une validation juridique si le binaire est redistribué avec le cadre.

### Estuary Flow

Plateforme temps réel (moteur Gazette, Go/Rust) unifiant batch et streaming autour de captures, collections et matérialisations, avec 100+ connecteurs dont la réutilisation de connecteurs Airbyte communautaires. Le dépôt est sous **BSL** avec date de bascule au 20 novembre 2029 vers Apache 2.0, et une clause interdisant l'usage comme « Data Processing Service » exposé à des tiers ; existent aussi des déploiements privés et BYOC. Le projet est modeste en visibilité (~963 étoiles). Techniquement crédible sur le CDC à faible latence, mais c'est un runtime distribué à opérer, source-available et non open source, avec un modèle d'objets propriétaire : deux frictions fortes pour un cadre générique et souverain non tranché.

### Fivetran (référence de comparaison)

SaaS propriétaire, aucune version auto-hébergeable au sens usuel. L'actualité 2026 est structurante pour le marché : **fusion avec dbt Labs finalisée le 1er juin 2026**, et reprise de la garde du projet open source Great Expectations annoncée le 13 mai 2026. On obtient un acteur intégré ingestion + transformation + qualité, positionné explicitement sur l'« agent-ready ». Pour notre cadre, l'outil est disqualifié par construction : pas d'auto-hébergement, tarification à la ligne modifiée, verrouillage maximal, et pilotage réduit à une API d'administration. Il reste un point de comparaison utile pour arbitrer un « faire vs acheter » côté SaaS.

### Briques Python bas niveau

Toutes maintenues et sous licences permissives : **SQLAlchemy** 2.0.52 (11 août 2026, MIT) pour la réflexion de schéma et l'abstraction dialecte ; **connectorx** 0.4.5 (18 janvier 2026, MIT) pour la lecture parallélisée base → Arrow/Polars/pandas ; **ADBC** (`adbc-driver-manager` 1.12.0, 28 juillet 2026, Apache 2.0) pour un accès colonne à colonne sans conversion ligne par ligne ; **DuckDB** 1.5.5 (22 juillet 2026, MIT) et **Polars** 1.44.1 (26 août 2026, MIT) pour lire directement CSV/Parquet/JSON, y compris sur objet distant ; **zeep** 4.3.3 (18 juin 2026, MIT) pour SOAP, explicitement « stable, peu d'évolutions » — ce qui est cohérent avec un protocole figé. Ces briques n'offrent ni état incrémental, ni idempotence, ni gestion du schema drift : elles sont le **socle** d'un connecteur, pas un connecteur. C'est précisément ce que dlt ajoute par-dessus, en les réutilisant comme backends.

## Grille comparative

| Solution | Licence | Auto-hébergeable | Lib ou plateforme | Incrémental / CDC | Effort nouveau connecteur | Risque de verrouillage | Adapté au cadre ? |
|---|---|---|---|---|---|---|---|
| **dlt** 1.30.0 | Apache 2.0 (`dlthub` séparé = commercial) | Oui, rien à héberger | **Bibliothèque** Python, appelable in-process | Curseurs + état persisté local **et** en destination (`_dlt_pipeline_state`) ; merge/append/replace ; pas de CDC log-based | Faible : REST déclaratif, SQL par réflexion, filesystem natif ; code Python pour le reste | Faible (Apache 2.0, code généré = Python lisible) | **Oui — socle recommandé** |
| **Airbyte** 2.0 | ELv2 (protocole MIT ; CDK annoncé MIT) | Oui (K8s/Docker, `abctl`, Helm V2) | **Plateforme** (serveur + UI + métadonnées) | Incrémental par curseur, CDC via connecteurs dédiés | Moyen : manifeste YAML low-code (HTTP only) ou CDK Python | Moyen-élevé : ELv2 + modèle d'objets propriétaire + composant stateful | Partiellement — comme parc de connecteurs SaaS, pas comme cœur |
| **PyAirbyte** | Suit Airbyte | Oui (lib) | Bibliothèque | Incrémental supporté, pas d'orchestration | Idem Airbyte | Moyen | Option d'appoint seulement |
| **Meltano + Singer SDK** | Apache 2.0 (projet repris par Matatika) | Oui, sans serveur obligatoire | Entre les deux : CLI + `meltano.yml` déclaratif | Bookmarks Singer (messages STATE) | Moyen : Singer SDK cadre bien l'écriture d'un tap | Faible techniquement, **moyen sur la gouvernance** (Arch fermé, reprise récente) | Repli crédible, pas premier choix |
| **Debezium** 3.6.1 | Apache 2.0 | Oui (Debezium Server, sans Kafka) | **Plateforme JVM** à configurer | **CDC log-based** : le meilleur du panel | Sans objet : connecteurs bases fournis | Faible | **Oui, en complément ciblé** (bases transactionnelles) |
| **Sling** 1.5.26 | GPL-3.0 (CLI) | Oui (binaire unique) | CLI + wrapper Python | Modes incrémentaux, CDC partiel (ex. MariaDB) | **Élevé pour une API inédite** : pas de SDK connecteur | Faible-moyen (GPL à valider si redistribution) | Non comme cœur ; utile en base-à-base tactique |
| **Estuary Flow** | **BSL** → Apache 2.0 le 20/11/2029 | Oui (private / BYOC) | **Plateforme** distribuée (Gazette) | Streaming + CDC faible latence | Moyen (connecteurs propres + Airbyte) | **Élevé** : source-available, modèle d'objets propriétaire | Non |
| **Fivetran (+ dbt Labs)** | Propriétaire | **Non** | SaaS | Incrémental + CDC managés | Sans objet (catalogue fermé) | **Maximal** | Non — comparaison uniquement |
| **ingestr** | FSL-1.1 (→ Apache 2.0 à 2 ans) | Oui | CLI + SDK Python | append / merge / delete+insert | Élevé pour une source inédite | Moyen (FSL = source-available temporaire) | Non — mais bon exemple d'ergonomie CLI |
| **Briques bas niveau** (SQLAlchemy, connectorx, ADBC, DuckDB, Polars, zeep) | MIT / Apache 2.0 | Oui | **Bibliothèques** | Aucun état fourni : à implémenter | Élevé (tout est à écrire) | Nul | **Oui, en socle** — notamment zeep pour SOAP |

## Ce que ça implique pour un runtime agentique

**L'agent doit configurer, pas improviser.** Le partage de responsabilité qui tient la route est : l'agent produit une **configuration déclarative** (un dict/YAML de source REST, une liste de tables, un glob de fichiers) et, seulement quand la source sort du cadre, un **module Python borné** ; le moteur d'ingestion, lui, reste déterministe et gère état, retries, typage et écriture. Laisser un LLM écrire à chaque exécution le code de pagination, de curseur et de reprise est le pire des mondes : non déterministe, non testable, non rejouable.

**Le coût est dans la génération, pas dans l'exécution.** Une fois la configuration produite et validée, elle est versionnée dans Git et rejouée sans LLM. Le modèle n'est rappelé qu'en cas de dérive (nouveau champ, changement de pagination, erreur inconnue). C'est exactement le découpage que dlt rend possible : la configuration REST est de la donnée, pas du code ; le pipeline est idempotent ; l'état vit dans la destination.

**Le déterminisme se gagne par des garde-fous, pas par le prompt.** Trois garde-fous minimum : (1) contrats de schéma en mode `freeze` ou `discard_row` en production, `evolve` seulement en découverte, pour que le schema drift lève une alerte au lieu de polluer silencieusement la cible ; (2) exécution en bac à sable d'abord (destination DuckDB locale) avec vérification de volumétrie et de clés avant promotion ; (3) revue humaine sur la promotion vers production — c'est le schéma « propose, verify, enforce » que dltHub formalise dans son AI Workbench, et il est reproductible sans souscrire à l'offre commerciale.

**Reprise sur erreur.** Le point critique est de ne jamais laisser l'état de curseur dans le contexte de l'agent. Avec dlt, l'état est écrit en destination et rechargé automatiquement : un conteneur qui meurt en cours de run redémarre à froid sans perte ni doublon (write disposition `merge` avec clé primaire). Toute solution où l'agent porterait lui-même le curseur est à rejeter.

**Une plateforme reste pilotable, mais mal.** Rien n'interdit à un agent de créer une source Airbyte par API REST. Mais il faut alors héberger un serveur, sauvegarder sa base de métadonnées, réconcilier son état avec Git, et l'agent perd le retour d'exécution fin (il sonde un statut de job). L'écart d'ergonomie agentique entre bibliothèque et plateforme est réel et défavorable à la plateforme.

## Recommandation

**Option 1 (recommandée) — dlt comme moteur unique d'Extract-Load, complété par Debezium là où le CDC est requis.**
dlt est la seule solution du panel qui soit à la fois une bibliothèque Python appelable in-process, sous Apache 2.0, sans composant à héberger, avec une gestion d'état survivant à un runtime éphémère et un contrôle explicite du schema drift. Elle couvre nativement trois des quatre familles de sources (SQL, REST, fichiers sur S3/SFTP), s'appuie sur les briques bas niveau déjà retenues (SQLAlchemy, connectorx, PyArrow) et se déploie en GitOps sans effort particulier — un dépôt, des pipelines Python, une CI. Le SOAP est le seul trou : il se comble avec `zeep` en amont d'un `@dlt.resource`, pattern simple et à écrire une fois. On adopte `dlt` (Apache 2.0) sans souscrire au paquet commercial `dlthub`, en réimplémentant les garde-fous de l'AI Workbench dans notre propre cadre. Debezium Server est ajouté **uniquement** pour les bases transactionnelles où la latence ou la capture des suppressions l'exigent : il se configure par fichier versionné, ce qui reste compatible GitOps.

**Option 2 (repli) — Meltano + Singer SDK.**
Pertinente si le client valorise un format de connecteur standardisé et un parc communautaire important, et si l'on accepte d'écrire les taps manquants avec le Singer SDK. Techniquement solide et bien maintenu (v4.2.2 en juillet 2026, SDK v0.54.5 en juin 2026). Deux réserves : le risque de gouvernance après la fermeture d'Arch et la reprise par Matatika, encore récente ; et l'ergonomie agentique inférieure — l'agent produit du YAML et des processus séparés plutôt que du code qu'il peut instrumenter.

**Option 3 (tactique, non structurante) — Airbyte OSS en satellite.**
À n'envisager que si le SI cible se révèle dominé par des SaaS grand public déjà couverts par le catalogue Airbyte, et à cantonner à ce rôle : Airbyte alimente une zone d'atterrissage, le cadre agentique reste piloté par dlt. On accepte alors d'héberger la plateforme et de vivre avec ELv2.

**Ce que j'écarte, et pourquoi :**
- **Fivetran (+ dbt Labs)** : pas d'auto-hébergement, verrouillage maximal, catalogue fermé — incompatible avec un cadre générique et une souveraineté non tranchée. La fusion finalisée le 1er juin 2026 renforce l'intégration verticale, donc le verrouillage.
- **Estuary Flow** : BSL jusqu'au 20 novembre 2029, runtime distribué à opérer, modèle d'objets propriétaire. Le gain (latence) ne correspond à aucun besoin exprimé.
- **Sling** comme brique centrale : aucun cadre de développement de connecteur, or la source inconnue est notre cas nominal. Reste acceptable en outil tactique base-à-base ; la GPL-3.0 demande une validation juridique en cas de redistribution.
- **ingestr** : bonne ergonomie CLI mais licence FSL-1.1 (source-available pendant deux ans) et même limite que Sling sur les sources inédites.
- **Airbyte comme cœur du runtime** : plateforme stateful à héberger, ELv2 restrictive si le cadre est un jour opéré pour des tiers, et pilotage par API qui dégrade la boucle de rétroaction de l'agent.
- **Une construction 100 % sur briques bas niveau** : re-développer état, idempotence, normalisation et évolution de schéma serait réinventer dlt, avec moins de tests.

**Points de décision ouverts :** validation juridique d'ELv2 et de la GPL-3.0 selon le modèle de distribution du cadre ; choix d'inclure ou non Debezium dès la V1 (dépend de l'existence d'un besoin CDC réel, inconnu tant qu'aucun SI n'est cible) ; degré d'autonomie laissé à l'agent sur la promotion en production ; traitement des documents non structurés, qui relève surtout d'un autre axe — dlt les transporte mais ne les comprend pas.

## Sources

- https://pypi.org/project/dlt/ — version 1.30.0 (11 août 2026), Apache 2.0, Python 3.10–3.14.
- https://github.com/dlt-hub/dlt — dépôt dlt : ~5,8k étoiles, fonctionnalités (REST/SQL/filesystem, contrats de schéma, incrémental).
- https://github.com/dlt-hub/dlt/releases — cadence de livraison récente (1.28 → 1.30).
- https://pypi.org/project/dlthub/ — paquet `dlthub` 0.30.0 (11 août 2026), extension **commerciale** sous licence, éditeur ScaleVector GmbH.
- https://dlthub.com/blog/ai-workbench — annonce de l'AI Workbench (preview publique, 23 mars 2026), toolkits et modèle « propose, verify, enforce ».
- https://dlthub.com/docs/general-usage/state — état de pipeline : répertoire local + table `_dlt_pipeline_state` en destination, restauration à froid.
- https://dlthub.com/docs/general-usage/incremental-loading — dispositions d'écriture append/replace/merge, curseurs.
- https://dlthub.com/docs/general-usage/schema-contracts — modes `evolve`/`freeze`/`discard_row`/`discard_value` sur tables, colonnes et types.
- https://dlthub.com/docs/dlt-ecosystem/verified-sources/filesystem/ — protocoles supportés dont **SFTP**, formats CSV/Parquet/JSONL, incrémental sur date de modification.
- https://dlthub.com/docs/dlt-ecosystem/verified-sources/sql_database/ — backends SQLAlchemy / PyArrow / pandas / ConnectorX, réflexion de schéma.
- https://dlthub.com/docs/dlt-ecosystem/verified-sources/ — sources core vs vérifiées ; **aucune source SOAP**.
- https://docs.airbyte.com/community/licenses — ELv2 pour plateforme et connecteurs, MIT pour le protocole.
- https://github.com/airbytehq/airbyte/releases — Airbyte 2.0 (15 octobre 2025) et historique 1.5 → 2.0.
- https://docs.airbyte.com/platform/connector-development/ — trois voies : Connector Builder no-code, CDK low-code YAML, CDK Python.
- https://docs.airbyte.com/platform/connector-development/connector-builder-ui/overview — manifeste YAML exportable, sources HTTP uniquement, assistant IA.
- https://github.com/airbytehq/PyAirbyte — usage des connecteurs Airbyte en Python, ~343 étoiles, absence d'orchestration et de supervision.
- https://github.com/meltano/meltano/releases — Meltano v4.2.2 (22 juillet 2026) et branche 3.9.x maintenue.
- https://github.com/meltano/sdk/releases — Singer SDK v0.54.5 (16 juin 2026), maintenance active.
- https://meltano.com/blog/looking-for-arch — fermeture d'Arch et reprise du projet Meltano par Matatika (date d'opération incertaine).
- https://hub.meltano.com/singer/spec/ — spécification Singer 0.3.0 : messages RECORD / SCHEMA / STATE, bookmarks.
- https://github.com/debezium/debezium/tags — 3.6.1.Final (4 août 2026), 3.7.0.Beta1 (26 août 2026).
- https://github.com/debezium/debezium-server — application Quarkus autonome, **sans Kafka obligatoire**, sinks disponibles, Apache 2.0.
- https://github.com/slingdata-io/sling-cli — Sling : GPL-3.0, 34+ bases, 9+ systèmes de fichiers, wrapper Python.
- https://github.com/slingdata-io/sling-cli/releases — v1.5.26 (17 août 2026), CDC MariaDB et support ADBC ajoutés à l'été 2026.
- https://github.com/estuary/flow — architecture Gazette, 100+ connecteurs, déploiements SaaS / privé / BYOC.
- https://github.com/estuary/flow/blob/master/LICENSE-BSL — BSL, bascule Apache 2.0 au 20 novembre 2029, clause « Data Processing Service ».
- https://www.fivetran.com/press — fusion Fivetran / dbt Labs **finalisée le 1er juin 2026** ; reprise de Great Expectations annoncée le 13 mai 2026.
- https://www.fivetran.com/blog — positionnement « agent-ready » du couple Fivetran + dbt.
- https://github.com/bruin-data/ingestr — ingestr : FSL-1.1 (Apache 2.0 à deux ans), ~3,9k étoiles, CLI + SDK Python.
- https://pypi.org/project/SQLAlchemy/ — 2.0.52 (11 août 2026), MIT.
- https://pypi.org/project/connectorx/ — 0.4.5 (18 janvier 2026), MIT, sorties Arrow/Polars/pandas.
- https://pypi.org/project/adbc-driver-manager/ — 1.12.0 (28 juillet 2026), Apache 2.0, interface DBAPI + Arrow.
- https://pypi.org/project/duckdb/ — 1.5.5 (22 juillet 2026), MIT.
- https://pypi.org/project/polars/ — 1.44.1 (26 août 2026), MIT.
- https://pypi.org/project/zeep/ — 4.3.3 (18 juin 2026), MIT, client SOAP déclaré stable et peu évolutif.

> Réserves de fiabilité : la licence MIT du CDK Airbyte et le chiffre de « 600+ connecteurs » du Meltano Hub proviennent de sources secondaires non revérifiées ici ; la date exacte de la reprise de Meltano par Matatika reste ambiguë (décembre 2025 selon une source secondaire, billet officiel daté de mars 2026). Le catalogue de « 9 700+ définitions de sources » de dltHub est une affirmation éditeur, non auditée.
