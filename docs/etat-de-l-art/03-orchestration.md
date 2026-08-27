# Axe 3 — Orchestration et exécution durable

## Périmètre et enjeu pour le projet

Le cadre doit extraire de la donnée depuis des sources d'un SI non défini, via des agents LLM qui exécutent réellement le travail en production. La couche d'orchestration doit donc répondre à deux besoins de nature différente :

1. **Déclencher et cadencer** des extractions (planifié, événementiel : arrivée de fichier, webhook, message), les rejouer, les backfiller, les observer.
2. **Exécuter de façon durable** des unités de travail dont la durée et la forme sont imprévisibles : un agent qui appelle un LLM, décide d'un outil, réessaie, attend une validation humaine, peut tourner des heures.

Contraintes structurantes : Python, GitOps (tout en code, déployé par CI/CD), souveraineté non tranchée — donc **auto-hébergement obligatoire comme option**, ce qui élimine d'emblée les offres SaaS-only et impose de regarder les licences de près (BSL, SSPL, AGPL ne sont pas équivalentes à Apache 2.0 pour un client public ou régulé).

## La tension orchestration data / exécution agentique

Les deux mondes ne font pas les mêmes hypothèses :

| Hypothèse | Orchestration data classique | Exécution agentique |
|---|---|---|
| Graphe | connu à l'avance, versionné | décidé au runtime par le LLM |
| Durée d'une tâche | minutes, bornée | minutes à heures, non bornée |
| Échec | déterministe, rejouable à l'identique | non déterministe (le rejeu ne redonne pas la même trajectoire) |
| Unité de reprise | la tâche | l'étape *à l'intérieur* de la tâche (appel LLM, appel outil) |
| Coût du rejeu | CPU/IO | tokens facturés, effets de bord déjà émis |

Le point dur est **la granularité de la reprise**. Un orchestrateur DAG considère une tâche comme atomique : si elle échoue à la 40ᵉ minute, il la relance depuis zéro. Pour un agent, cela signifie repayer 40 minutes de tokens et risquer de réémettre des effets de bord déjà appliqués (écriture, appel d'API tierce). Ce qu'il faut est un **journal d'exécution intra-tâche** — chaque appel LLM et chaque appel d'outil journalisé avant/après, rejeu par replay du journal — c'est exactement la définition de l'exécution durable (Temporal, Restate, DBOS).

Le second point dur est **le dynamisme**. Airflow et Argo veulent un graphe déclaré. Airflow le contourne par le *dynamic task mapping* (fan-out sur une collection calculée au runtime) — c'est d'ailleurs le motif que la fondation met en avant dans son billet « agentic workloads on Airflow » d'avril 2026 : chaque appel LLM devient une tâche mappée, nommée, loggée, rejouable. Cela marche pour un fan-out, pas pour une boucle agentique dont le nombre d'itérations et la nature des étapes ne sont pas connus avant l'itération précédente. Prefect, Windmill et les moteurs durables n'ont pas cette limite : le flux est du code ordinaire qui peut boucler et brancher.

Troisième point : **les tâches longues**. Un worker Airflow ou Celery occupé plusieurs heures immobilise un slot. Airflow répond par les opérateurs *deferrable* (la tâche rend la main au triggerer pendant l'attente) ; les moteurs durables répondent par la suspension native — le workflow ne consomme rien pendant qu'il attend une réponse ou une validation, et se réveille des heures plus tard.

Conclusion de la tension, développée en recommandation : les deux familles convergent en 2026 (Airflow ajoute human-in-the-loop et un state store, Kestra se dit « agentic », Temporal ajoute planification et scheduling), mais **aucune ne fait aujourd'hui correctement le travail de l'autre**. Ce n'est pas un accident de maturité, c'est un désaccord de modèle d'exécution.

## Panorama des solutions

### Apache Airflow 3.x

Apache 2.0, ASF, la référence du marché. 3.0.0 GA le 22 avril 2025 (architecture client-serveur via Task Execution API, versionnement des DAG, backfill géré par le scheduler, assets + *watchers* pour le déclenchement événementiel) ; 3.1.0 (25 sept. 2025, human-in-the-loop, deadline alerts) ; 3.2.0 (7 avr. 2026, partitionnement d'assets, multi-team) ; 3.3.0 (6 juil. 2026, state store AIP-103, Language Task SDK Java/Go AIP-108) ; 3.3.1 le 12 août 2026. **Airflow 2.x est EOL depuis le 22 avril 2026** — toute installation existante doit migrer. Un provider `common.ai` (avr. 2026) apporte opérateurs LLM et agents. Empreinte lourde : API server, scheduler, triggerer, workers, base de métadonnées. Excellent pour planifier/backfiller/observer, structurellement inadapté à porter la boucle agentique elle-même.

### Prefect (et Dagster, désormais même éditeur)

**Fait majeur de 2026 : Prefect a annoncé le 13 juillet 2026 l'acquisition de Dagster Labs**, l'entité combinée opérant sous le nom Prefect à partir d'août 2026 ; ~40 personnes de Dagster rejoignent, Nick Schrock quitte le projet ; Dagster OSS est annoncé comme « activement développé », Dagster+ conserve sa marque. Prefect (Apache 2.0, série 3.8.x en août 2026) définit les flux comme du Python ordinaire pouvant boucler et créer des tâches au runtime, avec reprises, états et *crash recovery* dans le moteur — c'est le plus « agent-compatible » des orchestrateurs data. Dagster (Apache 2.0, série 1.13.x) apporte le modèle asset-oriented, le partitionnement, le lignage et le catalogue — supérieur pour la gouvernance data, plus rigide pour l'exécution dynamique. Prefect possède aussi FastMCP, adopté comme SDK MCP officiel Python. **Risque à instruire : consolidation en cours, roadmap des deux produits pas encore stabilisée publiquement.**

### Temporal

MIT (serveur), ~22,6k étoiles, l'implémentation de référence de l'exécution durable : workflow déterministe + *activities* non déterministes, journalisation de chaque étape, replay après crash, suspension arbitrairement longue à coût nul, signaux, timers, versionnement du code de workflow. SDK Python de première classe. Intégrations officielles avec l'OpenAI Agents SDK et le SDK Vercel. Un **Agent Harness annoncé le 20 août 2026, explicitement pré-préview**, enveloppe Gemini / OpenAI Agents SDK / PydanticAI avec approbations sur appel d'outil, *Code Mode* et streaming d'événements — prometteur mais trop jeune pour être un socle. Contrepartie : c'est un cluster à opérer (services frontend/history/matching/worker + base Cassandra ou PostgreSQL/MySQL, Elasticsearch pour la visibilité avancée) et une discipline de déterminisme à tenir. Pas d'ordonnanceur data (pas de backfill, pas de lignage).

### Restate

Serveur sous **Business Source License 1.1** (bascule Apache 2.0 quatre ans après publication ; usage en service managé interdit), SDK sous MIT ; v1.7.7 le 26 août 2026, ~4,3k étoiles. Même promesse que Temporal — exécution durable, exactly-once, état K/V par entité, communication durable — mais dans un **binaire unique** avec son propre stockage (RocksDB), sans base externe : l'empreinte opérationnelle est de loin la plus faible du groupe. SDK Python, intégrations Pydantic AI et Google ADK. Contrôle de flux (VQueues, concurrence) depuis 1.7.0 (18 juin 2026). Écosystème et base d'exploitants nettement plus petits que Temporal ; la BSL est un point à faire valider juridiquement même si l'usage interne prévu est autorisé.

### DBOS Transact (Python)

MIT, ~1,6k étoiles. Approche minimaliste : **bibliothèque in-process**, pas de serveur, le journal d'exécution est stocké dans un PostgreSQL existant. Décorateurs `@DBOS.workflow` / `@DBOS.step`, files d'attente adossées à Postgres (pas de broker), cron durable, *durable sleep*, notifications exactement-une-fois, reprise programmatique depuis une étape donnée. Si le projet a déjà un PostgreSQL, le coût opérationnel marginal est proche de zéro — argument fort en GitOps et en contexte souverain. Limites : couplage Postgres, écosystème et outillage d'observabilité modestes, moins de recul en production que Temporal, pas d'ordonnanceur data.

### Kestra

Apache 2.0 (l'éditeur confirme explicitement ne pas changer de licence au passage en 2.0), flux déclaratifs en YAML, très orienté événementiel (triggers fichier, webhook, message). La 1.0 LTS se positionne comme « declarative agentic orchestration » : agents multi-fournisseurs (OpenAI, Gemini, Claude, Mistral, Bedrock, Vertex, Ollama), outils web/code/MCP/filesystem, approbations humaines, SLA de flux. La **2.0 est en release candidate** (v2.0.0-rc7 au 6 août 2026, pas de date GA annoncée) : moteur réécrit, workers stateless en gRPC sans accès direct à la base — donc déployables en réseau restreint ou air-gapped —, serveur MCP exposant les flux comme outils. JVM. Le YAML déclaratif est bon pour le GitOps mais contraint quand la logique d'extraction devient du vrai code Python.

### Windmill

**AGPLv3** pour la build sans flag *enterprise* (les binaires Community embarquent du code propriétaire avec restrictions) — point bloquant possible selon le client. Rythme de publication très soutenu (v1.797.0 le 27 août 2026, une release tous les 1 à 3 jours), moteur Rust rapide, scripts Python/TS transformés en workflows, webhooks, UI, planification, et depuis peu des **évaluations d'agents** (jeux de données, exécutions scorées, comparaison). Empreinte modérée (serveur + workers + Postgres). Bon candidat pour du « tout-en-un » interne, mais l'AGPL et la dépendance à un éditeur unique sont à arbitrer, et l'exécution durable y est moins formalisée que chez Temporal/Restate.

### Argo Workflows

Apache 2.0, projet CNCF **graduated** depuis décembre 2022 ; série 4.x en 2026 (4.0 le 4 févr. 2026, 4.1.2 le 21 août 2026 ; 3.7 en fin de support depuis le 14 août 2026). Orchestration de conteneurs native Kubernetes, CRD YAML — donc GitOps par construction, en particulier couplé à Argo CD. Chaque étape est un pod : isolation et scalabilité excellentes, latence de démarrage et verbosité élevées. DAG statique (avec `withItems`/`withParam` pour du fan-out), pas de journal intra-étape, pas de suspension longue économique au sens des moteurs durables. Suppose un Kubernetes maîtrisé.

### Orchestrateurs légers embarqués

**APScheduler** : MIT, 3.11.3 le 28 juin 2026 en stable ; **la refonte 4.0 reste en alpha (4.0.0a6, avril 2025) — chantier au point mort, ne pas parier dessus**. Utile pour un cron in-process, aucune durabilité réelle. **Celery** : BSD-3, 5.6.3 le 26 mars 2026, toujours vivant, mais file de tâches et non moteur de workflow : pas de reprise intra-tâche, la fiabilité repose sur l'idempotence du code et un broker à opérer. **arq** : MIT, ~3k étoiles, **explicitement en mode maintenance seule (issue #510) — à écarter pour un projet neuf**. Aucun de ces trois ne répond au critère d'exécution durable.

## Grille comparative

| Solution | Licence | Auto-hébergeable | Exécution durable | Graphe dynamique | Empreinte de déploiement | Maturité | Adapté au cadre ? |
|---|---|---|---|---|---|---|---|
| Apache Airflow 3.3 | Apache 2.0 | Oui | Non (reprise à la tâche, pas intra-tâche) ; deferrable pour l'attente | Limité (dynamic task mapping) | Lourde (API server, scheduler, triggerer, workers, DB) | Très élevée (ASF, 3.x depuis avr. 2025) | Oui, **comme couche de déclenchement** uniquement |
| Prefect 3.8 | Apache 2.0 | Oui | Partielle (états, retries, crash recovery ; pas de replay journalisé) | Oui (Python natif, tâches au runtime) | Moyenne (serveur + workers + DB) | Élevée, mais éditeur en pleine fusion | Oui, candidat de compromis |
| Dagster 1.13 | Apache 2.0 (core) | Oui | Partielle (retry de step, re-execution) | Faible (asset-oriented, déclaratif) | Moyenne | Élevée ; **gouvernance absorbée par Prefect (juil. 2026)** | Non prioritaire ici |
| Temporal | MIT (serveur) | Oui | **Oui, référence** (journal + replay, suspension illimitée) | Oui (code ordinaire) | Lourde (cluster multi-services + Cassandra/PG + ES) | Très élevée | Oui, **comme moteur d'exécution** |
| Restate 1.7 | **BSL 1.1** → Apache 2.0 à 4 ans ; SDK MIT | Oui | **Oui** (journal + replay, état K/V, exactly-once) | Oui | **Faible (binaire unique, stockage embarqué)** | Moyenne (v1.7, écosystème jeune) | Oui, meilleur ratio capacité/empreinte |
| DBOS Transact | MIT | Oui | **Oui** (checkpoints en Postgres, reprise par étape) | Oui | **Très faible (bibliothèque + Postgres existant)** | Moyenne-faible (~1,6k ★) | Oui pour un socle minimal |
| Kestra 1.x / 2.0-rc | Apache 2.0 | Oui | Partielle (reprise de tâche, SLA) | Faible (YAML déclaratif) | Moyenne (JVM ; 2.0 : workers stateless gRPC) | Élevée en 1.x ; **2.0 encore en RC** | Possible si l'on accepte le YAML |
| Windmill 1.79x | **AGPLv3** (build OSS) | Oui | Partielle | Oui (scripts) | Moyenne (serveur Rust + workers + PG) | Élevée en usage, éditeur unique | Réserve : licence |
| Argo Workflows 4.1 | Apache 2.0 | Oui (K8s requis) | Non (retry de step, pas de journal intra-étape) | Faible (`withParam`) | Moyenne, mais **exige Kubernetes** | Très élevée (CNCF graduated) | Seulement si K8s est déjà le socle |
| APScheduler / Celery / arq | MIT / BSD-3 / MIT | Oui | **Non** | s.o. | Très faible | 4.0 en alpha figée / actif / **maintenance seule** | Non comme socle |

## Ce que ça implique pour un runtime agentique

- **La reprise doit être intra-tâche.** Sans journal des appels LLM et outils, chaque crash repaie des tokens et risque de rejouer des effets de bord. Cela exclut Airflow, Argo et Celery comme *exécutants* de l'agent.
- **Les effets de bord d'extraction doivent être idempotents ou journalisés.** Un moteur durable garantit l'exactly-once sur les *steps* qu'il connaît ; il ne rend pas magiquement idempotente une écriture faite hors step. Convention à imposer dans le cadre : tout appel externe passe par un step déclaré.
- **Le non-déterminisme du LLM est incompatible avec le replay naïf.** La discipline Temporal/Restate — le workflow est déterministe, les appels LLM sont des activities dont on rejoue *le résultat enregistré*, pas l'appel — est la seule qui tienne. C'est une contrainte d'écriture du code, pas une option de configuration.
- **La suspension économique est un critère de coût réel** : validation humaine, quota fournisseur, attente d'un fichier. Un worker bloqué plusieurs heures est un slot payé pour rien.
- **GitOps** : les moteurs durables sont du code Python déployé comme n'importe quel service (image + manifeste) — très bon fit. Les orchestrateurs YAML (Kestra, Argo) sont excellents en déclaratif mais imposent leur DSL. Airflow/Prefect/Dagster déploient des DAG en Python versionnés, fit correct.
- **Observabilité** : les orchestrateurs data offrent nativement calendrier, backfill, lignage et relance manuelle ; les moteurs durables offrent l'historique d'exécution pas à pas et la reprise fine. **Ce sont deux vues complémentaires, pas redondantes.**

## Recommandation

**Position sur « une couche ou deux » : DEUX couches.** Un orchestrateur qui déclenche, planifie, backfille et observe au niveau métier ; un moteur d'exécution durable qui porte la boucle agentique. Prétendre couvrir les deux avec un outil unique conduit soit à un orchestrateur data qui perd les tokens à chaque crash (Airflow seul), soit à un moteur durable où l'on réimplémente à la main planification, backfill et lignage (Temporal seul). Le surcoût de la seconde couche est réel mais borné ; le coût de l'absence de journal intra-tâche est, lui, proportionnel au volume traité.

**Option 1 (recommandée) — Restate comme moteur durable + un déclencheur léger, Airflow ajouté plus tard si besoin.**
Restate en binaire unique porte les agents (durabilité, suspension, exactly-once, SDK Python) ; le déclenchement initial peut rester minimal (webhook + cron du serveur Restate). On n'introduit Airflow 3.3 (ou Prefect) que le jour où le besoin de backfill/lignage/calendrier métier est avéré. Deux composants au départ, trois au maximum. Réserve à lever : **la BSL 1.1 doit être validée juridiquement** (l'usage interne est autorisé, la revente en service managé ne l'est pas).

**Option 2 — DBOS Transact + Airflow 3.3.**
Si un PostgreSQL est déjà au socle, DBOS ajoute la durabilité sans aucun nouveau composant, sous MIT — c'est l'option la plus sobre et la plus sûre côté licence. Airflow 3.3 (Apache 2.0, gouvernance ASF, EOL de la 2.x déjà passé) fournit le calendrier, le backfill, le HITL et l'observabilité data. Réserve : DBOS est le projet le moins éprouvé de la sélection.

**Option 3 (repli si le client exige un seul fournisseur) — Temporal seul, ou Prefect seul.**
Temporal seul : durabilité de référence, mais cluster à opérer et fonctions data à réécrire ; l'Agent Harness est pré-préview et ne doit pas être en chemin critique. Prefect seul : bon compromis unique (Python dynamique, retries, self-hosted, Apache 2.0), mais durabilité sans replay journalisé et **incertitude de roadmap liée à l'absorption de Dagster**.

**Écartés explicitement :** *arq* (maintenance seule), *APScheduler 4* (alpha figée depuis avril 2025) et *Celery* comme socle (file de tâches, aucune reprise intra-tâche) ; *Argo Workflows* sauf si Kubernetes est déjà imposé (pas de durabilité intra-étape, verbosité YAML) ; *Windmill* par précaution de licence AGPLv3 ; *Dagster* comme cible neuve, non par défaut technique mais parce que sa gouvernance vient de changer de mains ; *Kestra 2.0* tant qu'il est en RC.

**Points de décision ouverts :** acceptabilité de la BSL 1.1 (Restate) et de l'AGPLv3 (Windmill) chez le client ; existence ou non d'un PostgreSQL et d'un Kubernetes au socle ; besoin réel de backfill et de lignage data — s'il est nul, la couche orchestrateur peut être différée ; stabilisation de la roadmap Prefect/Dagster à surveiller sur les prochains mois.

## Sources

- https://airflow.apache.org/blog/airflow-three-point-oh-is-here/ — annonce GA d'Airflow 3.0 (22 avril 2025) : Task Execution API, versionnement des DAG, assets et watchers.
- https://airflow.apache.org/blog/ — index des billets : dates et contenus des 3.1.0, 3.2.0, 3.3.0, provider `common.ai`, billet « agentic workloads ».
- https://airflow.apache.org/docs/apache-airflow/stable/installation/supported-versions.html — cycle de vie des versions ; EOL d'Airflow 2 au 22 avril 2026.
- https://github.com/apache/airflow/releases — publications récentes (3.3.0, 3.3.1, SDK Java beta).
- https://dagster.io/blog/prefect-is-acquiring-dagster — annonce du rachat de Dagster par Prefect (13 juillet 2026) et engagements sur l'OSS.
- https://github.com/dagster-io/dagster/releases — série 1.13.x, rythme de publication.
- https://www.prefect.io/ — positionnement 2026 : durabilité, workflows dynamiques, modèles de déploiement, FastMCP.
- https://github.com/PrefectHQ/prefect/releases — série 3.8.x.
- https://temporal.io/blog/temporal-agent-harness-durable-agent-infrastructure — Agent Harness (20 août 2026), maturité pré-préview, intégrations.
- https://github.com/temporalio/temporal — licence MIT du serveur, volumétrie du projet.
- https://github.com/restatedev/restate et .../releases — architecture, cas d'usage agents, versions jusqu'à 1.7.7 (26 août 2026).
- https://raw.githubusercontent.com/restatedev/restate/main/LICENSE — BSL 1.1, bascule Apache 2.0 à quatre ans, restriction service managé.
- https://github.com/dbos-inc/dbos-transact-py — DBOS Transact Python : MIT, workflows durables sur Postgres, files et cron.
- https://kestra.io/1-0 — Kestra 1.0 LTS, positionnement « declarative agentic orchestration », Apache 2.0.
- https://kestra.io/blogs/kestra-2-0-almost-here — Kestra 2.0 en RC (rc7 au 6 août 2026), workers stateless gRPC, MCP.
- https://github.com/windmill-labs/windmill/releases — v1.797.0 (27 août 2026), évals d'agents, cadence de publication.
- https://github.com/windmill-labs/windmill/blob/main/LICENSE — modèle AGPLv3 + enterprise.
- https://endoflife.date/argo-workflows — Argo Workflows 4.1.2 (21 août 2026), fenêtres de support.
- https://www.cncf.io/projects/argo/ — statut CNCF graduated.
- https://pypi.org/project/APScheduler/ — 3.11.3 stable (28 juin 2026), 4.0 toujours en alpha.
- https://pypi.org/project/celery/ — Celery 5.6.3 (26 mars 2026), BSD-3.
- https://github.com/python-arq/arq — mode maintenance seule.
- https://github.com/inngest/inngest — Inngest : SSPL + DOSP, auto-hébergement, durable functions (non retenu, licence).
- https://docs.langchain.com/oss/python/langgraph/durable-execution — checkpointers LangGraph (mémoire/SQLite/Postgres), portée limitée à la persistance d'état.
- https://www.reactify-solutions.com/articles/durable-ai-agents-2026 — comparatif tiers Temporal / Inngest / DBOS / Restate ; source secondaire, recoupée avec les dépôts.
