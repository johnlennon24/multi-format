# ADR 0007 — Structure de dépôt et distribution

- **Statut** : Proposé
- **Date** : 2026-08-27
- **Décideurs** : Repreneur du dépôt ; en attente d'arbitrage du porteur du projet
- **Axes de l'état de l'art concernés** : `07-gitops-ci-cd`, `00-synthese`

## Contexte

Le cadre sera **redéployé sur plusieurs SI**. La question n'est pas « quel CI choisir »
mais : comment un même noyau sert N instances client **sans que la N+1 impose de réécrire
le noyau**, et **sans qu'un correctif du noyau oblige à N migrations manuelles**.

Trois contraintes cadrent tout (axe 7) :

- **Socle d'hébergement non tranché** : aucune décision ne doit supposer Kubernetes, un
  cloud, ou un magasin de secrets particulier. Tout ce qui touche au socle passe par une
  indirection.
- **Runtime agentique** : les prompts et les jeux d'évaluation sont des artefacts de
  production — versionnés, revus, testés en non-régression (ADR 0001, 0004).
- **GitOps** au sens du porteur (everything-as-code + CI/CD, sans opérateur de
  réconciliation imposé) : préserver l'option Argo CD / Flux sans la payer aujourd'hui.

## Options envisagées

### Option A — Monorepo noyau (workspace uv) + dépôt d'instance par SI généré par Copier

Le noyau est un monorepo de paquets Python versionnés ; chaque SI a son dépôt d'instance,
**généré et maintenu à jour** par un template Copier. Apporte une frontière nette
noyau/instance et la repropagation des évolutions du noyau (`copier update`). Coûte la
tenue du template et de la frontière. Détail : axe 7.

### Option B — Monorepo unique noyau + instances

Tout dans un seul dépôt. Apporte la simplicité initiale. Coûte le mélange des droits
d'accès, des cycles de recette et des contraintes de confidentialité propres à chaque
client — déconseillé par l'axe 7 dès qu'il y a plusieurs SI.

Le point qui tranche entre Copier et Cookiecutter : **Copier sait rejouer le diff du
template sur un projet déjà généré** (`copier update`) et déclarer des migrations ;
Cookiecutter, non.

## Décision

**Le cadre retient l'option A.**

1. **Deux niveaux de dépôts.** Un **monorepo noyau** (`multi-format`) en workspace uv
   (lockfile unique, dépendances internes en `tool.uv.sources` `workspace = true`), et un
   **dépôt d'instance par SI** (`mf-instance-<client>`), généré par un template Copier
   hébergé dans le noyau et maintenu à jour par `copier update`.

2. **Trois canaux de réutilisation, pas un seul :** paquet Python versionné (le code),
   template Copier (le squelette), points d'entrée `importlib.metadata` (l'extension propre
   à un SI). **Règle de tri en revue** : *si la réponse à « est-ce vrai pour le prochain SI
   ? » est non, ça ne va pas dans le noyau.*

3. **Structure** (détail et arborescence : axe 7). Le noyau contient `mf-core` (domaine,
   ports, runtime), un paquet de connecteurs par famille de sources, `mf-prompts`,
   `mf-cli`, plus `prompts/`, `evals/`, `templates/instance/`, `ci/`, `deploy/base/`. Le
   dépôt d'instance ne contient **aucune ligne de logique** : configuration déclarative,
   surcharges de prompts, secrets chiffrés, manifestes de déploiement, et un `.gitlab-ci.yml`
   qui inclut les composants CI du noyau.

4. **Configuration.** YAML en couches (`base.yaml` + `env/<env>.yaml`, deltas seulement),
   validé par **Pydantic Settings** au démarrage. Ordre de priorité figé :
   `variables d'environnement > secrets déchiffrés > env/<env> > base > défauts du noyau`.
   `multiformat validate` charge et valide toute la configuration **sans effet de bord**,
   et c'est une étape **bloquante** de CI sur chaque environnement.

5. **Secrets.** **SOPS + clés `age`** aujourd'hui (zéro dépendance d'infrastructure),
   migrable vers le KMS du socle en changeant `.sops.yaml`, sans toucher au code ni aux
   fichiers. **Le code ne lit jamais un magasin de secrets** : il lit des variables
   d'environnement (ou un répertoire de secrets) ; ce qui les remplit est une affaire de
   déploiement.

6. **CI/CD.** **GitLab CI** par défaut, avec **composants versionnés** publiés par le
   noyau et inclus par tag épinglé dans chaque instance. Bloquants : `ruff`, `mypy
   --strict` (noyau), pytest avec doublure LLM déterministe, `multiformat validate`,
   `datacontract lint`/`changelog`, tests de contrat sur cassettes VCR, non-régression de
   prompts (promptfoo). Analyse de vulnérabilités en **avertissement** ; évaluation LLM
   complète en **nocturne**, jamais bloquante sur une fusion. **Images par digest**,
   signées (cosign / attestations), avec provenance SLSA.

7. **Prompts comme du code.** `prompts/<domaine>/<nom>.<n>.md` avec en-tête YAML
   (identifiant, version SemVer, modèle cible, paramètres, variables). Le `PromptStore`
   résout par empreinte et **refuse de démarrer** si le prompt manque ou diffère du verrou.
   SemVer : **majeure = changement de forme de sortie = rupture de contrat de données**,
   traitée comme un changement majeur de schéma. Un registre SaaS n'est accepté qu'en
   **miroir descendant** poussé depuis la CI, jamais en source.

8. **Tests d'un système appelant un LLM**, trois cercles : unitaire (doublure scriptée,
   chaque commit), contrat sur **cassettes VCR** (mode `none` par défaut, échec sur tout
   appel réseau imprévu), évaluation sur modèle réel (promptfoo / Inspect AI, nocturne).
   Aucune comparaison caractère par caractère : conformité au schéma, présence des champs,
   assertions sémantiques, et **seuil de réussite agrégé**.

9. **Trajectoire vers un GitOps strict** — sept décisions prises maintenant, à faible coût,
   qui rendent la bascule Argo CD / Flux mécanique : pas de script de déploiement impératif ;
   manifestes déclaratifs `deploy/base` + `deploy/overlays/<env>` dès le premier jour ;
   images par digest ; séparation build/deploy (la promotion est un commit qui ne change
   qu'un digest) ; patron des manifestes rendus ; configuration depuis un ConfigMap issu de
   Git ; secrets externalisés derrière des variables d'environnement chiffrées par SOPS.

## Exemple d'application — un SI avec PostgreSQL et Neo4j

Soit un SI cible comportant une base **PostgreSQL** (données relationnelles) et une base
**Neo4j** (graphe). Il donne lieu à un dépôt `mf-instance-<client>` généré par Copier,
contenant notamment :

- `config/sources/postgres-rh.yaml` — déclare la source PostgreSQL. Elle est consommée par
  le paquet de connecteurs SQL du **noyau** : PostgreSQL relève d'une famille de sources
  générique (SQL par réflexion, cf. ADR 0003), donc rien de spécifique au client.
- `config/sources/neo4j-graphe.yaml` — déclare la source Neo4j. Application de la règle de
  tri : si un connecteur graphe/Cypher est réutilisable d'un SI à l'autre, il devient un
  **paquet de connecteurs du noyau** (famille NoSQL/graphe) ; s'il porte une logique
  propre à ce client, il vit comme **plugin** dans son propre paquet, déclaré via
  `[project.entry-points."multiformat.connectors"]` et découvert par le noyau sans que
  celui-ci le connaisse. Dans les deux cas, **jamais** un `if client == "x"` dans le noyau.

Cet exemple illustre la frontière : le générique (PostgreSQL) monte dans le noyau, le
spécifique (un traitement Neo4j propre au client) reste en plugin, et la configuration des
deux sources vit dans le seul dépôt d'instance.

## Justification

L'option A est la seule qui réponde aux deux moitiés de la question du contexte à la fois :
le **paquet versionné** diffuse les correctifs du noyau par SemVer, et le **template Copier**
repropage les évolutions de squelette aux instances déjà déployées (`copier update`) — ce
que Cookiecutter ne sait pas faire, et qui est le critère décisif. Les **plugins** absorbent
le spécifique-SI sans polluer le noyau. La frontière « aucune logique dans l'instance »
garantit qu'un correctif de noyau ne se transforme jamais en N migrations manuelles.

Toutes les briques d'outillage (uv, Pydantic Settings, SOPS, GitLab Components, ruff, mypy,
VCR, promptfoo) sont permissives et auto-hébergeables, cohérentes avec l'ADR 0008 et avec
un socle non tranché. Le report de tout choix de socle derrière une indirection (secrets,
configuration, manifestes) préserve l'option GitOps stricte sans la payer aujourd'hui.

## Conséquences

### Ce que ça nous apporte

- Un noyau réutilisable dont les correctifs et le squelette se propagent aux instances.
- Une frontière nette noyau / instance / plugin, opposable en revue par une règle simple.
- Une CI qui traite prompts, contrats et cassettes comme des artefacts de production.
- Une trajectoire GitOps stricte préparée sans engagement de socle.

### Ce que ça nous coûte

- La tenue du template Copier et de ses migrations entre versions.
- La discipline des trois canaux (savoir, à chaque ajout, s'il va en paquet, template ou
  plugin).
- La contrainte du workspace uv : `requires-python` commun aux membres, mal adapté si des
  membres ont des exigences conflictuelles.

### Ce que ça nous ferme

- Le monorepo unique noyau+instances. Réouvrable si le client final est unique et interne.
- Jenkins comme CI par défaut (retenu seulement sous contrainte client) ; GitHub Actions
  reste un portage possible via *reusable workflows*.

## Critères de réexamen

Cette décision devra être rediscutée si :

- **Un membre du workspace uv exige un `requires-python` incompatible** avec les autres :
  imposerait de scinder le workspace.
- **Le SI cible impose GitHub ou Jenkins** : change la forme des composants CI (point ouvert
  à trancher avec le porteur).
- **Les dépôts d'instance doivent vivre chez le client** : change la stratégie de
  publication du noyau (index privé ou dépendance Git épinglée — point ouvert).
- **Un besoin d'opérateur de réconciliation** (Argo CD / Flux) devient réel : la trajectoire
  du point 9 doit alors être exécutée.
- **Une brique d'outillage change de licence** (critère bloquant, cf. ADR 0008).
