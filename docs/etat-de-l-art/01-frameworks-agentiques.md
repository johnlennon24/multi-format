# Axe 1 — Frameworks agentiques

> État de l'art arrêté au **27 août 2026**. Les versions et licences citées ont été
> relevées sur PyPI et sur la documentation officielle à cette date. Lorsqu'un élément
> n'a pas pu être confirmé sur une source consultable, il est noté « non vérifié ».

## Périmètre et enjeu pour le projet

Cet axe traite du **socle qui exécute la boucle agentique** : appel de modèle, appel
d'outil, routage entre étapes, état, reprise. Il ne traite pas de l'orchestration de
jobs (axe 3) ni des connecteurs de source (axe 2), même si la frontière est floue —
c'est précisément le point à trancher ici.

Le choix client d'un **runtime agentique** (les agents exécutent réellement
l'extraction en production) déplace le centre de gravité des critères. Un traitement
de 200 000 enregistrements sur 3 heures n'est pas une conversation : il faut
l'exécution durable, un graphe déterministe, et des tests reproductibles. La plupart
des frameworks de cet axe ont été conçus pour l'assistant conversationnel, pas pour
le batch de données. Deux conséquences :

1. **Un framework agentique n'est pas un moteur d'exécution durable.** Le
   checkpointing intégré (LangGraph, ADK, CrewAI) sauvegarde l'état dans une base,
   mais le run vit dans un processus : si le processus meurt, il faut un
   redéclencheur externe. Un vrai moteur durable (Temporal, DBOS) rejoue l'historique.
   C'est la critique documentée par Diagrid et ZenML, et elle est fondée.
2. **La souveraineté se joue ici.** Un framework couplé à un fournisseur de modèle
   unique ferme la porte à un modèle auto-hébergé (vLLM/Ollama) avant même l'axe 8.

## Panorama des solutions

### LangGraph (LangChain)

Bibliothèque Python/JS bas niveau de graphes d'état : nœuds, arêtes conditionnelles,
état typé partagé. Licence MIT, version **1.2.11** sur PyPI (constatée le 27/08/2026),
~40,5 k étoiles GitHub. Exécution : checkpoint à chaque *superstep*, checkpointers
`InMemorySaver`, `SqliteSaver`, `PostgresSaver` ; `interrupt()` pour le human-in-the-loop,
time travel et replay. Le contrôle du flux est explicite par construction — c'est son
point fort majeur pour notre cas. Limites : le run reste lié au processus, la croissance
non bornée des checkpoints est documentée, et le déploiement managé (LangSmith /
LangGraph Platform) réserve le self-hosted à l'offre **Enterprise** sur devis. La
bibliothèque elle-même reste auto-hébergeable sans licence.

### Pydantic AI

Framework agent typé de l'équipe Pydantic. Licence MIT, version **2.35.1** publiée le
**27 août 2026** — cadence très soutenue, API v2 stabilisée. Support « quasi tous les
fournisseurs » y compris **Ollama** et endpoints compatibles OpenAI. Point différenciant
pour nous : l'**exécution durable est déléguée à de vrais moteurs** — intégrations
première partie Temporal, DBOS et Prefect, plus Restate, Kitaru et Airflow. Outils
différés avec approbation humaine intégrée, OpenTelemetry natif, et un modèle `'test'`
qui exécute les agents hors-ligne sans appel LLM (testabilité déterministe réelle).
Limite : pas de moteur de graphe durable propriétaire — il faut opérer Temporal ou DBOS
à côté. Pydantic Graph existe mais reste moins outillé que LangGraph.

### Microsoft Agent Framework

Successeur officiel et fusion d'**AutoGen** et **Semantic Kernel**, tous deux passés en
mode maintenance (le README d'AutoGen le confirme : « will not receive new features »).
Licence MIT, version **1.15.0** (21/08/2026), .NET / Python / Go (Go en préversion).
Deux API de workflow en Python : graphe typé (`WorkflowBuilder`, checkpoints aux
frontières de superstep, `RequestInfoExecutor` pour le HITL) et une API fonctionnelle
**expérimentale**. Providers : Foundry, Azure OpenAI, OpenAI, Anthropic, **Ollama**.
Limites : gravité Azure assumée (les avertissements sur les « Third-Party Systems » sont
explicites dans la doc), écosystème Python plus jeune que .NET, et GA très récente
(1.0 en avril 2026) — peu de retours de production indépendants.

### Google ADK (Agent Development Kit)

Framework open source Apache-2.0, version **2.8.0** (26/08/2026), Python/TS/Go/Java/Kotlin.
ADK 2.0 a introduit des workflows par graphe en plus des agents séquentiel/parallèle/boucle,
avec sessions, state, memory services, compression de contexte, `resume` et `cancel`.
Support modèles large : Gemini, Claude, OpenAI, **LiteLLM, vLLM, Ollama**. Points faibles :
**rupture d'API entre 1.x et 2.0** (sessions 2.0 illisibles par les 1.x anciens) et cadence
bi-hebdomadaire — l'API bouge vite. La durabilité est de la reprise de session, pas du
rejeu transactionnel. Le chemin de production le plus documenté reste Vertex AI Agent
Engine, ce qui crée un biais GCP même si le conteneur générique est supporté.

### OpenAI Agents SDK

SDK léger MIT, version **0.22.0** (19/08/2026) — toujours en **0.x** après 18 mois, ce
qui est un signal de stabilité d'API à prendre au sérieux. Primitives : agents, handoffs,
guardrails, sessions (SQLite, SQLAlchemy, Redis, MongoDB), tracing intégré. Supporte plus
de 100 modèles via LiteLLM / any-llm, donc un modèle auto-hébergé est branchable. Exécution
durable via l'intégration **Temporal** (`temporalio.contrib.openai_agents`), annoncée GA le
23 mars 2026 selon Temporal — le README PyPI de `temporalio` mentionne encore « public
preview », divergence non résolue. Le modèle par handoffs pousse vers l'autonomie plutôt
que vers le graphe déterministe ; le tracing par défaut remonte chez OpenAI (désactivable).

### CrewAI

Framework MIT, version **1.15.17** (20/08/2026), ~50 k étoiles. Deux modèles : *Crews*
(agents à rôles, autonomes) et *Flows* (déterministes : `@start`, `@listen`, `@router`,
`or_()`, `and_()`, état Pydantic, décorateur `@persist` avec SQLite par défaut, reprise
par `id` ou fork par `restore_from_state_id`). Le socle OSS est réellement libre ; la
plateforme **AMP** est commerciale (offre gratuite plafonnée à 50 exécutions/mois,
Enterprise sur devis, surcoût à l'exécution — tarifs relevés sur des sources secondaires,
**non confirmés sur le site officiel**). Limites pour nous : la culture du produit reste
« équipe d'agents autonomes », et le `@persist` sur SQLite est du checkpointing d'état,
pas de l'exécution durable.

### LlamaIndex Workflows / LlamaAgents

`llama-index-workflows` MIT, version **2.23.3** (22/08/2026). Modèle événementiel
async-first : une étape reçoit un événement, en émet un autre ; branches et boucles en
Python natif plutôt qu'en DAG. État partagé `ctx.store`, sérialisation
`Context.to_dict()`/`from_dict()`, `WorkflowCheckpointer`, support DBOS pour la durabilité,
HITL, OpenTelemetry. Déploiement via `llama-agents-server` et `llamactl` ; `llama_deploy`
est **déprécié** au profit de `llama-agents`. C'est le framework le plus proche d'un moteur
d'ingestion (héritage RAG/parsing), mais aussi celui dont le nommage et le packaging ont le
plus bougé en 18 mois — coût de maintenance à anticiper.

### Temporal (+ appels LLM)

Pas un framework agentique : un **moteur d'exécution durable** MIT, self-hostable, serveur
en v1.31.x (juillet 2026, source secondaire), SDK Python `temporalio` **1.32.0**. Les appels
LLM et les outils deviennent des *Activities* retryables ; le workflow rejoue son historique
après crash. C'est la seule brique de cet axe qui répond réellement à « l'agent plante à la
2ᵉ heure sur 200 000 enregistrements ». Intégrations agentiques : OpenAI Agents SDK, Pydantic
AI, LangGraph (plugin annoncé côté Temporal), Gemini. Coût : contrainte de déterminisme dans
le code de workflow, un cluster à opérer (ou Temporal Cloud), et une courbe d'apprentissage
réelle. Alternatives du même registre : DBOS (in-process, checkpoints en base — beaucoup plus
léger à opérer), Restate, Prefect.

### Claude Agent SDK (Anthropic) — pour mémoire

Python/TypeScript, paquet `claude-agent-sdk` **0.2.145** (27/08/2026), code MIT mais usage
régi par les **Commercial Terms** d'Anthropic. Boucle d'agent, outils fichiers/bash, hooks,
sous-agents, permissions, MCP, sessions reprises/forkables. Excellent pour l'agent qui
manipule un poste de travail ; **modèles Claude uniquement** (API Anthropic, Bedrock, Vertex,
Foundry, ou passerelle) — aucun chemin vers un modèle auto-hébergé. Sessions stockées en JSONL
local par défaut, donc portabilité inter-hôtes à construire.

### AWS Strands Agents et Mastra — pour mémoire

**Strands Agents** (AWS, Apache-2.0, **1.53.0** du 21/08/2026) : approche « model-driven »,
très large support de providers dont Bedrock, Ollama, LiteLLM et llama.cpp. Peu de garanties
de durabilité documentées dans les sources consultées. **Mastra** (TypeScript, YC W25,
Apache-2.0 avec répertoires `ee/` sous licence Enterprise, ~27,5 k étoiles) : workflows par
graphe (`.then()`, `.branch()`, `.parallel()`) et suspend/resume persisté. Hors stack Python,
mentionné pour complétude ; le double licenciement core/entreprise est un point de vigilance.

## Grille comparative

| Solution | Licence | Auto-hébergeable | Exécution durable | Maturité (08/2026) | Effort d'intégration | Risque de verrouillage | Adapté au cadre ? |
|---|---|---|---|---|---|---|---|
| LangGraph | MIT (Platform : Enterprise) | Oui (lib) ; plateforme = Enterprise | Checkpointing superstep + replay ; **pas de rejeu si le process meurt** | Élevée, v1.2.x, gros usage | Moyen | Faible sur la lib, élevé sur LangSmith/Platform | **Oui**, comme couche de graphe |
| Pydantic AI | MIT | Oui | Déléguée : Temporal / DBOS / Prefect / Restate (1ʳᵉ partie) | Élevée, v2.35.x, cadence forte | Faible | Très faible | **Oui**, meilleur ratio |
| MS Agent Framework | MIT | Oui | Checkpoints superstep ; API fonctionnelle expérimentale | Moyenne (GA avril 2026) | Moyen | Moyen (gravité Azure/Foundry) | Si SI Microsoft |
| Google ADK | Apache-2.0 | Oui (conteneur) | Reprise de session, `resume`/`cancel` ; pas de rejeu | Moyenne, API instable (1.x→2.x) | Moyen | Moyen (Vertex Agent Engine) | Réservé |
| OpenAI Agents SDK | MIT | Oui | Via Temporal uniquement | Moyenne (**0.x**) | Faible | Faible (LiteLLM) ; tracing OpenAI par défaut | Partiellement |
| CrewAI | MIT (AMP commercial) | Oui | `@persist` SQLite, reprise/fork ; pas de rejeu | Moyenne, forte adoption | Faible | Moyen (pression vers AMP) | Non prioritaire |
| LlamaIndex Workflows | MIT | Oui | Checkpointer + DBOS | Moyenne, packaging instable | Moyen | Faible | Possible |
| Temporal (+ LLM) | MIT | Oui (cluster) | **Oui, rejeu d'historique** | Élevée, éprouvée hors IA | Élevé | Faible | **Oui**, en socle |
| DBOS | Open source (à vérifier) | Oui (lib + Postgres) | Oui, checkpoints en base, in-process | Moyenne | Faible | Faible | Oui, alternative légère |
| Claude Agent SDK | Code MIT / Commercial Terms | Non (modèle) | Sessions reprises, locales | Élevée sur son terrain | Faible | **Élevé** (Claude only) | Non pour le socle |
| Strands Agents | Apache-2.0 | Oui | Non documentée dans nos sources | Moyenne | Faible | Faible (mais orbite AWS) | Réservé |
| Mastra | Apache-2.0 + EE | Oui (core) | Suspend/resume persisté | Moyenne | N/A (TypeScript) | Moyen (répertoires `ee/`) | Non (hors stack) |

## Ce que ça implique pour un runtime agentique

- **Séparer la boucle agentique du moteur d'exécution.** Aucun framework de cet axe ne
  fournit à lui seul la garantie « reprise exacte après crash à la 2ᵉ heure ». Le
  checkpointing d'état et le rejeu d'historique ne sont pas la même chose, et seul le
  second survit à la mort du processus.
- **Le déterminisme doit être structurel, pas espéré.** Le graphe (LangGraph, MAF,
  Workflows) ou le code Python explicite (Pydantic AI + Temporal) définit le chemin ; la
  liberté du LLM est confinée à des nœuds nommés. Tout framework qui pousse d'abord les
  handoffs autonomes (OpenAI Agents SDK, CrewAI *Crews*) inverse cette charge.
- **La granularité de reprise doit être l'enregistrement, pas le run.** Sur 200 000
  enregistrements, un checkpoint par superstep de graphe global est inutile. Le motif
  viable est : un workflow durable par lot, une activité idempotente par enregistrement,
  l'agent LLM confiné dans l'activité.
- **L'indépendance modèle se décide au niveau du framework.** Pydantic AI, ADK, Strands,
  OpenAI Agents SDK (via LiteLLM) et MAF branchent Ollama/vLLM. Le Claude Agent SDK, non.
- **La testabilité est un critère discriminant sous-estimé.** Le modèle `'test'` de
  Pydantic AI et l'environnement de test à saut temporel de Temporal permettent des tests
  CI réellement déterministes, ce qui conditionne le GitOps de l'axe 7.

## Recommandation

**Option 1 (recommandée) — Pydantic AI + Temporal (ou DBOS pour démarrer).**
Pydantic AI apporte le typage, l'agnosticisme modèle (Ollama/vLLM sans adaptateur maison),
OpenTelemetry natif et un modèle de test hors-ligne ; Temporal apporte la seule exécution
durable réelle de cet axe. L'intégration est de première partie et co-maintenue, donc ce
n'est pas un assemblage à notre charge. Coût : opérer un cluster Temporal — d'où la variante
**DBOS** (bibliothèque in-process + Postgres) pour la première itération, avec bascule
ultérieure vers Temporal si le volume l'exige. C'est l'option qui laisse la souveraineté
entièrement ouverte.

**Option 2 — LangGraph pour le graphe, Temporal ou un scheduler externe pour la durabilité.**
Justifiée si l'équipe veut un modèle de graphe explicite outillé, du time travel et un
écosystème abondant. Réserve nette : ne pas confondre les checkpointers avec de la durabilité,
et ne pas s'engager sur LangGraph Platform / LangSmith self-hosted sans avoir chiffré l'offre
Enterprise (non publique).

**Option 3 — Microsoft Agent Framework**, uniquement si le SI cible s'avère Microsoft/Azure.
Il consolide AutoGen et Semantic Kernel, ce qui règle la question de la pérennité de ces deux
projets, mais sa GA d'avril 2026 est trop récente pour être un pari par défaut sur un SI non
défini.

**Écartés, et pourquoi :**

- **Claude Agent SDK** comme socle : verrouillage modèle total, incompatible avec l'hypothèse
  de souveraineté encore ouverte. À réévaluer si le client tranche pour du SaaS Anthropic.
- **AutoGen et Semantic Kernel** : en mode maintenance déclarée, aucun nouveau projet ne doit
  démarrer dessus.
- **CrewAI** : le modèle *Flows* est correct, mais la valeur du produit migre vers AMP et la
  philosophie « équipes d'agents » ne correspond pas à un pipeline de données contraint.
- **OpenAI Agents SDK** comme socle : encore en 0.x, orienté handoffs autonomes, et il faut
  de toute façon lui adjoindre Temporal — auquel cas Pydantic AI donne plus pour le même prix.
- **Mastra** : TypeScript, hors stack ; double licence core/entreprise à surveiller.
- **Google ADK** et **Strands Agents** : gardés en veille. ADK est pénalisé par sa rupture
  1.x→2.x et sa cadence bi-hebdomadaire ; Strands par l'absence de garanties de durabilité
  dans les sources consultées.

**Points ouverts pour la synthèse transverse :** le choix Temporal vs DBOS relève de l'axe 3
et doit y être tranché une seule fois ; l'agnosticisme modèle retenu ici doit être confronté
à l'axe 8 ; le tracing (LangSmith vs Logfire vs OTLP neutre) est un point de fuite de données
à arbitrer avec l'axe 5.

## Sources

- https://pypi.org/project/pydantic-ai/ — version 2.35.1 du 27/08/2026, MIT.
- https://pydantic.dev/docs/ai/overview/ — providers supportés, OTel natif, modèle `'test'`.
- https://pydantic.dev/docs/ai/capabilities/durable_execution/overview/ — backends durables Temporal/DBOS/Prefect/Restate.
- https://pypi.org/pypi/langgraph/json — version 1.2.11, licence MIT, Python >=3.10 (date de publication non fiable).
- https://github.com/langchain-ai/langgraph — MIT, ~40,5 k étoiles, promesses durable execution / HITL / mémoire.
- https://docs.langchain.com/oss/python/langgraph/durable-execution — checkpointers disponibles et limites (pertes en mémoire, croissance non bornée).
- https://www.langchain.com/pricing-langgraph-platform — self-hosted réservé au plan Enterprise, tarif non publié.
- https://www.diagrid.io/blog/checkpoints-are-not-durable-execution-why-langgraph-crewai-google-adk-and-others-fall-short-for-production-agent-workflows — critique (parti pris vendeur) de l'assimilation checkpoint = durabilité.
- https://pypi.org/project/agent-framework/ — Microsoft Agent Framework 1.15.0 du 21/08/2026, MIT.
- https://learn.microsoft.com/en-us/agent-framework/overview/agent-framework-overview — successeur d'AutoGen et Semantic Kernel, providers dont Ollama.
- https://learn.microsoft.com/en-us/agent-framework/concepts/workflows/ — API graphe vs fonctionnelle (expérimentale), checkpoints aux supersteps, HITL.
- https://github.com/microsoft/autogen — README : projet en mode maintenance, successeur = Agent Framework.
- https://pypi.org/project/google-adk/ — ADK 2.8.0 du 26/08/2026, Apache-2.0, rupture d'API 2.0.
- https://adk.dev/ — workflows séquentiel/parallèle/boucle, graphes 2.0, providers LiteLLM/vLLM/Ollama, resume/cancel.
- https://pypi.org/project/openai-agents/ — OpenAI Agents SDK 0.22.0 du 19/08/2026, MIT.
- https://openai.github.io/openai-agents-python/ — primitives, sessions, guardrails, tracing, adaptateurs LiteLLM.
- https://temporal.io/blog/announcing-openai-agents-sdk-integration — intégration Temporal / OpenAI Agents SDK.
- https://pypi.org/project/temporalio/ — SDK Python 1.32.0, MIT ; README mentionne encore « public preview » pour l'extra openai-agents.
- https://pypi.org/project/crewai/ — CrewAI 1.15.17 du 20/08/2026, MIT.
- https://docs.crewai.com/en/concepts/flows — Flows déterministes, `@persist`, reprise et fork d'état.
- https://pypi.org/project/llama-index-workflows/ — 2.23.3 du 22/08/2026, MIT.
- https://developers.llamaindex.ai/python/llamaagents/workflows/ — modèle événementiel, checkpointing, DBOS, `llama_deploy` déprécié.
- https://pypi.org/project/claude-agent-sdk/ — 0.2.145 du 27/08/2026, code MIT.
- https://code.claude.com/docs/en/agent-sdk/overview — capacités (hooks, sous-agents, sessions, MCP) et Commercial Terms.
- https://code.claude.com/docs/en/third-party-integrations — providers : Anthropic, Bedrock, Vertex, Foundry, passerelles ; pas de modèle tiers.
- https://pypi.org/project/strands-agents/ — AWS Strands 1.53.0 du 21/08/2026, Apache-2.0, providers dont Ollama et llama.cpp.
- https://github.com/mastra-ai/mastra — Apache-2.0 + licence Enterprise sur `ee/`, suspend/resume, ~27,5 k étoiles.
- https://mastra.ai/docs — périmètre fonctionnel de Mastra (TypeScript).
