# Axe 8 — Souveraineté, choix du LLM et conformité

> **Ce document n'est pas un avis juridique.** Il distingue ce qui est établi (texte publié, tarif public consulté) de ce qui relève de l'interprétation. Sources consultées le **27 août 2026**. Le budget de recherche ayant été écourté, les points non vérifiés sont signalés « non trouvé » plutôt qu'estimés.

## Périmètre et enjeu pour le projet

Le cadre fait tourner des agents LLM **en production** : le modèle voit la donnée métier réelle. D'où trois conséquences — le fournisseur de LLM devient un **sous-traitant RGPD** ; il devient une **dépendance d'exploitation** (disponibilité, tarif, dépréciation de modèle, reproductibilité) ; et comme **aucun SI cible n'est défini**, le cadre doit supporter *les trois* postures sans réécriture. C'est la contrainte de conception n°1 de cet axe. L'objectif n'est pas de trancher pour le client, mais de rendre la décision **réversible**.

## Les trois postures d'hébergement

| Critère | (a) API SaaS externe | (b) Auto-hébergé chez le client | (c) Hybride |
|---|---|---|---|
| Où va la donnée métier | Chez le fournisseur, souvent hors UE | Ne sort jamais du périmètre | Reste interne ; seuls schémas / métadonnées sortent |
| Qualité de raisonnement | La meilleure disponible | Bonne, un cran en dessous sur le raisonnement long | Meilleur modèle pour concevoir, modèle local pour exécuter |
| Coût à faible volume | Très bas (à l'usage) | Prohibitif (GPU payé même à vide) | Bas |
| Coût à très fort volume | Linéaire, peut exploser | Plafonné par le parc GPU | Intermédiaire |
| Délai de mise en service | Heures | Semaines à mois (GPU, MLOps) | Semaines |
| Compétence requise | Faible | Élevée | Moyenne |
| Reproductibilité | Faible : le modèle change sous le même alias | **Forte** : poids figés, versionnables | Forte sur la partie interne |
| Ce que ça interdit | Données classifiées ; secteurs sans clause d'hébergement UE | Les modèles propriétaires frontière | Les cas où la donnée elle-même doit être raisonnée par le meilleur modèle |
| Sous-traitance RGPD | Contrat art. 28 + analyse de transfert hors UE | Aucun sous-traitant supplémentaire | Limitée aux données non personnelles, **si l'anonymisation tient** |

**(a)** Bonne posture de démarrage, acceptable en secteur contraint **à condition** d'une résidence UE et d'une rétention nulle — les deux sont désormais contractualisables (Anthropic expose `inference_geo`, majoration ×1,1 ; Mistral facture l'inférence régionale UE +10 %). Piège : **une région d'inférence n'est pas une immunité juridique** ; un fournisseur soumis au droit américain le reste où que soient ses serveurs.

**(b)** La seule posture qui supprime réellement le transfert. Elle convertit un coût variable en coût fixe, et une dépendance contractuelle en dépendance de compétence. Seule à offrir la **reproductibilité forte** : les poids sont un artefact figé donc versionnable — exactement l'alignement recherché avec la contrainte GitOps du projet.

**(c)** Ne tient que si la frontière est nette : le modèle externe voit le **schéma** (noms de colonnes, types, cardinalités), jamais les **valeurs**. Dès qu'un échantillon de lignes réelles sort, on est en posture (a) avec un vernis. Découpage qui tient pour notre cas : *raisonnement sur schéma et génération de règles d'extraction en externe, application des règles aux données en interne*.

## Modèles à poids ouverts utilisables en production

Vérifié sur les fiches Hugging Face officielles. Attention : un tag `license: other` signale presque toujours une licence maison **potentiellement restrictive**, à faire lire avant tout usage commercial.

| Modèle | Taille | Licence | Multimodal | Contexte |
|---|---|---|---|---|
| `Qwen/Qwen3.8-27B` (+`-FP8`) | 27 B dense | **Apache-2.0** | **Oui** (images, vidéo) | 262 144 natif, ~1 M en RoPE scaling |
| `mistralai/Mistral-Small-4-119B-2603` | 119 B | **Apache-2.0** | — | non relevé |
| `mistralai/Mistral-Large-3-675B-Instruct-2512` | 675 B | **Apache-2.0** | — | non relevé |
| `mistralai/Ministral-3-3B / -8B / -14B` | 3 / 8 / 14 B | **Apache-2.0** | — | non relevé |
| `ibm-granite/granite-4.2-3b / -8b / -30b` | 3 / 8 / 30 B | **Apache-2.0** | — | non relevé |
| `deepseek-ai/DeepSeek-V4-Flash-0731` | 284 B / 13 B actifs | **MIT** | non documenté | 1 M |
| `deepseek-ai/DeepSeek-V4-Pro` | 1,6 T / 49 B actifs | **MIT** | non documenté | 1 M |
| `zai-org/GLM-5.2`, `GLM-5.3-Flash` | non divulguée | **MIT** | non documenté | 1 M |
| `Qwen/Qwen3.8-2.4T-A95B` | 2,4 T / 95 B actifs | `qwen3.8-max` — **restrictions non vérifiées** | Non (texte seul) | 262 k → ~1 M |
| `mistralai/Mistral-Medium-3.5-128B`, `moonshotai/Kimi-K3` | 128 B / n.r. | `other` — **à écarter par défaut** | — | — |

**Parsing documentaire.** À poids ouverts : `baidu/Unlimited-OCR` (MIT), `ATH-MaaS/OvisOCR2` (Apache-2.0), `dots-studio/dots3-note-prev` (Apache-2.0). En managé : **Mistral OCR 4.1** à **4 $/1 000 pages** (5 $ en mode Document AI). À noter : `Qwen3.8-27B` étant nativement vision-langage, il couvre à la fois le parsing et l'extraction — cela réduit le parc de modèles à exploiter, ce qui compte beaucoup en posture (b).

**Parc minimal auto-hébergé recommandé :** `Qwen3.8-27B-FP8` en modèle principal (~27 Go de poids en FP8, tient sur un H100 80 Go avec marge pour le cache KV) + un `Ministral-3-8B` ou `granite-4.2-8b` pour le routage et la classification à faible enjeu.

## Serveurs d'inférence

| Serveur | Cible | Sortie structurée contrainte | Verdict |
|---|---|---|---|
| **vLLM** | Production | **Oui** — `xgrammar`/`guidance`, JSON Schema, regex, **EBNF**, `choice`, *structural tags* ; actif par défaut | **Défaut recommandé.** PagedAttention, *continuous batching*, *prefix caching*, FP8/INT8/INT4/GPTQ/AWQ, parallélismes TP/PP/DP/EP, API OpenAI-compatible |
| **SGLang** | Production | **Oui** — `xgrammar` (défaut), `outlines`, `llguidance` ; mêmes formes | Alternative sérieuse, souvent en tête sur les MoE récents |
| **TensorRT-LLM** (v1.3.0rc25) | Production NVIDIA | Oui (*guided decoding*) | Le plus rapide sur NVIDIA, mais couplage constructeur et compilation par modèle. À réserver si le débit est le facteur limitant |
| **Ollama** | Poste de travail | Non documenté au README | Excellent en développement local ; **aucun positionnement production** |
| **llama.cpp** (`llama-server`) | Poste de travail / edge / CPU | **Oui** — grammaires **GBNF** | Indispensable sans GPU ; pas la cible d'un service concurrent |

**Point critique pour nous :** l'extraction structurée doit être *garantie*, pas espérée. Les trois moteurs de production contraignent au niveau du décodage — le modèle ne **peut pas** produire un JSON hors schéma. C'est un avantage net sur les API SaaS, et il pousse à traiter la contrainte de schéma comme une fonctionnalité **du cadre**, pas du fournisseur.

## Analyse de coût : API contre auto-hébergement

**Hypothèses explicites :** (H1) mix 90 % entrée / 10 % sortie, typique d'une extraction — prix mixte `0,9·P_in + 0,1·P_out` ; (H2) la production exige **2 GPU minimum** pour la redondance ; (H3) H100 80 Go entre **2,20 $/h** (DeepInfra dédié) et **3,99 $/h** (Lambda H100 SXM), RunPod PCIe *secure* à 2,89 $/h — valeur de travail **3,00 $/h** ; (H4) 730 h/mois ⇒ **4 380 $/mois pour 2 GPU** ; (H5) l'exploitation (MLOps, supervision, montées de version) est **non chiffrée** et probablement du même ordre la première année ; (H6) le débit réel sous charge `T` **n'a pas été mesuré** — c'est la variable la plus incertaine, à établir par un banc d'essai.

**Formule.** Volume d'équilibre `V* = C_self / P_mixte`, sous contrainte de capacité `V_max = T × 730 × 3600`. L'auto-hébergement n'est rentable que si **V\* < V_max** : il faut que le parc puisse physiquement absorber le volume d'équilibre.

| Référence API | P_in / P_out ($/M) | P_mixte | V\* (4 380 $/mois) | Débit agrégé requis |
|---|---|---|---|---|
| Mistral Small 4 | 0,15 / 0,60 | 0,195 | **22,5 Md tok/mois** | ~8 560 tok/s |
| Gemini 3.5 Flash-Lite | 0,30 / 2,50 | 0,52 | 8,4 Md tok/mois | ~3 200 tok/s |
| Claude Sonnet 5 | 2,00 / 10,00 | 2,80 | **1,56 Md tok/mois** | ~595 tok/s |
| gpt-5.6-terra | 2,00 / 12,00 | 3,00 | 1,46 Md tok/mois | ~555 tok/s |
| Claude Opus 5 | 5,00 / 25,00 | 7,00 | 0,63 Md tok/mois | ~238 tok/s |

**Lecture — résultat le plus important de cet axe.** Face aux **petits modèles d'API bon marché**, l'auto-hébergement **n'est jamais rentable sur le seul coût** : le débit requis (plusieurs milliers de tokens/s) dépasse ce que 2 GPU servent ; on rachèterait des GPU avant d'atteindre l'équilibre. Face aux **modèles haut de gamme**, la bascule est atteignable : ~**1,5 Md tokens/mois** contre un Sonnet 5, ~0,6 Md contre un Opus 5 — soit un pipeline documentaire lourd et permanent, pas un batch nocturne. Et les leviers tarifaires côté API **repoussent encore la bascule** : remise *batch* de 50 % (OpenAI, Anthropic, Gemini) et lecture de cache à 10 % du prix d'entrée, décisive sur une extraction à long préfixe stable (consignes + schéma cible).

**Conclusion : on n'auto-héberge pas pour économiser, mais pour la souveraineté, la reproductibilité et la prévisibilité budgétaire.** Le coût est une conséquence assumée, jamais un argument de vente.

## Rendre le fournisseur de LLM interchangeable

**Principe : le fournisseur est une ligne de configuration versionnée dans Git, jamais un `import` Python.** Trois couches :

1. **Port interne.** Le code métier ne connaît qu'une interface maison `ModelPort`, exprimée dans notre vocabulaire : `extract(document, schema) -> objet validé`. Elle prend un **schéma Pydantic** et rend un objet **validé côté cadre**. Aucun module métier n'importe `openai`, `anthropic` ou `mistralai`.
2. **Adaptateur.** Une implémentation par famille, mais **une seule dans le cas nominal** : un client HTTP **OpenAI-compatible**, plus petit dénominateur commun réel du marché — il couvre les API SaaS, vLLM, SGLang, llama.cpp et Ollama.
3. **Passerelle.** **LiteLLM** en *proxy* **auto-hébergé** (plus de 100 fournisseurs derrière le format OpenAI ; routage, *fallbacks*, budgets par clé et équipe, suivi de coût, garde-fous). Le mode proxy est préférable au SDK : il sort la configuration fournisseur du code applicatif et donne un point unique d'application des règles. **OpenRouter** est l'alternative managée — elle sait filtrer par politique de rétention et router vers des endpoints UE, mais elle **ajoute un maillon à la chaîne de sous-traitance RGPD**, ce qui est rédhibitoire pour un client contraint.

**Ce qui n'est PAS portable, et comment le traiter :**

| Fonctionnalité | Écart réel | Traitement dans le cadre |
|---|---|---|
| **Sortie structurée** | Contrainte de décodage *garantie* côté vLLM/SGLang ; côté API SaaS, degré de garantie et sous-ensemble de JSON Schema supportés variables | **Ne jamais faire confiance au fournisseur** : valider en sortie avec Pydantic, relancer avec l'erreur de validation. La contrainte fournisseur est une optimisation, la validation est la garantie |
| **Cache de prompt** | Sémantiques incompatibles : Anthropic exige des points de rupture explicites (`cache_control`, écriture 1,25×/2×, lecture 0,1×) ; OpenAI et Gemini sont largement implicites ; **aucun équivalent en auto-hébergé** hormis le *prefix caching* vLLM, gratuit mais non pilotable | Isoler dans l'adaptateur, et imposer une discipline de prompt : **préfixe stable en tête** (consignes + schéma), variable en queue. Elle profite à tous les backends |
| **Appels d'outils** | Format et fiabilité variables ; coût en tokens du préambule d'outillage variable | Normaliser via LiteLLM ; limiter le nombre d'outils exposés par appel |
| **Fenêtre de contexte** | De 262 k (Qwen3.8-27B natif) à 1 M (DeepSeek-V4, GLM-5.2, Claude 4.6+) | Déclarée par modèle dans la config ; le cadre **découpe** en fonction, il ne suppose jamais |
| **Tokenizer** | Un même texte ne donne pas le même compte : Anthropic documente ~+30 % sur les modèles 4.7+ vs précédents | Jamais de budget en tokens codé en dur ; toujours passer par le comptage du fournisseur |
| **Raisonnement / *thinking*** | Paramètres propriétaires (`reasoning_effort` chez Qwen ; Qwen3.8-2.4T **impose** le mode thinking) | Exposer un niveau abstrait (`bas`/`moyen`/`haut`) traduit par l'adaptateur |

**Conséquence GitOps :** un `models.yaml` versionné décrit chaque profil — fournisseur, identifiant de modèle **épinglé à une version datée**, fenêtre, capacités, coût unitaire. Changer de posture d'hébergement = changer ce fichier et rejouer la suite d'évaluation. C'est le test qui prouve que l'abstraction tient.

## Cadre réglementaire : RGPD et AI Act

**RGPD — établi.** *Sous-traitance (art. 28)* : tout fournisseur voyant de la donnée personnelle est sous-traitant — contrat écrit, sous-traitants ultérieurs, instructions documentées ; une passerelle managée ajoute un maillon. *Transferts (chap. V)* : une région UE réduit le risque sans l'éliminer si le fournisseur relève d'un droit extraterritorial. *Minimisation (art. 5)* : argument fort pour la posture (c) — n'envoyer que le schéma **est** une mesure de minimisation. *Décision automatisée (art. 22)* : s'active si l'extraction alimente une décision à effet juridique sur une personne — **à qualifier avec le client**, c'est un point ouvert du cadrage.

**Travaux récents (non définitifs).** Le CEPD a mis en consultation les **Guidelines 02/2026 (anonymisation)** et **03/2026 (*web scraping* en IA générative)**, ouvertes du **8 juillet au 30 octobre 2026** — donc pas encore adoptées. La CNIL a publié le **20 juillet 2026**, avec le Conseil de l'IA et du Numérique, une **note exploratoire « IA agentique et données personnelles »** : directement pertinente pour un runtime agentique et à lire avant toute AIPD (contenu non récupéré dans le budget).

**AI Act — établi** (chronologie officielle) : **2 février 2025** interdictions et littératie IA ; **2 août 2025** obligations GPAI, gouvernance, sanctions ; **2 août 2026** majorité des dispositions restantes et bacs à sable nationaux opérationnels ; **2 août 2027** mise en conformité des GPAI antérieurs au 2/8/2025.

**AI Act — à confirmer, interprétation.** La page de la Commission européenne consultée mentionne un paquet **« AI Omnibus » entré en vigueur le 27 juillet 2026**, qui reporterait l'application aux systèmes à haut risque de l'**annexe III au 2 décembre 2027** et de l'**annexe I (produits) au 2 août 2028**. **Cette information n'a pas pu être recoupée** sur une seconde source, et le portail `artificialintelligenceact.eu` n'en fait pas état. **À vérifier au JOUE avant tout engagement contractuel — ne pas l'opposer à un tiers en l'état.**

**Sommes-nous « haut risque » ?** Analyse, pas conclusion. Un cadre générique d'extraction **n'est pas en soi** à haut risque : la qualification se fait **par la finalité d'usage** listée à l'annexe III (biométrie, infrastructures critiques, éducation, emploi, accès aux services essentiels, migration, justice). **Le risque bascule avec le déploiement, pas avec le cadre** : le même connecteur devient potentiellement haut risque s'il alimente un tri de candidatures ou une décision d'accès à une prestation sociale. **Conséquence de conception :** la qualification AI Act doit être un **attribut de chaque déploiement**, déclaré dans la configuration versionnée, jamais une propriété du dépôt. Prévoir dès maintenant les artefacts qu'exigerait l'annexe IV (documentation technique, journalisation, supervision humaine, gestion des risques) : ils sont de bonne pratique même hors obligation.

## Options souveraines européennes

- **SecNumCloud (ANSSI).** Sont **en cours de qualification** à la date de consultation : Adista, Bleu SAS, BLUE, CEGEDIM.CLOUD, Cloud Temple, Ecritel, Free Pro, GIP Mipih, ITS Integra, NumSpot, NRB, Orange Business, OVH SAS, PROLIVAL, Scaleway, Scalingo. **Liste des offres déjà qualifiées et version courante du référentiel : non trouvées** — à vérifier sur `cyber.gouv.fr` avant toute mention en appel d'offres. Point dur : **la disponibilité de GPU dans un périmètre SecNumCloud est plus rare que la qualification elle-même**, et c'est souvent le vrai facteur limitant.
- **Modèles européens.** **Mistral AI** publie une large part de son catalogue en **Apache-2.0** (`Mistral-Large-3-675B`, `Mistral-Small-4-119B`, `Ministral-3`, `Devstral-Small-2-24B`) : cela rend possible une posture (b) **avec un modèle européen**, combinaison rare et forte en argumentaire. Réserves : `Mistral-Medium-3.5-128B` et `Devstral-2-123B` sont sous licence `other`, et la page tarifaire indique que **les déploiements commerciaux de certains modèles ouverts requièrent une licence Mistral distincte** — à faire lire par un juriste. Mistral propose aussi l'inférence régionale UE (+10 %) et l'offre d'infrastructure « Mistral Compute » (annoncée le 11 juin 2025), dont les garanties de résidence précises **n'ont pas été trouvées**.
- **Hébergeurs GPU européens.** Scaleway publie des tarifs en euros (L4 à 0,79 €/h ; H100 et HGX B300 en PAR-2, **prix non relevés**). OVHcloud annonce L4, L40S, H100, H200 et des services AI Deploy / AI Endpoints avec mode batch — **tarifs non collectés**.
- **Trois niveaux de « souveraineté » à ne pas confondre :** (1) *résidence des données* — les octets sont en UE, le plus facile et le moins protecteur ; (2) *immunité juridique* — l'exploitant échappe au droit extraterritorial, c'est la cible de SecNumCloud ; (3) *autonomie technologique* — on peut continuer sans le fournisseur. **Seule la posture (b) avec un modèle Apache-2.0/MIT atteint le niveau 3.**

## Risques de sécurité spécifiques

1. **Injection de prompt indirecte — le risque n°1 de ce projet.** Notre entrée est du texte contrôlé par un tiers : un PDF, un courriel, voire un **nom de colonne** en base peut porter des instructions (LLM01 du Top 10 OWASP pour applications LLM ; le projet a migré vers OWASP GenAI Security Project et la version courante **n'a pas pu être confirmée**). Parades à intégrer au cadre : traiter **tout contenu extrait comme une donnée, jamais comme une instruction** (délimitation stricte, consigne système d'ignorer les directives du document) ; **sortie contrainte par grammaire**, qui réduit drastiquement la surface d'attaque et constitue le second argument fort pour vLLM/SGLang ; **moindre privilège** — l'agent qui lit ne détient pas les droits d'écriture sur la cible, séparer agent de lecture et agent d'écriture ; **validation en sortie et plafonds** sur les volumes écrits.
2. **Fuite par les journaux et la télémétrie** — le risque le plus fréquent en pratique. Les passerelles et les intégrations d'observabilité journalisent les prompts complets par défaut. **Interdire la journalisation du contenu par défaut** ; n'autoriser qu'identifiants de corrélation et compteurs de tokens ; activation du contenu explicite, tracée et bornée dans le temps.
3. **Rétention côté fournisseur** — vérifiable et contractualisable : Anthropic documente un régime *zero data retention* et l'absence d'entraînement sur les données retenues sans autorisation expresse ; OpenRouter permet de restreindre le routage aux fournisseurs à rétention nulle. **En faire un critère d'éligibilité fournisseur, exprimé dans la configuration.**
4. **Reproductibilité et dérive de modèle** — un alias comme `claude-sonnet-5` ou `gpt-5.6-terra` ne désigne pas des poids figés, et les fournisseurs déprécient des modèles. Une extraction validée en recette peut dériver silencieusement. Parades : **épingler les identifiants datés**, conserver un jeu d'évaluation *golden* rejoué en CI à chaque changement, et journaliser le modèle réellement utilisé dans les métadonnées de chaque extraction.
5. **Chaîne d'approvisionnement des poids** — en posture (b), les poids sont un artefact de production : miroir interne, vérification d'empreintes, et méfiance envers les innombrables ré-uploads communautaires (« abliterated », « uncensored ») qui saturent les classements Hugging Face sans garantie d'intégrité.

## Recommandation

**Pas de réponse unique — une posture par profil de client :**

| Profil | Posture | Configuration cible |
|---|---|---|
| **PME / non régulé, pas de donnée personnelle** | **(a) API SaaS** | Modèle économique (Mistral Small 4, Gemini Flash-Lite), remise batch et cache de prompt activés, rétention nulle contractualisée. **Ne pas auto-héberger** : jamais rentable à cette échelle |
| **ETI / régulé « souple » (banque de détail, assurance, industrie)** | **(c) Hybride, sinon (a) avec résidence UE** | Raisonnement sur schéma en externe, application aux valeurs en interne ; endpoint régional UE (+10 % constatés chez Mistral et Anthropic) ; passerelle **LiteLLM auto-hébergée**, jamais un intermédiaire managé |
| **Secteur public, santé, défense, données classifiées** | **(b) Auto-hébergé, sans exception** | `Qwen3.8-27B-FP8` (Apache-2.0, multimodal) ou `Mistral-Small-4-119B` (Apache-2.0, éditeur européen) sur vLLM, en infrastructure qualifiée ou on-premise. Budget ~4 400 $/mois de GPU pour 2×H100 redondés, **plus** l'exploitation non chiffrée |
| **Très fort volume documentaire permanent (> ~1,5 Md tokens/mois)** | **(b)**, même hors contrainte réglementaire | La bascule économique est franchie face aux modèles haut de gamme. À valider par un banc d'essai de débit **avant** engagement |

**Transverse, valable pour tous les profils :** (1) **construire pour (b) dès le départ, même en démarrant en (a)** — le port `ModelPort`, la validation Pydantic systématique et un `models.yaml` versionné coûtent peu au début, alors que partir couplé à une API et vouloir rapatrier ensuite est un chantier de réécriture ; (2) **un client HTTP OpenAI-compatible unique** comme adaptateur nominal, LiteLLM en proxy auto-hébergé pour routage, budgets et fallbacks ; (3) **contrainte de schéma côté serveur *et* validation côté cadre** — la première est une optimisation portable-au-mieux, la seconde est la garantie ; (4) **journalisation du contenu désactivée par défaut**, rétention nulle exigée, identifiants de modèle épinglés, jeu *golden* en CI.

**Points de décision à remonter au porteur du projet :** y a-t-il des **données à caractère personnel** dans le périmètre (sinon la moitié de cet axe devient sans objet et (a) s'impose) ? Quels **secteur et finalité d'usage aval** — c'est cela qui déclenche la qualification « haut risque », pas notre code ? Quel **ordre de grandeur de volumétrie**, sans lequel l'analyse de coût reste paramétrique ? Le client a-t-il, ou peut-il acquérir, une **compétence GPU/MLOps** interne — c'est là que la posture (b) échoue le plus souvent, bien plus que sur la technique ? Enfin, **le report éventuel des obligations « haut risque » de l'AI Act doit être vérifié au JOUE** avant d'être opposé à quiconque.

## Sources

Toutes consultées le 27 août 2026.

- <https://artificialintelligenceact.eu/implementation-timeline/> — chronologie officielle d'application de l'AI Act, date par date ; ne mentionne aucun report.
- <https://artificialintelligenceact.eu/> — actualités du portail ; confirme l'absence de mention d'un report des obligations haut risque.
- <https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai> — **seule** source mentionnant l'« AI Omnibus » (en vigueur 27/07/2026) et les dates du 2/12/2027 et 2/08/2028. Non recoupée.
- <https://www.edpb.europa.eu/our-work-tools/general-guidance/guidelines-recommendations-best-practices_en> — Guidelines CEPD 02/2026 (anonymisation) et 03/2026 (scraping IA générative), consultation 8/07–30/10/2026 ; 01/2025 (pseudonymisation).
- <https://www.cnil.fr/fr/intelligence-artificielle> — note exploratoire « IA agentique et données personnelles » du 20/07/2026 ; outil de traçabilité des modèles ouverts mis à jour le 26/08/2026.
- <https://cyber.gouv.fr/secnumcloud> — présentation SecNumCloud et liste des 16 prestataires en cours de qualification. Version du référentiel et offres qualifiées : non trouvées.
- <https://huggingface.co/api/models?sort=trendingScore&limit=60&filter=text-generation> — état réel du catalogue ouvert en août 2026 (licences, dates, téléchargements).
- <https://huggingface.co/api/models?author=Qwen&sort=createdAt&direction=-1&limit=45> et <https://huggingface.co/api/models?author=mistralai&sort=createdAt&direction=-1&limit=40> — catalogues et licences par modèle ; confirment Apache-2.0 sur Mistral-Large-3, Mistral-Small-4 et Ministral-3.
- <https://huggingface.co/api/models?pipeline_tag=image-text-to-text&sort=trendingScore&limit=40> — modèles multimodaux pour le parsing documentaire.
- <https://huggingface.co/Qwen/Qwen3.8-27B/raw/main/README.md> — 27 B dense, Apache-2.0, vision-langage natif, 262 144 tokens, vLLM/SGLang recommandés.
- <https://huggingface.co/Qwen/Qwen3.8-2.4T-A95B/raw/main/README.md> — 2,4 T/95 B actifs, licence `qwen3.8-max`, texte seul, mode thinking imposé.
- <https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro/raw/main/README.md> — MIT ; V4-Pro 1,6 T/49 B actifs, V4-Flash 284 B/13 B actifs, contexte 1 M.
- <https://huggingface.co/zai-org/GLM-5.2/raw/main/README.md> — MIT, contexte 1 M, versions minimales vLLM 0.23.0 / SGLang 0.5.13.
- <https://developers.openai.com/api/docs/pricing> — tarifs OpenAI par modèle (entrée, entrée en cache, sortie) et remise batch 50 %.
- <https://platform.claude.com/docs/en/about-claude/pricing> — tarifs Claude, multiplicateurs de cache (1,25×/2× écriture, 0,1× lecture), batch 50 %, majoration ×1,1 pour l'inférence régionale.
- <https://platform.claude.com/docs/en/manage-claude/data-residency> — paramètre `inference_geo`, restrictions par workspace, limites actuelles.
- <https://platform.claude.com/docs/en/manage-claude/api-and-data-retention> — régime *zero data retention* et engagement de non-entraînement sur les données retenues.
- <https://ai.google.dev/gemini-api/docs/pricing> — tarifs Gemini, remises batch/flex 50 %, cache à ~10 % du prix d'entrée.
- <https://mistral.ai/pricing/api> et <https://mistral.ai/pricing> — grille complète, OCR 4.1 à 4 $/1 000 pages, inférence régionale UE +10 %, cache −90 % ; mention d'une licence Mistral distincte pour certains déploiements commerciaux.
- <https://mistral.ai/news/mistral-compute> — offre d'infrastructure européenne (11 juin 2025) ; garanties de résidence non détaillées.
- <https://deepinfra.com/pricing> — modèles ouverts hébergés et **GPU dédiés à l'heure** (H100 80 Go à 2,20 $/h), borne basse du calcul de bascule.
- <https://www.runpod.io/pricing> — tarifs GPU horaires par type (H100 PCIe secure 2,89 $/h).
- <https://lambda.ai/service/gpu-cloud#pricing> — tarifs GPU horaires (H100 SXM 3,99–4,29 $/h), borne haute du calcul.
- <https://www.scaleway.com/en/pricing/gpu/> — tarifs GPU européens en euros (L4 à 0,79 €/h) ; H100 et HGX B300 en PAR-2, prix non relevés.
- <https://www.ovhcloud.com/fr/public-cloud/prices/> — gamme GPU OVHcloud et services AI Deploy / AI Endpoints ; tarifs non collectés.
- <https://openrouter.ai/models?order=top-weekly> et <https://openrouter.ai/docs/features/privacy-and-logging> — tarifs de passerelle ; filtrage des fournisseurs par politique de rétention et routage régional UE/US.
- <https://docs.vllm.ai/en/latest/features/structured_outputs.html> et <https://github.com/vllm-project/vllm/blob/main/README.md> — backends `xgrammar`/`guidance`, JSON Schema, regex, EBNF, structural tags, actif par défaut ; PagedAttention, continuous batching, prefix caching, parallélismes.
- <https://docs.sglang.io/advanced_features/structured_outputs.html> — `xgrammar` par défaut, `outlines` et `llguidance` en option ; mêmes formes de contrainte.
- <https://github.com/NVIDIA/TensorRT-LLM/blob/main/README.md> — v1.3.0rc25 ; in-flight batching, décodage guidé, déploiement via Triton et NVIDIA Dynamo.
- <https://github.com/ollama/ollama/blob/main/README.md> — positionnement local/poste de travail ; ni production ni sortie structurée documentées.
- <https://github.com/ggml-org/llama.cpp/blob/master/README.md> — `llama-server` OpenAI-compatible, grammaires GBNF, très large couverture matérielle.
- <https://docs.litellm.ai/docs/> — plus de 100 fournisseurs derrière un format unique ; SDK et proxy auto-hébergeable ; routage, fallbacks, budgets, garde-fous, suivi de coût.
- <https://owasp.org/www-project-top-10-for-large-language-model-applications/> — LLM01 Injection de prompt ; projet migré vers OWASP GenAI Security Project, version courante non confirmée (page servant une version archivée).
