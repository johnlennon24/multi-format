# ADR 0002 — Socle agentique

- **Statut** : Proposé
- **Date** : 2026-08-27
- **Décideurs** : Repreneur du dépôt ; en attente d'arbitrage du porteur du projet
- **Axes de l'état de l'art concernés** : `01-frameworks-agentiques`, `00-synthese`

## Contexte

Le cadre a besoin d'un **socle qui exécute la boucle agentique** : appel de modèle,
appel d'outil, routage entre étapes, état. L'ADR 0001 a fixé la frontière — le LLM
produit des artefacts, il ne traite pas chaque enregistrement — ce qui cadre ce que le
socle doit savoir faire : piloter des agents qui découvrent, proposent et diagnostiquent,
pas traiter du volume ligne à ligne.

Deux contraintes issues de l'axe 1 pèsent sur le choix :

- **Un framework agentique n'est pas un moteur d'exécution durable.** Le checkpointing
  d'état intégré (LangGraph, ADK, CrewAI) sauvegarde en base, mais le run vit dans un
  processus : si le processus meurt, rien ne rejoue. Seul un vrai moteur durable
  (Temporal, DBOS) rejoue l'historique. Les deux notions ne doivent pas être confondues.
- **La souveraineté se joue au niveau du framework.** Un socle couplé à un fournisseur
  de modèle unique ferme la porte à un modèle auto-hébergé (Ollama, vLLM) avant même que
  la posture d'hébergement (axe 8) ne soit tranchée.

Le GitOps (ADR 0001, méthode du dépôt) ajoute une exigence : la boucle agentique doit
être **testable en CI de façon déterministe**, sans appel LLM réel.

## Options envisagées

### Option A — Pydantic AI

Framework agent typé (MIT), agnostique du fournisseur de modèle (Ollama et endpoints
compatibles OpenAI inclus), OpenTelemetry natif, et **mode `'test'` qui exécute les
agents hors-ligne sans appel LLM**. L'exécution durable est **déléguée à de vrais
moteurs** (intégrations première partie Temporal, DBOS, Prefect, Restate). Coûte de
devoir opérer un moteur durable à côté, et un moteur de graphe propriétaire moins outillé
que celui de LangGraph. Détail et versions : axe 1.

### Option B — LangGraph

Bibliothèque de graphes d'état (MIT), contrôle de flux explicite, écosystème abondant,
time travel et replay. Coûte deux points : le self-hosted de la plateforme managée
(LangSmith / LangGraph Platform) est réservé à une offre Enterprise au tarif non public,
et son checkpointing est fréquemment pris à tort pour de la durabilité. Détail : axe 1.

Autres candidats (MS Agent Framework, Google ADK, OpenAI Agents SDK, CrewAI, Claude Agent
SDK) : écartés par l'axe 1 pour verrouillage modèle, statut de maturité, orientation
« agents autonomes » ou gravité vers un cloud particulier. Non repris ici.

## Décision

**Le cadre retient Pydantic AI comme socle agentique (option A).**

1. **Socle.** Pydantic AI est le framework de la boucle agentique. Les agents, leurs
   outils et leurs sorties typées sont définis avec lui.

2. **Séparation boucle / durabilité, gravée en principe.** La boucle agentique et le
   moteur d'exécution durable sont deux couches distinctes. Le socle ne fournit pas la
   garantie de reprise après crash ; celle-ci vient d'un moteur dédié. Le code doit être
   structuré pour que le choix du moteur reste un branchement, pas une réécriture.

3. **Choix du moteur durable renvoyé à l'ADR 0009.** Le socle retenu délègue la
   durabilité à Temporal, DBOS, Prefect ou Restate via des intégrations de première
   partie. Lequel de ces moteurs est adopté dépend du SI cible (présence d'un Postgres,
   échelle attendue) et n'est pas tranché ici. Cet ADR n'exige que la compatibilité, déjà
   acquise.

4. **Testabilité en CI.** Le mode `'test'` hors-ligne de Pydantic AI est le mécanisme
   retenu pour rendre les tests d'agents déterministes, sans appel LLM réel. Toute
   nouvelle capacité agentique doit être couverte par des tests utilisant ce mode.

5. **Renvois.** L'agnosticisme modèle qu'offre Pydantic AI est exploité, mais
   l'abstraction du fournisseur de LLM (port interne, `models.yaml`) est l'objet de
   l'ADR 0006 et n'est pas définie ici. Le contrôle de flux reste explicite en Python ;
   l'usage de Pydantic Graph est possible mais n'est pas imposé.

## Justification

Pydantic AI est retenu parce qu'il satisfait les trois contraintes du contexte sans
compromis, là où les alternatives en sacrifient au moins une : il n'enferme pas dans un
fournisseur de modèle (contrairement au Claude Agent SDK), il ne confond pas checkpointing
et durabilité mais délègue à de vrais moteurs (contrairement à LangGraph, CrewAI, ADK), et
son mode `'test'` répond directement à l'exigence de CI déterministe du GitOps — critère
que l'axe 1 qualifie de discriminant et sous-estimé.

Face à LangGraph, l'écart décisif est le tarif non public de l'auto-hébergement de sa
plateforme, incompatible avec un cadre destiné à être redéployé sur plusieurs SI sans
coût de licence surprise, et le risque de confusion checkpoint/durabilité que la
séparation en couches de cet ADR vise précisément à éviter.

## Conséquences

### Ce que ça nous apporte

- Un socle typé, agnostique du fournisseur, testable hors-ligne en CI.
- Une durabilité assurée par un moteur éprouvé, branchable sans réécriture.
- La souveraineté (modèle auto-hébergé possible) laissée entièrement ouverte.
- Un verrouillage fournisseur très faible (MIT, pas de plateforme managée imposée).

### Ce que ça nous coûte

- Devoir opérer un moteur d'exécution durable à côté du socle (Temporal à opérer, ou DBOS
  plus léger) — coût réel, instruit et tranché à l'ADR 0009, pas ici.
- Un moteur de graphe propriétaire (Pydantic Graph) moins outillé que LangGraph ; si un
  besoin de graphe très riche apparaissait, il faudrait le réévaluer.
- Une dépendance à la cadence de publication soutenue de Pydantic AI (API v2) : à suivre
  en veille de version.

### Ce que ça nous ferme

- LangGraph comme socle. Réouvrable si un besoin de modèle de graphe explicite outillé
  (time travel, replay natif) devient structurant, par un nouvel ADR.
- Les socles à verrouillage modèle (Claude Agent SDK) tant que la souveraineté n'est pas
  tranchée pour du SaaS d'un fournisseur unique.

## Critères de réexamen

Cette décision devra être rediscutée si :

- **Pydantic AI change de licence** ou introduit une dépendance à une plateforme managée
  payante pour ses fonctions cœur (critère licence bloquant, cf. synthèse).
- **Le mode `'test'` disparaît** ou cesse de permettre une CI déterministe : c'est l'un
  des motifs principaux du choix.
- **Le SI cible impose un socle particulier** (par exemple un SI Microsoft/Azure où le MS
  Agent Framework deviendrait le choix par défaut).
- **Un besoin de graphe riche** non couvert par Pydantic Graph apparaît et justifie de
  réévaluer LangGraph.
