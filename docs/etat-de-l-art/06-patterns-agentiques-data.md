# Axe 6 — Patterns agentiques appliqués à la donnée

> État de l'art arrêté au **27 août 2026**. Chiffres issus de pages réellement consultées (liste en fin de document). Un chiffre non vérifié sur sa source primaire est marqué **« non vérifié »** et ne doit pas fonder de décision. Campagne de recherche écourtée pour raison de budget : les sections « décider par LLM, exécuter par code », la grille de décision, l'avis MCP et la recommandation sont complètes ; le catalogue est resserré et les lacunes sont signalées.

## Périmètre et enjeu pour le projet

Le client a tranché : **runtime agentique**. Cet axe répond donc à la seule question qui compte ensuite : *que confie-t-on effectivement au LLM, à quel moment, sous quelles contraintes ?* La réponse naïve — « l'agent lit la source et sort la donnée » — est réfutée par les mesures. Trois faits cadrent tout le reste.

1. **L'écart benchmark / production est d'un ordre de grandeur.** Spider 1.0 : >86 %. Spider 2.0, qui reproduit des environnements d'entreprise (>3 000 colonnes, dialectes multiples) : 10-17 % pour les modèles génériques. Le classement n'est remonté que grâce à des *agents* spécialisés, pas à des modèles.
2. **Les échecs sont majoritairement silencieux.** Un agent de code produit un patch dans 100 % des exécutions et ne résout la tâche que dans 44 % des cas (arXiv 2603.25764). Sur de la donnée, un résultat « plausible mais faux » est pire qu'une exception.
3. **Le coût est piloté par l'architecture, pas par le modèle.** Un même travail varie d'un facteur **139×** selon l'échafaudage (arXiv 2608.08654). Le choix de pattern pèse plus lourd que le choix de fournisseur de LLM.

## Le principe directeur : décider par LLM, exécuter par code

C'est le cœur de l'axe, et la règle à inscrire en ADR :

> **Le LLM produit des artefacts — un mapping, une requête, une fonction de parsing, une règle de validation. Ce sont ces artefacts, versionnés et testés, qui touchent le volume. Le LLM ne touche jamais chaque ligne.**

**Argument de coût.** La littérature « semantic operators » de 2026 documente exactement ce basculement. **SemBaker** (2608.06677, 07/08/2026) compile la tâche en *fonctions Python déterministes* au lieu d'un appel LLM par élément : **4,8 à 6,3× de vitesse et 5,4 à 10,7× de coût en moins** sur trois charges de 200 requêtes. **SEMA-SQL** (2604.23477) réduit de **93 %** les invocations LLM sur des jointures sémantiques par mise en lots. **Larch** (2606.07923) : **3× à 19×** de tokens en moins. **CADENZA** (2606.29151) : jusqu'à **165,7× de latence et 310,3× de coût** gagnés sur SemBench. Aucune de ces optimisations ne consiste à mieux prompter ; toutes consistent à *sortir le LLM de la boucle par ligne*.

**Argument de reproductibilité.** Le projet est en GitOps. Un pipeline qui rappelle un LLM sur chaque enregistrement n'est ni rejouable à l'identique, ni couvrable par des tests de non-régression, ni auditable. Un pipeline dont le LLM a produit *un artefact commité* est reproductible par construction : c'est l'artefact qu'on relit en revue, qu'on diffe, qu'on rejoue.

**Argument de fiabilité.** Les erreurs par élément se composent. À 97 % de justesse par ligne — valeur optimiste — un lot d'un million de lignes contient 30 000 erreurs, non déterministes donc non reproductibles. Une erreur dans un artefact généré est au contraire *systématique* : détectable une fois, par un jeu de test.

**Le pattern d'échantillonnage, concrètement.** (1) Échantillonner de façon informée — pas les 100 premières lignes, mais un échantillon stratifié : valeurs distinctes, nulls, extrêmes, encodages atypiques. (2) Faire décider le LLM sur cet échantillon : mapping, code de parsing, requête, règles de validation. (3) Valider l'artefact sur un jeu de contrôle **disjoint**, avec des assertions déterministes (cardinalités, types, unicité, totaux de contrôle). (4) Commiter l'artefact et l'exécuter au volume en code pur. (5) Ne réinvoquer le LLM que sur dérive détectée — schéma changé, taux de rejet au-delà d'un seuil — en repassant par l'étape 3.

**Limite connue.** **SemCEB** (2606.23081) montre que l'échantillonnage est robuste pour estimer la sélectivité d'opérateurs sémantiques mais **ne passe pas à l'échelle**. Il faut donc borner la taille d'échantillon et accepter qu'un cas rare non échantillonné produise un *rejet*, jamais une valeur fausse. C'est le bon arbitrage.

**Zone grise assumée.** Les **documents non structurés** : pas de schéma source à mapper, chaque document est singulier, l'inférence par unité est irréductible (axe 4). La frontière est donc : *décision par LLM à la population* pour les sources structurées, *inférence par LLM à l'unité* pour les documents — validation déterministe en aval dans les deux cas.

## Catalogue des patterns

### 1. Découverte et profilage automatique de source

L'agent explore une source inconnue (`information_schema`, OpenAPI/WSDL, échantillons de valeurs) et produit rôles de tables, sémantique des colonnes, clés candidates, cardinalités, qualité. C'est **le pattern qui rend le cadre réellement enfichable**, et le meilleur rapport valeur/coût du catalogue : on paie une fois par source, pas par exécution. **APEX-SQL** (2602.16720) montre que le profilage validant les rôles de colonnes *contre les données réelles* est ce qui fait passer un agent text-to-SQL à 51,01 % sur Spider 2.0-Snow. Fiabilité partielle : sur OADD-Bench (2608.04536), un agent atteint **0,465 de rappel** contre 0,185 en recherche directe — deux fois mieux, mais moins de la moitié des colonnes cibles trouvées ; « Do Data Agents Need Semantic Metadata? » (2605.28787) mesure **+44,9 % à +65,7 % de précision** quand des métadonnées existent : l'agent amplifie la documentation, il ne la remplace pas. **À éviter** : les bases sans nommage lisible (le rappel s'effondre) et le profilage exhaustif sur données personnelles — échantillonner et masquer. Implémentations : `postgres-mcp` (MIT) ; briques académiques (StraTyper, UrbanTrace) ; pas d'outil générique mûr identifié.

### 2. Mapping et alignement de schéma assistés par LLM

Aligner une source inconnue sur un modèle cible : correspondances, transformations, conflits sémantiques. Tâche coûteuse en expertise humaine et parfaitement adaptée au principe directeur, puisque le livrable est un fichier de mapping versionné ; le coût d'exécution est nul, il est dominé par la revue humaine. C'est le pattern le mieux mesuré : **SINT-Flow** (2607.24492, 27/07/2026) rapporte **F1 96 % sur les types d'entités, 85 % sur les attributs, 83 % sur le mapping** — la marche descend à mesure qu'on approche du livrable utile. Les gains publiés sur MIMIC-OMOP sont réels mais incrémentaux (Schemora +7,49 % HitRate@5 ; KG-RAG +35,89 % de précision ; KcMF +17,93 % de F1). **Contre-indication absolue** : appliquer un mapping sans relecture — 83 % de F1 signifie un attribut faux sur six, et un mapping faux corrompt silencieusement l'entrepôt. Sur vocabulaire codifié (santé, finance), préférer l'appariement à un référentiel. Aucune bibliothèque de production dominante identifiée : l'état de l'art est académique (Magneto, Matchmaker, LLMatch, ReMatch). **À instruire avant tout choix.**

### 3. Text-to-SQL et génération de requêtes d'extraction

Utile en génération *assistée* d'une requête qu'un humain valide et commite ; douteux en boucle fermée. **Les chiffres.** BIRD (consulté le 27/08/2026) : meilleur système public *AskData + GPT-4o* à **77,64 % dev / 81,95 % test** (entrées datées de 09/2025) contre **92,96 % humain**. Spider 2.0 (consulté le 27/08/2026, 547 exemples par piste) : **Lite 76,23 %** (Tianqiong + GLM 5.2), **DBT 65,6 %** sur 68 tâches. **Spider2-Snow affiche 96,70 %** — que je tiens pour **non exploitable** : page sans date de mise à jour, correctif de la suite d'évaluation en 10/2025, compte Snowflake d'évaluation suspendu le 12/08/2026, et 20 points d'écart inexpliqués avec Lite sur le *même corpus de questions*. Les variantes dures sont sans appel : **BIRD-Interact 24,4 %** (conversationnel) et **17,78 %** (agentique), **LiveSQLBench-Lite 44,81 %**. La validité même des benchmarks est contestée (erreurs d'annotation, CIDR 2026 p5-jin) : **non vérifié**, PDF illisible. **Contre-indications** : toute requête générée exécutée sur la production avec un compte privilégié. Deux risques documentés : le *fan-out trap* (jointure 1-N avant agrégation → CA 5× surévalué, sans erreur levée) et l'exécution de code (**CVE-2024-5565**, Vanna.AI, SQL passé à `exec()`, RCE par injection de prompt). Alternative à privilégier : **outils de domaine paramétrés plutôt que SQL généré** — **0,939 de justesse contre 0,666** sur 17 tâches clients, et des modèles de 3-8 Md suffisent (2608.22063, 22/08/2026).

### 4. Génération puis exécution de code (« code as action » / CodeAct)

L'agent écrit du Python (parsing, appel d'API, transformation) et l'exécute, au lieu d'enchaîner des appels d'outils. Gain mesuré et important : interfaces CodeAct contre outils bash bruts → **−41,6 % d'étapes, −56,3 % de tokens** à performance égale, et jusqu'à **4,7× de constance** (2608.11386, 11/08/2026) ; sur MCP, Anthropic rapporte **150 000 → 2 000 tokens, −98,7 %** (04/11/2025) ; AutoRPA −82 % à −96 %. Le coût se déplace du LLM vers l'infrastructure : bac à sable, limites CPU/mémoire/temps, egress contrôlé, supervision — coût réel et permanent. Intérêt majeur pour la donnée : le code est **lisible, testable, commitable**. **Contre-indication absolue** : l'exécution non isolée — le code généré est formellement du code fourni par un tiers non fiable, puisqu'il dépend du contenu des données lues (injection indirecte). Attention au couplage runtime : un agent entraîné avec état persistant consomme **3,5× plus de tokens** en runtime sans état, l'inverse produisant des erreurs de variable manquante dans **~80 %** des épisodes (2603.01209). Choix du bac à sable (E2B, Modal, Daytona, conteneurs jetables, WASM) : **non instruit, non vérifié**. Critère : isolation réseau par défaut + image reproductible.

### 5. Extraction structurée sous contrainte de schéma

Contraindre la sortie à un schéma (JSON Schema, grammaire, function calling) par décodage contraint plutôt que par consigne. Supprime la classe entière des erreurs de format ; le décodage par grammaire comble **71 % en médiane** de l'écart avec l'effet de contenu seul (2608.04355), et le surcoût est réglé (PSC : **700×** plus rapide sur grammaires complexes, **30×** sur JSON Schema). Coût marginal côté calcul ; non nul côté conception, le schéma devenant un artefact à versionner — ce qui sert le GitOps. **Piège central : conformité ≠ justesse.** La conformité de base mesurée est de **85,9-91,6 %**, portée à **99,0 %** par une boucle valider-réparer (2607.24371) ; mais « Where vs What » (2608.25358, 26/08/2026) montre que la fidélité structurelle se dégrade *avant* le contenu : à haute complexité, **35 % des valeurs sont placées dans le mauvais champ** (74 % pour un 7 Md) — sortie valide, sémantiquement fausse. **Contre-indications** : schémas profonds ou récursifs ; schéma placé hors prompt système (**−11 à −13 points**) ou en conflit avec lui (**−5 à −45 points**, 2608.08254) ; pénalité de répétition mal réglée, qui fait tomber la conformité de **97 % à 23 %** (2607.09791) ; usage exploratoire, la contrainte réduisant la diversité (réponse modale 41 % → 64 % sur 44 modèles, 2607.18476). **Règle** : contrainte *plus* validation Pydantic *plus* assertions métier.

### 6. Pipelines auto-réparants

L'agent détecte l'échec, diagnostique, corrige, rejoue. Cela fonctionne **quand un vérificateur encadre la reprise** : **98,8 % de réussite** contre 94,5 % pour un simple retry et 93,8 % pour une replanification complète, sous injection de fautes (2606.01416) ; un routage auto-réparant réduit de **93 %** les appels LLM du plan de contrôle (2603.01548). Le coût est imprévisible par nature — c'est le principal danger budgétaire d'un runtime agentique. **Le risque réel n'est pas la boucle infinie**, qui se borne trivialement : c'est la **réparation qui corrompt silencieusement**. Les mêmes travaux le disent : sans vérificateur, les lignes de base renvoient des sorties *« wrong-but-plausible »*. Une « correction » qui remplace un NULL par un défaut, ou relâche une contrainte de jointure pour faire passer le lot, produit un pipeline vert et un entrepôt faux. **Contre-indication** : toute réparation touchant la **sémantique** (valeurs, jointures, filtres) doit escalader, jamais s'auto-appliquer. La réparation légitime est technique et idempotente : reprise réseau, réauthentification, pagination, adaptation à un renommage *confirmé par le catalogue*. **Règle à adopter telle quelle** : *toute défaillance est soit un reroutage journalisé, soit une escalade explicite, jamais un saut silencieux*. Architecture de référence ouverte et sans dépendance fournisseur : arXiv 2608.01955 (supervision + métadonnées + historique d'incidents + **contrôles de politique déterministes** + diagnostic assisté).

### 7. MCP comme couche d'accès normalisée

Traité en section dédiée ci-dessous.

### 8. Les garde-fous indispensables

Ce ne sont pas des options : ce sont les conditions rendant un non-déterminisme admissible en production. **Échantillonnage** — le garde-fou de coût, traité en section 2. **Budget de tokens borné par tâche**, arrêt dur : le facteur **139×** entre échafaudages et les **140,2 k tokens de définitions d'outils occupant 70,1 % du contexte** avant même la tâche (SCOUT, 2608.23992) l'imposent. **Sélection dynamique d'outils** plutôt que catalogue complet : SCOUT descend à 1,3 k tokens (**−99 %** en production chez PayPal) ; Snowflake le confirme côté fournisseur (maximum 50 outils par serveur, « des nombres plus élevés dégradent la justesse de sélection »). **Boucles bornées** en itérations *et* en budget cumulé, escalade en fin de budget. **Validation croisée** : le consensus formel multi-modèles atteint **94,7 % de précision à 27 % de couverture** (2608.21962) — filtre de confiance, pas oracle. **Moindre privilège** : lecture seule, RLS en base, délais d'exécution ; un post-entraînement dédié fait tomber les erreurs d'excès d'autorité de **4,56 % à 0,79 %** (2608.18351), ce qui dit surtout qu'elles étaient à 4,56 %. **Minimisation des données transmises** : **81 à 88 % des appels d'outils** transportent par défaut des données personnelles inutiles (2608.24957) — point RGPD direct, à relier à l'axe 8. **Humain dans la boucle au point de commit de l'artefact**, pas à chaque ligne : conséquence pratique du principe directeur, on revoit un mapping, pas un million de valeurs. **Mise en cache** : prompt caching et réutilisation de KV-cache entre opérateurs (Kalypso, jusqu'à **4,57×**), ce qui impose d'ordonner les prompts partie stable en tête.

### 9. Évaluation d'un pipeline agentique

Trois niveaux distincts : jeu d'évaluation figé (entrée → sortie attendue), tests de non-régression sur prompts et artefacts commités, évaluation en ligne sur échantillon. Sans jeu d'évaluation versionné, un changement de modèle ou de prompt est un déploiement à l'aveugle ; en GitOps, ce jeu est un livrable au même titre que le code. Coût initial élevé (annotation), amorti ensuite — c'est le poste le plus souvent sous-budgété. **Limites du LLM-juge, sévères et fraîchement documentées** : biais d'ancrage sur 192 000 évaluations, où des métadonnées d'ancrage **bloquent 48 % des corrections d'erreur et inversent 10,18 % des jugements corrects** (2608.25869, 26/08/2026) ; validité de construit faible, des prédicteurs purement superficiels reproduisant **55 à 67 %** des étiquettes de juges dont **67,4 %** des votes humains MT-Bench (2608.24419) ; incohérence entre programmes équivalents à l'exécution (2608.22938). **Conséquence opérationnelle** : sur de la donnée, on dispose presque toujours d'un oracle déterministe (totaux de contrôle, cardinalités, intégrité, rejeu contre une extraction de référence). Il faut s'en servir et **réserver le LLM-juge aux seules sorties non vérifiables mécaniquement**, après calibration sur un sous-ensemble annoté (0,80 → 0,86, 2608.21057). Outillage (promptfoo, DeepEval, Ragas, Braintrust) : **non instruit, non vérifié**.

## MCP comme couche d'accès aux sources

**Où en est le protocole.** Révision courante **2026-07-28** (consultée le 27/08/2026). Le protocole est désormais explicitement **sans état** : plus de session au niveau protocole, un serveur qui a besoin d'état émet un *handle* passé en argument d'outil. Primitives serveur : Resources / Prompts / Tools ; côté client : Elicitation. Des **extensions** optionnelles négociées à l'initialisation existent, dont **Tasks** (exécution asynchrone d'opérations longues, poignées durables) — la seule qui compte vraiment pour nous, une extraction de plusieurs minutes n'entrant pas dans un appel d'outil synchrone.

**Où en est l'écosystème.** *Il faut regarder ce que le dépôt de référence a retiré, pas ce qu'il annonce.* `modelcontextprotocol/servers` ne contient plus que **sept implémentations de référence** (Everything, Fetch, Filesystem, Git, Memory, Sequential Thinking, Time) ; **PostgreSQL et SQLite ont été archivés**, comme GitHub, GitLab, Google Drive et Slack. Le README est explicite : démonstrations du protocole, **pas des solutions de production**. L'offre de production est donc côté éditeurs : Snowflake propose un serveur MCP managé en GA (Cortex Analyst, Cortex Search, exécution SQL, UDF) ; Databricks et BigQuery sont annoncés, **non vérifié** sur documentation officielle. Côté communauté, `postgres-mcp` (MIT) est sérieux : mode *restricted* en lecture seule avec limite de temps, EXPLAIN avec index hypothétiques, contrôles de santé. Un registre officiel existe (`registry.modelcontextprotocol.io`).

**Deux limites décident.** *Le volume* : MCP est un canal de contrôle, pas un canal de données. Snowflake tronque les réponses d'outils à **250 Ko**, explicitement « pour éviter de saturer la fenêtre de contexte » ; les seules définitions d'outils atteignent **140,2 k tokens, 70,1 % du contexte** avant tout travail. Toute architecture faisant transiter des lignes de données par MCP est disqualifiée — ce que confirme par la négative le pattern d'exécution de code d'Anthropic, dont les **98,7 %** d'économie viennent précisément de ne plus faire passer les résultats intermédiaires par le contexte. *La sécurité* : la spécification catalogue elle-même le confused deputy, le token passthrough (interdit — un serveur **NE DOIT PAS** accepter un jeton qui ne lui a pas été émis), le SSRF lors de la découverte OAuth (y compris vers `169.254.169.254`), le détournement de *state handle*, la compromission de serveur local, l'escalade XSS → RCE via transport stdio en architecture proxy. Ce n'est pas théorique : attaques de confiance différée à **69,5 % de succès moyen** (2608.23763) ; part des outils MCP modifiant un état externe passée de **27 % à 65 %**, protections mesurées arrêtant **moins de 30 %** des attaques (2608.17275).

**Avis.** **Adopter MCP pour le plan de contrôle uniquement, et sans en dépendre.** Concrètement : **oui** pour la découverte et le profilage, l'inspection de schéma, l'exécution de requêtes de petit volume, la consultation de catalogue — c'est là que l'interface normalisée fait gagner du temps de connecteur et que le volume reste dans les clous. **Non** pour le transfert de données : l'extraction au volume passe par du code déterministe parlant directement à la source avec ses propres identifiants. Cette séparation résout en prime le moindre privilège — compte du plan de contrôle en lecture seule et limité en temps, compte d'extraction distinct et non exposé à l'agent. **Sous condition** d'une abstraction interne : la leçon de l'archivage des serveurs officiels est que l'écosystème est instable ; les connecteurs doivent implémenter une interface à nous (`Source` → `describe()`, `sample()`, `extract()`) dont MCP n'est *qu'une* implémentation. Programmer directement contre MCP rendrait le cadre otage d'un protocole de deux ans. **Alternative sérieuse à ne pas écarter** : les « outils de domaine paramétrés » (2608.22063), où l'on expose non pas une base mais un petit jeu d'outils dont le SQL est écrit à la main et versionné — **0,939 contre 0,666**. C'est du GitOps appliqué à l'accès aux données, plus proche de l'esprit du projet que du SQL généré à la volée.

## Observabilité et évaluation d'un pipeline agentique

**Le standard n'est pas prêt.** Les conventions sémantiques GenAI d'OpenTelemetry ont migré dans un dépôt dédié (`open-telemetry/semantic-conventions-genai`) et restent au statut **« Development »** — donc non stable — au 27/08/2026 ; le dépôt ne publie aucune version taguée. Elles couvrent événements, exceptions, métriques, *model spans* et *agent spans*, avec des conventions Anthropic, OpenAI, Bedrock, Azure AI Inference **et MCP**. Instrumenter en OTel GenAI reste le bon pari d'interopérabilité, mais il faut isoler les attributs derrière une couche à soi et s'attendre à des renommages.

**Les outils, eux, sont matures et très actifs.** **Langfuse** v4.22.0 publiée le **27/08/2026** (évaluateurs configurables, SLO d'exécution d'évaluateurs, gestion des catégories de score ; MIT + édition entreprise, détail **non vérifié**). **Arize Phoenix** v20.4.0 le **26/08/2026** (évaluateur de pertinence de récupération, outillage MCP intégré). Braintrust : **non instruit**. Deux exigences priment sur le choix d'outil : l'auto-hébergement doit rester possible, la souveraineté n'étant pas tranchée (axe 8) ; et la trace doit être **rattachée au commit** de l'artefact exécuté, sinon l'observabilité ne sert pas le GitOps. Piste sérieuse à croiser avec l'axe 5 : la provenance vérifiable pour traitements pilotés par LLM (2608.25210).

## Retours d'expérience et échecs documentés

- **Gartner, 25/06/2025** : « plus de 40 % des projets d'IA agentique seront annulés d'ici fin 2027, en raison de coûts croissants, d'une valeur métier peu claire ou de contrôles de risque inadéquats ». Le communiqué renvoie un 403 ; citation reprise via Reuters. Les trois causes sont exactement celles qu'adressent les garde-fous du §8.
- **Text-to-SQL en production** (tianpan.co, 10/04/2026) : *fan-out trap* (CA 5× surévalué sans erreur levée), motifs N+1 (**501 allers-retours** au lieu d'un), contournement de RLS par comptes de service trop permissifs, **CVE-2024-5565** (RCE par `exec()`), porte dérobée à **85,81 % de succès en empoisonnant 0,44 %** des données d'entraînement, et un écart annoncé de **80-90 % en benchmark contre 10-31 % en production sans couche sémantique** (94-99 % avec). Ces derniers chiffres sont ceux du billet, **non recoupés** : ordre de grandeur seulement.
- **Échecs silencieux des agents de code** (2603.25764) : GPT-5 soumet un patch dans **100 %** des exécutions, en résout **44 %** ; les échecs sémantiques silencieux dominent **68 %** de ses exécutions ratées et **80 %** de celles de Llama 4.
- **Dispersion des coûts** (2608.08654) : **139×** d'écart entre échafaudages pour la même tâche, MCP contre CLI de 0,43× à 29×. Un chiffrage de POC ne prédit pas le coût de production.
- **Fuite de données par défaut** (2608.24957) : **81 à 88 %** des appels d'outils transportent des données personnelles non nécessaires.
- **Lacune assumée** : les post-mortems industriels nommés (entreprise identifiée, incident daté) n'ont pas été trouvés dans le temps imparti. C'est le premier complément à apporter à ce document.

## Grille de décision : quand l'agent, quand le code

| Situation | Agent LLM | Code déterministe | Pourquoi |
|---|---|---|---|
| Découvrir et décrire une source inconnue | **Oui** | Non | Une fois par source, aucune règle écrivable à l'avance ; rappel ~0,47, deux fois mieux que la recherche directe |
| Proposer un mapping source → cible | **Oui**, sur échantillon | Non | Décision unique, artefact versionnable ; F1 ~0,83 donc revue humaine obligatoire |
| Appliquer le mapping à 10 M de lignes | Non | **Oui** | Les erreurs par ligne se composent ; 5 à 10× moins cher en code compilé (SemBaker) |
| Écrire une requête d'extraction | **Oui**, en assistance | **Oui**, à l'exécution | La requête est commitée puis rejouée, jamais regénérée à chaque exécution |
| Exécuter une requête sur la production | Non | **Oui** | Compte dédié en lecture seule, RLS, délai d'exécution ; l'agent n'a pas les identifiants |
| Écrire un parseur pour un format tordu | **Oui** | Non | Code lisible, testable, commité : c'est le bon livrable |
| Parser 500 000 fichiers avec ce parseur | Non | **Oui** | Coût et reproductibilité |
| Extraire des champs d'un PDF singulier | **Oui**, sous contrainte de schéma | Non | Pas de schéma source à mapper, inférence irréductible (axe 4) |
| Valider type, cardinalité, contrainte | Non | **Oui** | Oracle déterministe disponible : aucune raison de payer un LLM ni d'accepter son incertitude |
| Décider si un lot est conforme | Non | **Oui** | Un LLM-juge se laisse ancrer (10,18 % de jugements corrects inversés) |
| Diagnostiquer et classer un échec technique | **Oui** | Non | Bonne tâche de classification sur texte d'erreur |
| Réparer un incident technique idempotent | **Oui**, borné | **Oui** | Reprise réseau, réauthentification, pagination : sans effet sémantique |
| Réparer en modifiant la sémantique de la donnée | **Jamais** | **Jamais** sans humain | La réparation silencieuse est le mode d'échec le plus coûteux |
| Détecter une dérive de schéma | **Oui**, pour qualifier | **Oui**, pour détecter | La détection est un diff de catalogue ; seule la qualification demande du jugement |
| Ordonnancer, réessayer, gérer les dépendances | Non | **Oui** | Rôle de l'orchestrateur (axe 3), pas d'un LLM |
| Résumer ou qualifier librement un contenu | **Oui** | Non | Pas d'oracle déterministe ; seul terrain légitime du LLM-juge |

## Recommandation

1. **Inscrire le principe directeur en ADR** : le LLM produit des artefacts versionnés, le code déterministe traite le volume. Toute dérogation doit être justifiée par l'absence de schéma source (documents non structurés) et rester validée en aval.
2. **Adopter en premier, dans cet ordre** : (a) découverte et profilage de source — meilleur rapport valeur/coût, coût borné, sert tous les connecteurs ; (b) extraction structurée sous contrainte de schéma avec boucle valider-réparer déterministe (85,9-91,6 % → 99,0 % de conformité) ; (c) mapping de schéma assisté, revue humaine obligatoire au commit ; (d) génération de code d'extraction exécuté en bac à sable isolé, le code étant le livrable commité.
3. **Écarter pour l'instant** : le **text-to-SQL autonome en production** — 77-82 % sur BIRD contre 93 % humain, 17-24 % sur les variantes interactives, et des modes d'échec silencieux qui corrompent sans lever d'erreur ; lui préférer des **outils de domaine paramétrés** au SQL écrit et versionné (0,939 contre 0,666). Écarter aussi la **réparation autonome à effet sémantique** (escalade obligatoire) et le **LLM-juge comme porte de qualité** partout où un oracle déterministe existe, c'est-à-dire presque partout sur de la donnée structurée.
4. **MCP** : oui pour le plan de contrôle, non pour le transport de données, toujours derrière une interface `Source` interne. Ne pas construire le cadre *sur* MCP ; le brancher *à* MCP.
5. **Instrumenter dès la première ligne** en OpenTelemetry GenAI (statut Development : isoler les attributs), avec Langfuse ou Phoenix auto-hébergés, chaque trace rattachée au commit de l'artefact exécuté.
6. **Poser trois budgets durs dans le socle** : tokens par tâche, itérations par boucle, temps d'exécution par requête. Le facteur 139× rend tout chiffrage de POC non transposable.

**Points de décision ouverts** : choix du bac à sable d'exécution ; outillage d'évaluation et de non-régression de prompts ; existence d'une bibliothèque de mapping de schéma exploitable en production ; recoupement des chiffres de production text-to-SQL ; recherche de post-mortems industriels nommés.

## Sources

- https://spider2-sql.github.io/ — classement Spider 2.0 au 27/08/2026 : Snow 96,70 % / Lite 76,23 % / DBT 65,6 % ; aucune date de mise à jour affichée.
- https://github.com/xlang-ai/Spider2 — 547/547/68 exemples ; correctif de la suite d'évaluation le 29/10/2025 ; compte Snowflake d'évaluation suspendu le 12/08/2026.
- https://bird-bench.github.io/ — BIRD : 77,64 % dev / 81,95 % test (09/2025), humain 92,96 % ; BIRD-Interact 24,4 % / 17,78 % ; LiveSQLBench-Lite 44,81 %.
- https://www.vldb.org/cidrdb/papers/2026/p5-jin.pdf — « Text-to-SQL Benchmarks are Broken » (CIDR 2026) : **PDF illisible à la consultation**, aucun chiffre repris.
- https://arxiv.org/abs/2602.16720 — APEX-SQL : 70,65 % BIRD, 51,01 % Spider 2.0-Snow ; profilage validant les rôles de colonnes.
- https://arxiv.org/abs/2608.22063 — outils de domaine paramétrés contre SQL généré (22/08/2026) : 0,939 contre 0,666 ; modèles 3-8 Md suffisants ; MCP Blueprint.
- https://arxiv.org/abs/2608.06677 — SemBaker : compilation en fonctions Python déterministes, 4,8-6,3× vitesse, 5,4-10,7× coût.
- https://arxiv.org/abs/2604.23477 — SEMA-SQL : −93 % d'invocations LLM sur jointures sémantiques.
- https://arxiv.org/abs/2606.07923 — Larch : 3-19× de tokens en moins sur prédicats sémantiques.
- https://arxiv.org/abs/2606.29151 — CADENZA : jusqu'à 165,7× latence / 310,3× coût sur SemBench.
- https://arxiv.org/abs/2606.23081 — SemCEB : l'échantillonnage est robuste mais ne passe pas à l'échelle.
- https://arxiv.org/abs/2607.23815 — Kalypso : réutilisation de KV-cache entre opérateurs, jusqu'à 4,57×.
- https://arxiv.org/abs/2608.04536 — OADD-Bench : 0,465 de rappel contre 0,185 en recherche directe.
- https://arxiv.org/abs/2605.28787 — « Do Data Agents Need Semantic Metadata? » : +44,9 % à +65,7 % de précision avec métadonnées.
- https://arxiv.org/abs/2607.24492 — SINT-Flow (27/07/2026) : F1 96 % / 85 % / 83 % (types, attributs, mapping).
- https://arxiv.org/abs/2501.08686 — KG-RAG pour le schema matching : +35,89 % précision, +30,50 % F1 sur MIMIC.
- https://arxiv.org/abs/2608.11386 — CodeAct contre bash : −41,6 % d'étapes, −56,3 % de tokens, jusqu'à 4,7× de constance.
- https://www.anthropic.com/engineering/code-execution-with-mcp — Anthropic, 04/11/2025 : 150 000 → 2 000 tokens (−98,7 %) ; obligation de bac à sable.
- https://arxiv.org/abs/2603.25764 — « Confident and Wrong » : 100 % de patchs soumis, 44 % résolus ; 68 % / 80 % d'échecs sémantiques silencieux.
- https://arxiv.org/abs/2603.01209 — persistance du runtime : 3,5× de tokens, ~80 % d'erreurs de variable manquante en cas de désaccord runtime/entraînement.
- https://arxiv.org/abs/2608.25358 — « Where vs What » (26/08/2026) : 35 % à 74 % de valeurs mal placées malgré une sortie valide.
- https://arxiv.org/abs/2607.24371 — boucle valider-réparer : conformité 85,9-91,6 % → 99,0 %.
- https://arxiv.org/abs/2608.08254 — placement du schéma : −11 à −13 points hors prompt système ; −5 à −45 points en cas de conflit.
- https://arxiv.org/abs/2607.09791 — pénalité de répétition mal réglée : conformité 97 % → 23 % sur 200 schémas réels.
- https://arxiv.org/abs/2607.18476 — la sortie structurée réduit la diversité : réponse modale 41 % → 64 % sur 44 modèles.
- https://arxiv.org/abs/2606.01416 — orchestrateurs auto-réparants : 98,8 % contre 94,5 % / 93,8 % ; sorties « wrong-but-plausible » sans vérificateur.
- https://arxiv.org/abs/2603.01548 — routage auto-réparant : −93 % d'appels LLM ; « jamais un saut silencieux ».
- https://arxiv.org/abs/2608.01955 — architecture de référence d'auto-réparation, ouverte et sans dépendance fournisseur (03/08/2026).
- https://arxiv.org/abs/2608.25869 — biais d'ancrage du LLM-juge sur 192 000 évaluations : 48 % de corrections bloquées, 10,18 % de jugements inversés.
- https://arxiv.org/abs/2608.24419 — validité de construit : 55-67 % des étiquettes reproduites par des prédicteurs superficiels, dont 67,4 % des votes MT-Bench.
- https://arxiv.org/abs/2608.21962 — consensus formel multi-modèles : 94,7 % de précision à 27 % de couverture.
- https://arxiv.org/abs/2608.21057 — calibration humaine d'un juge : 0,80 → 0,86.
- https://modelcontextprotocol.io/specification/latest — spécification MCP 2026-07-28 : protocole sans état, Resources/Prompts/Tools, Elicitation, extensions Tasks / Skills over MCP / MCP Apps.
- https://modelcontextprotocol.io/specification/2026-07-28/basic/security_best_practices — confused deputy, token passthrough interdit, SSRF, state handle hijacking, compromission de serveur local, XSS→RCE via proxy stdio, minimisation des scopes.
- https://github.com/modelcontextprotocol/servers — sept serveurs de référence ; PostgreSQL et SQLite archivés ; « reference implementations, not production solutions ».
- https://registry.modelcontextprotocol.io/v0/servers — registre officiel MCP, API v0 fonctionnelle.
- https://github.com/crystaldba/postgres-mcp — Postgres MCP Pro (MIT) : modes restricted/unrestricted, index tuning, EXPLAIN avec hypopg, contrôles de santé.
- https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-mcp — serveur MCP managé Snowflake en GA : troncature à 250 Ko, maximum 50 outils, dégradation de la sélection au-delà.
- https://arxiv.org/abs/2608.23992 — SCOUT : 140,2 k tokens (70,1 % du contexte) → 1,3 k (0,8 %), −99 % en production chez PayPal.
- https://arxiv.org/abs/2608.08654 — 139× d'écart de coût entre échafaudages ; MCP contre CLI de 0,43× à 29×.
- https://arxiv.org/abs/2608.23763 — TrustShiftProbe : attaques de confiance différée sur serveurs MCP, 69,5 % de succès moyen.
- https://arxiv.org/abs/2608.17275 — outils MCP modifiant un état externe : 27 % → 65 % ; protections arrêtant moins de 30 % des attaques.
- https://arxiv.org/abs/2608.24957 — ToolMinimize : 81-88 % des appels d'outils transportent des données personnelles inutiles.
- https://arxiv.org/abs/2608.18351 — moindre privilège appris : erreurs d'excès d'autorité 4,56 % → 0,79 %.
- https://arxiv.org/abs/2608.25210 — provenance vérifiable pour les traitements de données pilotés par LLM.
- https://github.com/open-telemetry/semantic-conventions-genai — conventions sémantiques GenAI : statut « Development », aucune version publiée ; spans modèle et agent, conventions MCP et par fournisseur.
- https://github.com/langfuse/langfuse/releases — Langfuse v4.22.0 du 27/08/2026.
- https://github.com/Arize-ai/phoenix/releases — Arize Phoenix v20.4.0 du 26/08/2026.
- https://tianpan.co/blog/2026-04-10-text-to-sql-failure-modes-production — modes d'échec en production (10/04/2026) : fan-out trap, N+1, CVE-2024-5565, porte dérobée à 85,81 %, écart 80-90 % / 10-31 % ; chiffres du billet, non recoupés.
- https://www.reuters.com/business/over-40-agentic-ai-projects-will-be-scrapped-by-2027-gartner-says-2025-06-25/ — reprise de la prévision Gartner du 25/06/2025 (le communiqué Gartner renvoie un 403).
