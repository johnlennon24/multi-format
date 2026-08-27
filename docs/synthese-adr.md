# Synthèse des décisions d'architecture (ADR) — à valider par le porteur

> Document de synthèse à destination du porteur du projet. Daté du 27 août 2026.
> Il résume les huit ADR indépendants du SI, rédigés à partir de l'état de l'art, et
> signale les points sur lesquels un arbitrage est explicitement attendu.
> Les ADR complets sont dans [`docs/adr/`](adr/) ; leur matériau dans
> [`docs/etat-de-l-art/`](etat-de-l-art/).

## Où en est-on

Les **huit ADR indépendants du SI (0001 à 0008) sont rédigés**, tous en statut `Proposé`
— c'est-à-dire en attente d'arbitrage. Conformément à la méthode du dépôt, un ADR n'engage
le projet qu'une fois passé en `Accepté`, et cette bascule revient au porteur.

**Rien de structurant ne démarre avant cette validation** : l'ADR 0001 commande tous les
autres. Les trois ADR dépendants du SI (0009 à 0011) ne sont pas encore rédigés et ne
bloquent pas la V1 du noyau.

## Les huit décisions en une ligne

| ADR | Décision retenue | En une phrase | Source |
|---|---|---|---|
| **0001** | Frontière décision/exécution | Le LLM produit des artefacts versionnés ; le code déterministe traite le volume. Régime strict, seule exception : les documents non structurés. | axe 6 |
| **0002** | Socle agentique | **Pydantic AI** (MIT), agnostique du fournisseur, testable hors-ligne en CI ; le moteur durable est renvoyé à l'ADR 0009. | axe 1 |
| **0003** | Extract-Load | **dlt** (Apache 2.0) comme moteur unique, en bibliothèque ; Debezium différé au besoin CDC avéré. | axe 2 |
| **0004** | Contrats & validation | **ODCS + datacontract-cli + Pydantic/Instructor** ; validation en trois points, quarantaine rejouable, seuil paramétrable. | axe 5 |
| **0005** | Lignage agentique | **OpenLineage** + trois facettes propres (ancrage source→valeur, version de prompt, confiance par champ) ; collecteur maison, provenance en table dédiée. | axe 5 |
| **0006** | Abstraction du LLM | Port interne + adaptateur OpenAI-compatible unique + LiteLLM + `models.yaml` épinglé ; la posture d'hébergement est renvoyée à l'ADR 0010. | axe 8 |
| **0007** | Structure de dépôt | Monorepo noyau (workspace uv) + dépôt d'instance par SI généré par **Copier** + plugins ; CI GitLab, secrets SOPS+age, prompts versionnés. | axe 7 |
| **0008** | Politique de licences | Critère licence **bloquant en revue** + porte de sortie par dépendance ; caractère disqualifiant lié à la finalité de distribution. | synthèse |

## Points nécessitant explicitement votre arbitrage

Ces points sont signalés dans les ADR concernés ; ils ne peuvent pas être tranchés sans le
porteur.

- **ADR 0001** — La tension assumée : le code généré par le LLM est autorisé *dans son
  principe*, mais son exécution reste **verrouillée** tant que l'ADR bac à sable (sécurité,
  non instruit) n'est pas accepté, et la revue humaine des artefacts est renvoyée à un ADR
  ultérieur. À valider en connaissance de cause.
- **ADR 0004** — Le **seuil de mise en quarantaine** (strict, ou dégradé avec score de
  confiance) est un arbitrage métier, laissé paramétrable par contrat.
- **ADR 0007** — GitLab ou GitHub ; dépôts d'instance chez nous ou chez le client ; niveau
  SLSA visé ; budget d'évaluation LLM ; adoption ou non d'un registre de prompts en miroir.
- **ADR 0008** — La **finalité de distribution** du cadre (usage interne, revente, ou
  opération pour des tiers), qui détermine quelles licences deviennent disqualifiantes.

## Questions de cadrage encore ouvertes (n'empêchent pas la validation des ADR)

- La donnée va où : le cadre s'arrête-t-il au chargement, ou couvre-t-il la restitution ?
- Régime d'exécution (lots planifiés, événementiel, temps réel) et volumétrie anticipée.
- Données à caractère personnel ? Secteur régulé ?
- Contributeur unique ou équipe ? (dimensionne l'outillage qualité)

## Comment valider

Pour chaque ADR, dans l'ordre (0001 d'abord) : passer le statut de `Proposé` à `Accepté`
(ou `Rejeté`, en conservant la raison), et renseigner le champ « Décideurs ». Les ADR
dépendants du SI (0009 à 0011) seront rédigés ensuite, dès qu'un SI cible est connu.
