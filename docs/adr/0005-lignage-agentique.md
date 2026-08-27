# ADR 0005 — Lignage agentique

- **Statut** : Proposé
- **Date** : 2026-08-27
- **Décideurs** : Repreneur du dépôt ; en attente d'arbitrage du porteur du projet
- **Axes de l'état de l'art concernés** : `05-contrats-qualite-tracabilite`, `00-synthese`

## Contexte

Le cadre doit pouvoir reconstituer a posteriori **d'où vient une valeur** : par quelle
source, quel traitement, quel modèle et quelle version de prompt. Sur un runtime
agentique, cette traçabilité n'est pas un confort d'audit : c'est ce qui permet
d'expliquer et de rejouer une décision non déterministe, et ce qui rend l'échec
silencieux (valeur plausible mais fausse) détectable après coup.

L'état de l'art (axe 5) établit deux faits :

- **OpenLineage est le standard consolidé** (Apache 2.0, graduated LF AI & Data), avec un
  modèle à facettes (`schema`, `columnLineage`, `dataQualityMetrics`,
  `dataQualityAssertions`…) et un mécanisme officiel de **facettes custom** (préfixe
  `{prefix}_{name}`). Aucun standard concurrent crédible.
- **Aucun standard ne couvre le cas agentique.** OpenLineage ne dit rien de l'ancrage
  source→valeur, de la version de prompt comme entité, ni de la confiance par champ. Les
  conventions **OpenTelemetry GenAI** couvrent l'inférence (modèle, prompt, tokens,
  trace) mais sont au statut *development*, donc instables sur 12 à 18 mois, et leur
  corrélation avec OpenLineage n'est pas normalisée.

C'est à la fois une lacune et l'opportunité de différenciation identifiée par la synthèse :
ces trois facettes manquantes sont une part réelle de la valeur réutilisable du cadre.

## Options envisagées

### Option A — OpenLineage + facettes custom, backend maison, DataHub différé

Émettre du OpenLineage dès la V1, spécifier les trois facettes manquantes, et recevoir
les événements dans un **collecteur maison simple**. Le catalogue riche (DataHub) est
reporté en phase 2. Apporte une V1 sans brique lourde à opérer et une chaîne entièrement
maîtrisée. Coûte l'écriture du collecteur et des facettes. Détail : axe 5.

### Option B — OpenLineage + Marquez comme backend

Marquez est l'implémentation de référence (Apache 2.0). Coûte un risque de maintenance
réel : aucune release depuis octobre 2024 malgré des commits en 2026. Détail : axe 5.

### Option C — DataHub dès la V1

Catalogue complet (lignage colonne, assertions, lecture des contrats ODCS depuis Git).
Coûte une brique lourde à opérer, et DataHub OSS n'exécute pas les assertions (runner
externe requis) — surdimensionné pour une V1 à un connecteur. Détail : axe 5.

## Décision

**Le cadre retient l'option A.**

1. **Standard.** OpenLineage est le standard de lignage. Le cadre émet des événements
   OpenLineage (`START` / `COMPLETE` / `FAIL`) et utilise ses facettes standard :
   `dataSource`, `columnLineage` (avec types `DIRECT`/`INDIRECT` et sous-types),
   `dataQualityAssertions` et `dataQualityMetrics` alimentées par le résultat de
   `datacontract test` (cf. ADR 0004).

2. **Trois facettes custom à spécifier** — c'est le livrable propre du cadre, ce qui le
   distingue d'un assemblage d'outils :
   - **Ancrage source→valeur** : page, offset ou bbox du texte source, et extrait littéral
     (*evidence span*), pour les documents non structurés notamment.
   - **Version de prompt** comme entité versionnée de premier rang, référencée par son
     hash et son chemin Git.
   - **Score de confiance par champ**.
   Elles suivent la convention officielle `{prefix}_{name}` d'OpenLineage.

3. **Corrélation avec l'inférence.** Les métadonnées d'inférence (identifiant et version
   du modèle, hash de prompt, température, tokens, latence) sont captées selon les
   conventions **OpenTelemetry GenAI**, corrélées à l'événement OpenLineage par un
   identifiant de trace propagé manuellement. Ces attributs OTel étant au statut
   *development*, ils sont **isolés derrière une couche interne** au cadre, pour absorber
   les renommages à venir sans propager le changement.

4. **Provenance par valeur : table dédiée côté cible.** La provenance par valeur extraite
   est persistée dans une **table dédiée, à côté de la donnée métier, dans la destination**.
   Elle est ainsi auditable avec la donnée, requêtable en SQL, et indépendante de la survie
   du backend de lignage.

5. **Backend de lignage.** Les événements OpenLineage sont reçus par un **collecteur
   maison simple** en V1. Le catalogue DataHub est différé en phase 2 (plusieurs
   connecteurs et consommateurs) ; il lit alors les contrats ODCS depuis Git et consomme
   OpenLineage, ce qui préserve le modèle GitOps.

6. **Écarté.** Marquez comme brique de production (maintenance lente). `data-diff`
   (archivé) déjà écarté par l'ADR 0004.

## Justification

OpenLineage s'impose faute de concurrent crédible et parce qu'il est le seul standard
extensible par facettes custom — mécanisme officiel qui rend spécifiable le cas agentique
sans sortir du standard. Spécifier nous-mêmes les trois facettes n'est pas un contournement :
c'est la valeur réutilisable que la synthèse identifie comme différenciante.

Le backend maison plutôt que Marquez ou DataHub découle de la même logique que les ADR
précédents : une V1 modeste, sans brique lourde à opérer, et le refus de s'adosser à un
projet en maintenance lente (Marquez) ou surdimensionné (DataHub) pour un besoin d'un seul
connecteur. Émettre un standard garde toutes les portes ouvertes : le collecteur maison
peut être remplacé par DataHub sans changer ce qui est émis.

La provenance en table dédiée côté cible plutôt que dans le seul backend répond à
l'exigence d'auditabilité : l'audit ne doit pas dépendre de la disponibilité et de la
rétention d'un service de lignage ; le lien valeur↔provenance doit rester direct et
requêtable là où vit la donnée.

## Conséquences

### Ce que ça nous apporte

- Une traçabilité fondée sur un standard graduated, extensible sans le quitter.
- Les trois facettes agentiques, différenciantes et réutilisables d'un SI à l'autre.
- Une provenance auditable en SQL, survivant à la perte du backend de lignage.
- Une V1 sans composant lourd, avec une porte de sortie propre vers DataHub.

### Ce que ça nous coûte

- L'écriture du collecteur maison et, surtout, la **spécification et l'implémentation des
  trois facettes custom** — c'est le poste d'effort principal de cet ADR.
- Le stockage supplémentaire de la provenance par valeur près de la donnée métier.
- La corrélation OpenLineage ↔ OpenTelemetry GenAI, non normalisée, portée manuellement, et
  exposée aux renommages des conventions GenAI encore instables.

### Ce que ça nous ferme

- Marquez, tant qu'aucune reprise active du projet n'est constatée.
- Rien du côté catalogue : DataHub reste ouvert en phase 2 sans coût de bascule sur ce qui
  est émis.

## Critères de réexamen

Cette décision devra être rediscutée si :

- **Les conventions OpenTelemetry GenAI se stabilisent** (passage hors *development*,
  version taguée) : simplifierait la couche d'isolation du point 3, voire normaliserait la
  corrélation avec OpenLineage.
- **OpenLineage publie des conventions GenAI/agent** : nos facettes custom devraient alors
  être alignées ou migrées vers le standard.
- **Le besoin d'un catalogue** apparaît (plusieurs connecteurs, plusieurs consommateurs) :
  déclenche la phase 2 (DataHub).
- **Marquez reprend un rythme de release** : pourrait redevenir un backend candidat.
- **Le volume de provenance par valeur** devient un coût de stockage significatif : à
  réexaminer (rétention, agrégation, ou bascule vers le backend seul).
