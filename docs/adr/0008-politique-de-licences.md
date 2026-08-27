# ADR 0008 — Politique de licences

- **Statut** : Proposé
- **Date** : 2026-08-27
- **Décideurs** : Repreneur du dépôt ; en attente d'arbitrage du porteur du projet
- **Axes de l'état de l'art concernés** : `00-synthese` (constats transverses), tous les axes

## Contexte

Le fait le plus frappant de l'état de l'art (synthèse) : **la licence a éliminé plus de
candidats que la technique**. Sur les huit axes, les disqualifications tiennent bien plus à
la gouvernance qu'aux capacités. Cas relevés : Soda Core bascule d'Apache 2.0 vers Elastic
2.0, Estuary est sous BSL jusqu'en 2029, Restate sous BSL 1.1, Windmill sous AGPLv3, dbt
Fusion sous ELv2, Great Expectations passe sous la garde de Fivetran, Fivetran fusionne avec
dbt Labs, Prefect rachète Dagster Labs, AutoGen et Semantic Kernel entrent en maintenance ;
plusieurs paquets (`soda-core`, `openmetadata-ingestion`, certains modèles Mistral,
`sling-cli`) portent des licences restrictives ou ambiguës.

Pour un cadre **destiné à durer sur plusieurs SI** et potentiellement revendu ou opéré pour
des tiers, une bascule de licence d'une dépendance structurante peut rendre le cadre
**indistribuable**. Le risque n'est pas technique : il est juridique et il est silencieux.

## Options envisagées

### Option A — Critère licence bloquant + porte de sortie par dépendance

La licence est un critère de revue bloquant, au même titre qu'un test qui échoue, et chaque
dépendance structurante a une porte de sortie documentée. Apporte une protection réelle
contre une bascule de licence. Coûte une discipline de revue et la tenue d'un inventaire.

### Option B — Critère consultatif, arbitrage au cas par cas

La licence est signalée mais n'arrête rien automatiquement. Apporte de la souplesse. Coûte
l'exposition au scénario exact que l'état de l'art documente à répétition : une dépendance
adoptée pour ses capacités devient interdite d'usage après une bascule, sans qu'aucun
garde-fou ne l'ait signalé à temps. Écartée.

## Décision

**Le cadre retient l'option A.**

1. **Critère bloquant en revue.** La licence de toute dépendance structurante est vérifiée
   en revue et **bloque** au même titre qu'un test en échec.

2. **Classes de licences** :
   - **Permissives** (MIT, Apache 2.0, BSD, ISC) — **acceptées** sans arbitrage. C'est le
     cas de toutes les briques retenues aux ADR 0002 à 0007.
   - **Copyleft fort** (GPL, AGPL) — **au cas par cas**. GPL acceptable uniquement en
     invocation par sous-processus (CLI), et sous **validation juridique** si le cadre est
     redistribué avec le binaire (ex. `sling-cli`). AGPL écartée par défaut.
   - **Source-available / restrictives** (BSL, Elastic 2.0, SSPL, FSL, Confluent Community,
     Collate Community, licence maison marquée `other`) — **bloquantes par défaut**.
     L'adoption exige un **arbitrage juridique explicite consigné dans un ADR**, et
     seulement pour un modèle de distribution compatible avec la clause.

3. **Porte de sortie obligatoire.** Chaque ADR adoptant une dépendance structurante doit
   documenter sa **porte de sortie** : par quoi la remplacer, et à quel coût, si sa licence
   bascule. Cette exigence devient un contenu obligatoire du gabarit d'ADR, pas une option.

4. **Dépendance au modèle de distribution.** Certaines licences (ELv2, BSL) interdisent
   d'offrir le produit en service managé ou de l'exposer à des tiers. Le caractère
   disqualifiant dépend donc de la **finalité de distribution** — usage interne, revente,
   ou opération pour des tiers. Cette finalité est un **attribut de chaque déploiement/SI**,
   à déclarer, pas une propriété du dépôt.

5. **Inventaire et contrôle automatisé.** Un inventaire des licences des dépendances est
   tenu et versionné ; la CI vérifie la licence des dépendances et **alerte** (ou bloque)
   sur toute licence non permissive dépourvue d'arbitrage consigné.

6. **Points juridiques ouverts, hérités des axes**, à faire trancher avant tout engagement :
   la licence exacte d'`openmetadata-ingestion` (divergence dépôt / PyPI), la licence
   commerciale distincte exigée par Mistral pour certains modèles ouverts, et la GPL-3.0 de
   `sling-cli` si ce dernier est redistribué.

## Justification

L'option A est retenue parce que l'état de l'art démontre empiriquement que le risque de
licence est le premier facteur d'élimination et qu'il se matérialise **par bascule**,
c'est-à-dire après l'adoption, quand la dépendance est déjà installée au cœur du cadre. Un
critère consultatif ne protège pas de ce scénario : seul un critère bloquant, doublé d'une
porte de sortie préparée à l'avance, permet de réagir sans réécriture d'urgence.

Rattacher le caractère disqualifiant à la **finalité de distribution** plutôt qu'à la
licence seule évite deux erreurs symétriques : interdire une brique parfaitement utilisable
en interne, ou autoriser en revente une brique dont la clause l'interdit. C'est cohérent
avec la logique « attribut de déploiement » déjà retenue pour la qualification AI Act
(axe 8) et la posture d'hébergement (ADR 0006 / 0010).

## Conséquences

### Ce que ça nous apporte

- Une protection explicite contre l'indisponibilité juridique d'une dépendance.
- Une porte de sortie prête pour chaque brique structurante, avant d'en avoir besoin.
- Une revue et une CI qui traitent la licence comme un critère de qualité de premier rang.
- Une décision de distribution (interne / revente / tiers) rendue explicite par déploiement.

### Ce que ça nous coûte

- La discipline de revue et la tenue de l'inventaire de licences.
- Le travail de veille : une licence acceptée aujourd'hui peut basculer demain.
- Le recours à un avis juridique pour les cas gris (copyleft redistribué, licences maison).

### Ce que ça nous ferme

- L'adoption opportuniste d'une brique source-available sans arbitrage — au prix d'un
  démarrage parfois plus lent, assumé.
- Rien d'irréversible : une brique bloquée peut être adoptée plus tard si un arbitrage
  juridique la valide pour le modèle de distribution retenu.

## Critères de réexamen

Cette décision devra être rediscutée si :

- **Une dépendance structurante bascule de licence** : déclenche l'activation de sa porte
  de sortie et un nouvel ADR.
- **Le modèle de distribution du cadre change** (d'un usage interne vers la revente ou
  l'opération pour des tiers) : réévalue toutes les briques sous licence à clause de service.
- **Un besoin réel impose une brique sous licence restrictive** : déclenche l'arbitrage
  juridique et sa consignation en ADR.
- **Un outillage de contrôle de licence** (scanner de dépendances) devient disponible et
  fiable : à intégrer en CI pour automatiser le point 5.
