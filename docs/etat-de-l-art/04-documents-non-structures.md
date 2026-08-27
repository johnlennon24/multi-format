# Axe 4 — Documents non structurés

> État de l'art arrêté au **27 août 2026**. Toutes les versions, licences et tarifs
> cités ont été revérifiés en ligne à cette date ; les liens figurent en fin de document.
> Lorsqu'un chiffre n'a pas pu être confirmé sur une source consultable, il est noté
> explicitement « non trouvé » plutôt qu'estimé.

## Périmètre et enjeu pour le projet

Le cadre agentique doit ingérer des sources de SI hétérogènes. Une partie de ces sources
n'est pas requêtable : PDF natifs, PDF scannés, courriels (EML/MSG) avec pièces jointes,
bureautique (DOCX/XLSX/PPTX), images de télécopies, archives papier numérisées. C'est
l'axe où l'agent LLM apporte réellement de la valeur, parce qu'il n'existe pas de schéma
source à mapper : il faut *reconstruire* une structure.

Trois exigences du projet cadrent l'analyse et éliminent d'emblée une partie du marché :

1. **Connecteurs enfichables** : le parsing doit être une brique remplaçable derrière une
   interface stable (`bytes + mime-type -> DocumentModel`), pas un SDK qui contamine
   l'architecture. Cela pousse vers les outils qui exposent un modèle de document
   sérialisable et documenté, et écarte les bibliothèques qui ne rendent que du markdown.
2. **Souveraineté non tranchée** : l'analyse doit fournir un chemin 100 % local *et* un
   chemin managé, avec le delta de coût et de qualité chiffré, pour que le client tranche
   sur des faits.
3. **GitOps** : le parsing doit être reproductible. Un pipeline dont la sortie change parce
   qu'un fournisseur a mis à jour son modèle derrière l'API est un problème de gouvernance,
   pas seulement de qualité. Ce critère pèse lourd contre les API SaaS à modèle opaque.

Le périmètre exclut la couche RAG/chunking (traitée ailleurs) et se concentre sur :
conversion document → représentation structurée, puis représentation → objet métier conforme
à un schéma.

## Pipeline classique contre VLM de bout en bout

C'est l'arbitrage central de l'axe, et il a nettement bougé en 2025-2026. Il faut d'abord
poser une distinction que le marketing brouille systématiquement : **« VLM » recouvre deux
choses très différentes**.

- **VLM généraliste** (GPT-5.2, Gemini 3.x, Claude, Qwen3-VL…) : on envoie l'image de page
  et on demande du markdown ou du JSON. Modèle non spécialisé, autorégressif, pas de
  contrainte visuelle forte.
- **VLM de document spécialisé, petit et à poids ouverts** (PaddleOCR-VL 1.6 ≈ 0,9 Md,
  MinerU 2.5 ≈ 1,2 Md, GLM-OCR ≈ 0,9 Md, granite-docling 258 M, olmOCR 2 = 7 Md) : entraîné
  uniquement sur la tâche de conversion de document, souvent avec des sorties ancrées
  (boîtes englobantes, balisage de blocs).

### Ce que disent les mesures

Sur **OmniDocBench v1.6** (1 651 pages, 10 types de documents), le classement consulté sur
le dépôt du benchmark donne :

| Système | Type | Score global | TEDS tableaux |
|---|---|---|---|
| PaddleOCR-VL-1.6 (0,9 Md) | VLM spécialisé | 96,34 | 94,76 |
| MinerU2.5-Pro (1,2 Md) | VLM spécialisé | 95,75 | 93,42 |
| GLM-OCR (0,9 Md) | VLM spécialisé | 95,22 | 92,83 |
| GPT-5.2 / Gemini-3 / InternVL3.5 | VLM généraliste | 83 – 93 | non détaillé |
| MinerU-Pipeline | pipeline classique | 86,47 | 81,88 |
| Marker | pipeline hybride | 78,44 | 65,77 |

Sur OmniDocBench v1.5, un relevé indépendant donne GLM-OCR 94,6, PaddleOCR-VL-1.5 94,5,
DeepSeek-OCR-2 91,1, dots.ocr 88,4, et Gemini-3-Flash 90,1.

**Trois conclusions se dégagent, et elles ne vont pas dans le sens attendu.**

**1. Le pipeline OCR + layout + règles a perdu sur la qualité brute.** L'écart
MinerU-pipeline (86,47) contre MinerU-VLM (95,30) est mesuré *sur le même produit, sur le
même benchmark, par la même équipe* : ce n'est pas un biais de comparaison. Le pipeline
classique reste néanmoins imbattable sur un cas précis — le PDF **né numérique** avec couche
texte propre — où l'extraction directe de la couche texte est exacte par construction, à coût
et latence quasi nuls, et déterministe.

**2. Le VLM généraliste n'est pas le gagnant non plus.** Des modèles de 0,9 à 1,2 milliard de
paramètres dépassent GPT-5.2 et Gemini 3 sur la conversion de document. La conclusion
opérationnelle est nette : *envoyer une page à un LLM frontière pour la transcrire est à la
fois le choix le plus cher et pas le plus précis*. Le VLM généraliste garde sa place en aval
(raisonnement sur le contenu déjà transcrit), pas en amont.

**3. Le vrai clivage n'est pas « pipeline vs VLM » mais « sortie ancrée vs sortie
générée ».** Une étude PP-OCRv6 (juin 2026) mesure un banc de hallucination : PP-OCRv6_medium
(34,5 M de paramètres, décodage CTC+NRTR) atteint 93,2 %, contre 85,0 % pour Kimi-K2.6,
80,6 % pour Qwen3-VL-235B et 72,6 % pour MiniMax-M3. L'explication donnée est structurelle :
un décodeur CTC est contraint par les features visuelles, un décodeur autorégressif peut
produire du texte plausible sans support visuel. Une seconde étude (OpenReview, 2026) montre
que sur entrées dégradées les VLM produisent du texte fluide et faux *sans signaler
d'incertitude*, parce que le post-entraînement récompense la réponse plutôt que l'abstention.

### Les trois autres axes de l'arbitrage

**Coût.** Le VLM spécialisé auto-hébergé est le moins cher à volume. Un relevé de mai 2026
sur DeepSeek-OCR donne 0,163 $/1 000 pages sur A100 cloud à 2,45 $/h, 0,051 $/1 000 pages sur
T4 cloud, et de l'ordre de 0,010 $/1 000 pages sur RTX 4090 possédée (amortissement matériel
exclu). À comparer à 4 $/1 000 pages pour Mistral OCR 4 en API, 10 $/1 000 pour Azure Layout
ou Textract Tables (15 $), 30 $/1 000 pour Unstructured en pay-as-you-go. Le facteur est de
**1 à 1 000** entre l'auto-hébergement amorti et le SaaS haut de gamme. Attention : ces
mesures de débit sont à prendre avec précaution, la même source annonçant à la fois
« 200 000 pages/jour sur A100 » (≈ 8 300 pages/h) et « 15 000 pages/h sur A100 » — un facteur
2 d'incohérence interne.

**Latence.** Le pipeline pur texte reste hors catégorie : Marker 2 en mode « disable OCR »
annonce 23,7 pages/s sur CPU. Un VLM de 1 Md sur GPU traite typiquement quelques pages par
seconde ; un VLM frontière en API, quelques secondes par page. Pour un runtime agentique qui
doit répondre dans une boucle, cette différence dicte l'architecture : on ne peut pas se
permettre le VLM sur chaque page si 80 % du corpus est né numérique.

**Reproductibilité.** C'est l'argument décisif pour le GitOps, et il tranche contre les API
SaaS. Un modèle à poids ouverts épinglé par hash dans le dépôt donne la même sortie dans six
mois. Une API managée ne le garantit pas. Le décodage autorégressif ajoute une variabilité
propre, atténuable en fixant la température à 0 et la seed, mais pas totalement supprimée en
inférence par lots sur GPU.

### Position retenue pour le projet

**Routage par classe de page, pas choix unique.** Un pré-classifieur bon marché décide :

- page avec couche texte fiable → extraction texte directe (pdfplumber/pypdfium2), coût ≈ 0 ;
- page sans couche texte, ou couche texte incohérente avec le rendu, ou détectée comme
  contenant un tableau → VLM de document spécialisé auto-hébergé ;
- page en échec ou score de confiance sous seuil → escalade vers un second modèle, puis
  file de relecture humaine.

Ce routage est le seul moyen d'avoir simultanément le coût du pipeline et la qualité du VLM.
Il est aussi ce qui rend le connecteur enfichable : la politique de routage est une
configuration versionnée, pas du code.

## Panorama des solutions

### Socle bas niveau — PyMuPDF, pdfplumber, Apache Tika

Ce sont les briques déterministes, sans modèle, du bas de la pile. **pdfplumber** (MIT) est
la référence pour l'extraction de tableaux à partir de la géométrie des traits et des mots ;
il est lent (≈ 18 pages/s sur du texte simple) mais sa licence permissive le rend utilisable
partout. **PyMuPDF** est 8 à 12× plus rapide sur le texte brut (≈ 180 pages/s) mais repose sur
MuPDF en **AGPL-3.0** : toute distribution impose l'AGPL sur le produit, ou l'achat d'une
licence commerciale Artifex. Pour un cadre réutilisable destiné à être livré à des clients,
c'est un risque juridique qu'il faut expliciter — `pypdfium2` (BSD/Apache) est l'alternative
usuelle. **Apache Tika** (Apache 2.0) est à part : ce n'est pas un bon parseur PDF, mais c'est
le meilleur détecteur de type MIME et le plus large extracteur de texte et de métadonnées sur
un millier de formats bureautiques, archives et courriels. La version courante est **4.0.0**
(Java 17+, distribution en zip), la branche 3.x étant maintenue en 3.3.2 avec une fin de
support annoncée en juin 2026. Le coût d'exploitation est un service JVM à côté d'un cadre
Python — via `tika-server` en HTTP, pas via le binding Python qui télécharge le jar à chaud.
Rôle dans le cadre : triage d'entrée et chemin rapide, jamais la qualité finale.

### Docling (projet LF AI & Data, origine IBM Research)

Version **2.123.0 au 26 août 2026**, licence **MIT**, hébergé par la LF AI & Data Foundation.
C'est le projet le mieux gouverné de la liste : licence permissive sans réserve, fondation
neutre, et une offre managée (« Docling for IBM watsonx », GA en juin 2026 sur IBM Marketplace
et AWS Marketplace) qui tourne explicitement sur *les mêmes dépôts* que la version
communautaire — pas de fork, pas de réimplémentation propriétaire. C'est le seul acteur du
panorama qui offre une porte de sortie managée sans changement de technologie, ce qui est
exactement ce qu'il faut quand la souveraineté n'est pas tranchée.

Couverture de formats la plus large : PDF, DOCX, PPTX, XLSX, HTML, EPUB, Pages, EML/MSG,
images, LaTeX, et même audio (ASR). Sortie : un `DoclingDocument` avec `prov` (numéro de page
et bbox) sur les éléments — une **traçabilité déterministe sans appel LLM supplémentaire**,
ce qui est un atout majeur pour la validation. Docling expose aussi une API d'**extraction
structurée** acceptant un modèle Pydantic ou un dictionnaire de schéma, et un projet annexe
`docling-graph` qui construit un graphe de connaissances validé à partir de ces modèles.

Points faibles : sur la qualité brute de parsing, Docling est derrière. Un comparatif tiers
restreint aux PDF nés numériques le place à 64,0 contre 83,5 pour Marker 2 et 83,3 pour le
pipeline MinerU (suite d'évaluation non standardisée, à prendre comme un ordre de grandeur).
Le thème de 2026 côté Docling n'a pas été la précision mais le durcissement : parseur PDF
thread-safe et à mémoire bornée, isolation des erreurs page par page, endpoint de traitement
par lots. Le modèle maison **granite-docling-258M** (Apache 2.0, annoncé janvier 2026) est
remarquablement petit mais ne joue pas dans la catégorie de PaddleOCR-VL.

### MinerU (OpenDataLab)

**v3.4 au 18 juin 2026** (publication PyPI du 14 août 2026). C'est le meilleur rapport
qualité/effort en open source aujourd'hui, avec trois backends sur le même produit :
`pipeline` (CPU, 86,47 sur OmniDocBench), `vlm` (95,30) et `hybrid` (95,26 à 95,39 selon
l'« effort »). Le paramètre `effort` introduit en 3.3 est bien pensé : passer de « high » à
« medium » coûte 0,13 point de score et rend 35 % à 220 % de vitesse selon le matériel.
L'OCR est passé à PP-OCRv6 en 3.4 (+11 % de précision, 2× plus rapide). Couverture :
PDF, images, DOCX/PPTX/XLSX natifs, 109 langues. Prérequis : 4 Go de VRAM minimum, 16 Go de
RAM, Python 3.10-3.13 ; le backend pipeline tourne en CPU pur.

**Le point d'attention est la licence.** MinerU est passé d'AGPLv3 à une « MinerU Open Source
License » en v3.1.0 (18 avril 2026) : basée sur Apache 2.0 **avec conditions additionnelles**.
C'est une amélioration par rapport à l'AGPL, mais ce n'est pas une licence OSI standard : elle
doit être lue intégralement et validée par le juridique avant tout engagement, en particulier
pour un cadre destiné à être redistribué. Ce type de changement — deux fois en dix-huit mois —
est aussi un signal de risque de gouvernance à documenter.

### Marker 2 et Surya (Datalab)

**Marker 2.0.0 le 20 juillet 2026**, réécriture complète autour de Surya OCR 2, d'un modèle de
mise en page de 20 M de paramètres et d'un `pdftext` 3× plus rapide. Trois modes assumés :
« balanced » (VLM Surya + OCR pleine page si la couche texte est mauvaise) à 76,0 % sur
olmOCR-bench ; « fast » (layout rf-detr/onnx, VLM minimal) à 66,6 % ; « disable OCR » (couche
texte pure, CPU seul) à 43,6 % pour **23,7 pages/s**. Ce dernier mode est intéressant comme
chemin rapide du routage décrit plus haut.

**Le blocage est la licence des poids.** Le code Surya est en Apache 2.0, mais les poids sont
sous une *AI Pubs Open RAIL-M* modifiée : usage libre en recherche, usage personnel et pour
les startups sous un seuil de chiffre d'affaires / financement — les sources consultées citent
5 M$ pour Surya et 2 M$ pour les outils Datalab, ce qui est déjà une incohérence à faire
lever. Au-delà, il faut acheter une licence commerciale. Pour un cadre réutilisable vendu à
des clients de taille indéterminée, **c'est disqualifiant en l'état** : on ne peut pas
embarquer un composant dont la conformité dépend du chiffre d'affaires de l'utilisateur final.
Datalab a par ailleurs poussé une offre API (disponibilité sur Replicate depuis avril 2026),
ce qui confirme la trajectoire commerciale.

### VLM de document à poids ouverts spécialisés

C'est la catégorie qui a le plus bougé et celle qui porte la recommandation. Quatre modèles à
retenir, tous auto-hébergeables :

- **PaddleOCR-VL-1.6** (0,9 Md, **Apache 2.0**, 28 mai 2026) : 96,34 sur OmniDocBench v1.6,
  TEDS tableaux 94,76 — état de l'art. Architecture identique à la 1.5, donc migration sans
  coût. Builds quantifiés utilisables via Ollama et LM Studio. Apache 2.0 sans réserve.
- **olmOCR 2** (`olmOCR-2-7B-1025`, **Apache 2.0**, Allen AI) : fine-tune de Qwen2.5-VL-7B
  entraîné par RLVR, 82,4 sur olmOCR-Bench. Plus gros donc plus cher à servir, mais
  l'écosystème (toolkit de linéarisation, poids FP8) est pensé pour le traitement massif par
  lots. Bon sur les mises en page dégradées et l'écriture manuscrite.
- **DeepSeek-OCR 2** (fin janvier 2026) : 91,1 sur OmniDocBench v1.5, positionné sur la
  compression de contexte optique et le débit. C'est le champion du coût par page en batch.
- **GLM-OCR** (0,9 Md) : 95,22 / TEDS 92,83, dans le peloton de tête ; **dots.ocr** à 88,4 est
  désormais derrière.

L'intérêt structurel de cette famille : Apache 2.0, taille sub-milliard, donc une seule GPU
modeste suffit, aucun appel externe, poids épinglables par hash dans le dépôt. Le risque : ce
sont des modèles autorégressifs, donc sujets à l'hallucination sur entrée dégradée (cf.
supra) ; ils exigent une couche de validation, pas une confiance aveugle.

### Mistral OCR 4 / Document AI

**Sorti le 23 juin 2026.** C'est le seul acteur SaaS du panorama qui coche à la fois la
souveraineté européenne et le déploiement sur site : OCR 4 est livré en **conteneur unique
auto-hébergeable** pour les clients entreprise, donc extraction dans le périmètre de sécurité
du client, aucun document chez Mistral. Tarif API : **4 $/1 000 pages**, **2 $/1 000 pages**
en Batch ; « Document AI » (extraction structurée avec schéma personnalisé) à **5 $/1 000
pages**. 170 langues. 85,20 sur olmOCR-Bench, soit au-dessus d'olmOCR 2 (82,4).

Ce qui le distingue fonctionnellement : la sortie porte **des boîtes englobantes, une
classification de blocs (titres, tableaux, équations, signatures) et des scores de confiance
en ligne, par page et par mot**. C'est la brique de validation la plus complète du panorama,
et c'est précisément ce qui manque à Docling et aux VLM à poids ouverts. Réserves : le tarif
du conteneur auto-hébergé n'est pas public (**non trouvé**), le modèle reste propriétaire donc
non épinglable au sens GitOps, et les chiffres de préférence humaine (« 72 % de taux de
victoire ») sont des mesures internes du fournisseur.

### Unstructured et LlamaParse

Deux produits différents, même trajectoire : une bibliothèque open source qui sert de tête de
pont vers une plateforme payante.

**Unstructured** : bibliothèque Apache 2.0, ~15,3 k étoiles, activité soutenue (1 919 commits,
186 issues ouvertes). Sa valeur est la couverture de formats et la normalisation en
« éléments » typés, pas la qualité de parsing PDF. **Piège juridique documenté** : certaines
dépendances (notamment `ultralytics`) sont en **AGPLv3** — la licence Apache 2.0 du dépôt ne
protège donc pas l'installation par défaut. Tarif plateforme : 15 000 pages gratuites par
mois, puis **0,03 $/page (30 $/1 000 pages)** — le plus cher du panorama à volume. Une source
mentionne aussi « à partir de 1 $/1 000 pages » pour l'API serverless : les deux chiffres sont
contradictoires et devraient être confirmés en devis.

**LlamaParse** (LlamaCloud) : modèle à crédits, **1 000 crédits = 1,25 $**. 1 crédit/page sans
IA (**0,00125 $/page**), 3 crédits en mode « cost-effective » (**0,00375 $/page**), jusqu'à
90 crédits/page en mode agentique avec un modèle de pointe (**0,1125 $/page**) ; une source
annonce une fourchette de 0,00125 à 0,05625 $/page, incohérente avec le plafond de
90 crédits — à vérifier en conditions réelles. 10 000 crédits gratuits par mois. Techniquement
compétent sur les mises en page complexes, mais **100 % SaaS, aucun mode local** : ni la
souveraineté ni la reproductibilité GitOps ne sont satisfaites.

### Services managés hyperscalers et plateformes agentiques

**AWS Textract** (tarifs officiels, US West Oregon) : `DetectDocumentText` 1,50 $/1 000 pages
(0,60 $ au-delà d'1 M) ; `AnalyzeDocument` **Tables 15 $**/1 000 (10 $ au-delà d'1 M) ;
**Forms 50 $**/1 000 (40 $) ; **Queries 15 $**/1 000 ; **Layout gratuit** s'il est utilisé avec
Tables ; `AnalyzeExpense` 10 $/1 000 (8 $).

**Google Document AI** : OCR Entreprise 1,50 $/1 000 pages (0,60 $ au-delà de 5 M/mois) ;
Form Parser et Custom Extractor **30 $/1 000** (20 $ au-delà d'1 M/mois) ; parsers spécialisés
facture/dépense à 0,10 $ par tranche de 10 pages, soit 10 $/1 000. *La page tarifaire
officielle n'a pas renvoyé les montants lors de la consultation ; ces chiffres viennent de
synthèses tierces de 2026 et doivent être reconfirmés.*

**Azure AI Document Intelligence** : Read 1,50 $/1 000 ; Layout 10 $/1 000 ; modèles
préconstruits 10 $/1 000 ; extraction personnalisée 30 $/1 000 (20 $ au-delà d'1 M/mois) ;
classifieur 3 $/1 000 ; Query Fields 10 $/1 000. *Même réserve : la page officielle
n'affichait que des espaces réservés ; source tierce datée de juillet 2026.* Azure est en
revanche le seul à documenter des **scores de confiance et un ancrage (grounding) sur tous les
types de champs** dans ses API GA 2025-11-01 et préversion 2026-06-01.

**Plateformes agentiques** (Reducto, LandingAI ADE, Extend, Box Extract) : catégorie apparue
en 2025-2026, qui applique une boucle multi-passes de relecture par un agent au-dessus de
l'OCR, avec ancrage visuel systématique. Reducto Deep Extract revendique 99,6 % de précision
et de rappel sur LongExtractBench (225 documents de ~358 pages en moyenne) — chiffre publié
par le fournisseur, à traiter comme tel. Ces plateformes sont l'état de l'art fonctionnel,
mais aucune n'est auto-hébergeable et les tarifs sont sur devis (**non trouvé**).

## Grille comparative

| Solution | Licence | 100 % local | Qualité tableaux | Sortie structurée | Coût indicatif par page | Maturité | Adapté au cadre ? |
|---|---|---|---|---|---|---|---|
| **Docling 2.123** | MIT (LF AI & Data) | Oui, complet | Moyenne (TEDS non publié sur v1.6 ; 64,0 global sur un comparatif tiers PDF natifs) | Native : schéma Pydantic ou dict, + `docling-graph` | 0 $ de licence ; coût CPU/GPU seul | Élevée — durcissement 2026, offre managée GA | **Oui — socle de la couche document** |
| **MinerU 3.4** | « MinerU Open Source License » (base Apache 2.0 + conditions) | Oui (backend pipeline en CPU pur) | **Bonne** — TEDS 93,42 (VLM), 81,88 (pipeline) | Markdown/JSON ; pas de schéma Pydantic natif | 0 $ de licence ; ~4 Go VRAM | Élevée, mais 2 changements de licence en 18 mois | **Oui, sous réserve juridique** |
| **PaddleOCR-VL 1.6** | Apache 2.0 | Oui | **Meilleure mesurée** — TEDS 94,76 | Markdown/JSON ancré ; schéma via décodage contraint en aval | 0 $ de licence ; ~0,01–0,16 $/1 000 p. selon GPU | Élevée sur la tâche, écosystème plus étroit | **Oui — moteur VLM par défaut** |
| **olmOCR 2 (7B)** | Apache 2.0 | Oui | Non publié sur OmniDocBench | Texte linéarisé ; schéma en aval | 0 $ de licence ; 7 Md = plus coûteux à servir | Moyenne-élevée (Allen AI) | Second moteur, escalade |
| **Marker 2 / Surya** | Code Apache 2.0 ; **poids RAIL-M restreints** | Oui techniquement | Faible — TEDS 65,77 | Markdown + JSON | Licence commerciale requise au-delà du seuil (**non trouvé**) | Élevée | **Non — licence des poids** |
| **Mistral OCR 4** | Propriétaire ; conteneur auto-hébergeable (entreprise) | Oui, si conteneur sur site | Non publié en TEDS ; 85,20 olmOCR-Bench | Oui — Document AI, schéma personnalisé | 4 $/1 000 p. API ; 2 $ en batch ; 5 $ Document AI ; sur site **non trouvé** | Élevée | Oui — option souveraine managée |
| **Unstructured** | Apache 2.0 mais **dépendances AGPLv3** | Oui (lib) | Faible sur PDF complexes | Éléments typés, pas de schéma métier | 30 $/1 000 p. (plateforme) | Élevée en couverture, moyenne en qualité | Partiel — pré-traitement seulement |
| **LlamaParse** | Propriétaire SaaS | **Non** | Bonne réputée, non chiffrée ici | Oui (modes agentiques) | 1,25 $ à 112,50 $/1 000 p. selon mode | Élevée | **Non — pas de mode local** |
| **Azure DI / Textract / Document AI** | Propriétaire SaaS | **Non** | Textract Tables et Azure Layout : bons, TEDS non publié | Oui (Query Fields, Custom Extractor) | 1,50 $ (OCR) à 50 $ (Forms)/1 000 p. | Très élevée | Non par défaut ; repli si le client impose son cloud |
| **PyMuPDF / pdfplumber / Tika 4.0** | AGPL-3.0 / MIT / Apache 2.0 | Oui | pdfplumber : correct sur tableaux à traits | Non | ≈ 0 | Très élevée | Oui — chemin rapide et triage |

## Fiabilité et validation des extractions

C'est le sujet le plus mal traité par les outils et celui qui déterminera si le cadre est
utilisable en production. Une extraction fausse mais bien formée est plus dangereuse qu'un
échec, parce qu'elle passe tous les contrôles de type.

**1. Garantir la forme : le décodage contraint.** Le problème « obtenir un JSON conforme à un
schéma » est aujourd'hui résolu au niveau du décodeur, pas du prompt. **XGrammar** est depuis
mars 2026 le backend de génération structurée par défaut de **vLLM, SGLang et TensorRT-LLM**,
avec moins de 40 µs par token de surcoût. `llguidance` (Microsoft, parseur Earley en Rust,
~50 µs/token) est l'alternative. Point technique à ne pas rater : les schémas **récursifs**
exigent un moteur à grammaire hors-contexte (XGrammar, llguidance) ; les moteurs à automate
fini comme Outlines les rejettent ou aplatissent la récursion à une profondeur fixe. En
pratique : définir les schémas en Pydantic, exporter en JSON Schema, servir en vLLM avec
XGrammar. **La conformité au schéma devient alors une garantie, pas une probabilité.**

**2. Ne pas confondre conformité et véracité.** C'est l'erreur la plus fréquente. Le décodage
contraint garantit que le champ `montant_ttc` est un nombre ; il ne garantit pas que c'est le
bon nombre. Le travail de 2026 sur les petits modèles (« When Correct Isn't Usable ») montre
précisément cet écart entre sortie correcte au sens du schéma et sortie exploitable. Il faut
donc une seconde couche.

**3. Ancrer chaque champ sur une zone source.** C'est la technique la plus rentable. Trois
niveaux disponibles selon l'outil :
- **Docling** : `prov` (page + bbox) sur chaque élément, produit de façon déterministe, sans
  appel LLM supplémentaire ;
- **Mistral OCR 4** : boîtes englobantes, classification de blocs et scores de confiance par
  page et par mot ;
- **Azure Document Intelligence** : confiance et grounding sur tous les types de champs
  (API GA 2025-11-01 et préversion 2026-06-01) ; les plateformes agentiques (Reducto,
  LandingAI ADE) en font leur argument principal.

Règle à imposer dans le cadre : **tout champ extrait doit porter (page, bbox, texte source)**.
Un champ sans provenance est rejeté. Cela rend la vérification humaine possible en quelques
secondes au lieu de quelques minutes, et permet un contrôle automatique de cohérence : la
valeur extraite doit être retrouvable dans le texte de la zone citée (comparaison exacte ou
distance d'édition bornée pour les nombres et dates).

**4. Confiance calibrée.** Distinguer trois sources de confiance, qui ne mesurent pas la même
chose : la confiance OCR (par mot, fiable et calibrée, disponible chez Mistral OCR 4 et Azure),
la log-probabilité du décodeur (mal calibrée sous décodage contraint, la contrainte écrase les
alternatives) et la confiance auto-déclarée par le modèle. Un travail de 2026 rapporte que la
confiance auto-déclarée surpasse d'environ 50 % en métriques de calibration les approches par
log-probabilité — contre-intuitif, mais cohérent avec le fait que la contrainte de grammaire
détruit l'information probabiliste. En pratique : utiliser la confiance OCR pour router les
pages, et l'auto-consistance pour router les champs.

**5. Auto-consistance et multi-passes.** La technique la plus robuste et la plus chère :
exécuter l'extraction N fois (température non nulle, ou deux moteurs différents) et ne retenir
que les champs concordants ; les champs divergents partent en relecture. C'est ce que font les
plateformes agentiques sous le nom d'« agentic OCR » (Reducto : segmentation CV, interprétation
par région, puis couche de relecture multi-passes ; +7,5 % de TEDS corrigé rapporté sur
l'approche agentique de parsing de tableaux). Coût : ×2 à ×3. À réserver aux champs critiques,
sélectionnés par schéma, pas au document entier.

**6. Contrôles métier déterministes.** Les moins chers et les plus efficaces, à écrire une fois
par schéma : sommes de contrôle (somme des lignes = total), cohérence de dates, formats
réglementaires (SIRET, IBAN, TVA intracommunautaire), plages de valeurs, unicité. Un échec de
contrôle métier est un signal beaucoup plus fiable qu'un score de confiance de modèle.

**7. Boucle de relecture humaine.** Non négociable en production. Le budget se pilote par le
taux d'escalade, qui doit être une métrique de premier ordre du cadre, mesurée en continu et
versionnée avec les jeux d'évaluation.

**8. Jeu de régression versionné.** Corollaire GitOps : un corpus de pages annotées, dans Git
(ou en LFS/DVC), rejoué en CI à chaque changement de modèle, de version ou de prompt, avec des
seuils bloquants. Sans cela, aucune mise à jour de modèle n'est sûre. C'est ce qui rend le
choix « poids ouverts épinglés » supérieur au SaaS : on contrôle *quand* le modèle change.

## Ce que ça implique pour un runtime agentique

**L'agent ne doit pas faire l'OCR.** La transcription est une tâche déterministe et mesurable ;
la confier à un LLM généraliste coûte cher, dégrade la qualité (cf. section 2) et détruit la
reproductibilité. Le rôle de l'agent est en amont (routage, choix de schéma, classification du
document) et en aval (résolution d'ambiguïté, réconciliation entre sources, décision
d'escalade).

**Le parsing est un outil, pas une étape de la boucle.** Concrètement : une interface
`parse(bytes, mime) -> DocumentModel` et `extract(DocumentModel, schema) -> (objet, provenance,
confiance)`. Le `DocumentModel` de Docling est un candidat sérieux comme représentation
pivot — MIT, sérialisable, avec provenance native — quel que soit le moteur qui l'a produit.
Cela permet de remplacer PaddleOCR-VL par Mistral OCR 4 sans toucher au reste du cadre.

**Idempotence et cache.** Clé de cache = hash du contenu + identifiant de version du moteur +
hash du schéma. Rejouer un traitement ne doit pas repayer le parsing. C'est aussi ce qui rend
le pipeline rejouable en CI sur le corpus de régression sans coût.

**Traitement par lots, pas page par page dans la boucle.** Les VLM spécialisés donnent leur
débit en batch sur GPU. Un runtime qui appelle le parseur une page à la fois depuis une boucle
d'agent perd un ordre de grandeur. Séparer un service de parsing (file d'attente, batch GPU) du
runtime agentique.

**Budget explicite par document.** Chaque exécution doit porter un plafond en pages traitées,
en appels de modèle et en euros, avec dégradation contrôlée (basculer en mode rapide, ou
escalader en relecture) plutôt qu'un échec.

**Isolation des échecs.** Une page corrompue ne doit pas faire tomber un lot de 10 000. Docling
a explicitement travaillé ce point en 2026 (isolation d'erreur par page, parseur à mémoire
bornée) — c'est un critère de production souvent négligé au moment du choix.

## Recommandation

### Option 1 (recommandée) — Socle 100 % local, routé, avec Docling comme modèle pivot

- **Triage** : Apache Tika 4.0 (détection MIME, formats bureautiques et courriels) +
  pdfplumber/pypdfium2 pour le chemin rapide sur PDF nés numériques.
- **Modèle pivot** : Docling (MIT) comme représentation de document et couche de provenance.
- **Moteur VLM** : PaddleOCR-VL-1.6 (Apache 2.0, 0,9 Md, TEDS 94,76) auto-hébergé, servi en
  vLLM ; olmOCR 2 (Apache 2.0) en second moteur pour l'escalade et l'auto-consistance.
- **Extraction structurée** : Pydantic → JSON Schema → décodage contraint XGrammar sous vLLM.
- **Validation** : provenance obligatoire, contrôles métier, auto-consistance sur champs
  critiques, corpus de régression en CI.

*Pourquoi* : c'est la seule combinaison qui donne simultanément la meilleure qualité mesurée
(96,34 sur OmniDocBench v1.6), des licences permissives sans réserve (MIT + Apache 2.0), zéro
appel externe, et des poids épinglables par hash — donc une reproductibilité compatible GitOps.
Coût marginal proche de zéro à volume (~0,01 à 0,16 $/1 000 pages selon le GPU). *Coût* :
il faut exploiter au moins une GPU et assumer l'ingénierie du routage et de la validation.

### Option 2 — Socle local + Mistral OCR 4 comme moteur de qualité

Même architecture, mais Mistral OCR 4 remplace ou double le VLM à poids ouverts, en API
(4 $/1 000 pages, 2 $ en batch) au démarrage, puis en conteneur auto-hébergé si le client
tranche pour la souveraineté.

*Pourquoi* : c'est le seul produit qui livre nativement **boîtes englobantes + classification de
blocs + scores de confiance par mot**, ce qui économise une part significative de la couche de
validation à construire soi-même ; et c'est le seul chemin managé → sur site sans changer de
technologie, avec un fournisseur européen. *Pourquoi ce n'est pas l'option 1* : modèle
propriétaire donc non épinglable, tarif du conteneur sur site non public, et coût API
20 à 400× supérieur à l'auto-hébergement de poids ouverts à volume.

### Option 3 (repli contraint) — Service managé du cloud imposé par le client

Si le client impose Azure ou AWS pour des raisons de contrat-cadre : Azure Document
Intelligence (Layout 10 $/1 000 pages, extraction personnalisée 30 $/1 000) pour la maturité de
ses scores de confiance et de son grounding sur tous les types de champs ; Textract sinon
(Tables 15 $/1 000, Layout gratuit avec Tables). Conserver impérativement l'interface
`parse()`/`extract()` pour que le moteur reste substituable.

*Pourquoi c'est un repli et pas un choix* : coût 100 à 1 000× supérieur à volume, aucune
maîtrise du versionnement du modèle (donc GitOps partiel), et sortie du périmètre de
souveraineté.

### Ce que nous écartons explicitement

- **Marker 2 / Surya** — malgré d'excellentes performances et le meilleur mode CPU du
  panorama (23,7 pages/s), les **poids sont sous licence RAIL-M restreinte** conditionnée au
  chiffre d'affaires de l'utilisateur. Un cadre réutilisable ne peut pas embarquer un composant
  dont la conformité dépend du client final.
- **LlamaParse** — aucun mode local, donc incompatible avec l'hypothèse de souveraineté encore
  ouverte, et grille tarifaire à crédits dont les chiffres publiés sont contradictoires.
- **PyMuPDF en dépendance de distribution** — AGPL-3.0. Utilisable en interne, pas dans un
  produit livré ; `pypdfium2` le remplace pour l'extraction texte.
- **Unstructured comme moteur de parsing principal** — qualité PDF insuffisante, 30 $/1 000
  pages, et dépendances AGPLv3 qui neutralisent l'Apache 2.0 affichée. Reste acceptable en
  pré-traitement de formats exotiques si Tika ne suffit pas.
- **VLM généraliste (GPT-5.2, Gemini 3, Claude) en transcription de page** — plus cher et moins
  précis que des modèles spécialisés de 1 Md (83–93 contre 95–96 sur OmniDocBench v1.6). Sa
  place est le raisonnement en aval, pas l'OCR.
- **Plateformes agentiques SaaS (Reducto, LandingAI ADE, Extend)** — fonctionnellement en
  avance, mais aucune n'est auto-hébergeable et les tarifs sont sur devis. À reconsidérer si le
  client abandonne l'exigence de souveraineté et privilégie le délai de mise en œuvre.

### Points de décision restant ouverts

1. **Souveraineté** : trancher avant de figer le moteur. Option 1 si local strict, Option 2 si
   « européen mais managé acceptable », Option 3 si cloud client imposé.
2. **Budget GPU** : l'Option 1 suppose au minimum une GPU dédiée au service de parsing.
3. **Licence MinerU** : faire lire la « MinerU Open Source License » par le juridique si l'on
   veut bénéficier de son backend hybride (95,39) et de son support natif DOCX/PPTX/XLSX.
4. **Tarifs à confirmer en devis** : conteneur Mistral OCR 4 sur site, licence commerciale
   Datalab, tarifs Google/Azure (pages officielles non exploitables lors de la consultation).
5. **Corpus de référence** : aucune décision de moteur ne devrait être figée sans un jeu de
   50 à 200 pages représentatives du SI cible, annoté et rejoué. Les scores OmniDocBench sont
   un filtre de présélection, pas une preuve sur les documents du client.

## Sources

- https://pypi.org/project/docling/ — page PyPI officielle : Docling 2.123.0 du 26/08/2026, licence MIT, hébergement LF AI & Data, liste des formats et extras.
- https://docling.ai/blog/20260615_00_docling_for_ibm_watsonx/ — annonce de l'offre managée « Docling for IBM watsonx » (GA, IBM et AWS Marketplace), et détail du durcissement 2026 (parseur thread-safe, isolation d'erreurs, endpoint batch).
- https://docling-project.github.io/docling/examples/extraction/ — documentation de l'API d'extraction structurée de Docling (schéma dict ou Pydantic).
- https://github.com/docling-project/docling-graph — projet annexe : documents → objets Pydantic validés → graphe de connaissances.
- https://www.ibm.com/new/announcements/granite-docling-end-to-end-document-conversion — annonce granite-docling-258M, licence Apache 2.0, filiation SmolDocling.
- https://huggingface.co/ibm-granite/granite-docling-258M — carte du modèle granite-docling-258M.
- https://www.linuxfoundation.org/press/lf-ai-data-foundation-launches-doclang-specification-working-group-to-advance-an-open-standard-for-ai-native-documents — gouvernance LF AI & Data et groupe de travail DocLang (juin 2026, IBM/NVIDIA/Red Hat/ABBYY).
- https://github.com/opendatalab/MinerU — dépôt MinerU : v3.4 du 18/06/2026, scores par backend (pipeline 86,47 / hybrid 95,26-95,39 / VLM 95,30), prérequis matériels, changement de licence en 3.1.0.
- https://opendatalab.github.io/MinerU/reference/changelog/ — journal des versions MinerU (PP-OCRv6 en 3.4, paramètre `effort` en 3.3, DOCX natif en 3.0).
- https://pypi.org/project/mineru/ — publication PyPI de MinerU (14/08/2026).
- https://github.com/opendatalab/OmniDocBench — benchmark de référence : v1.6 (1 651 pages), classement PaddleOCR-VL-1.6 / MinerU2.5-Pro / GLM-OCR, TEDS tableaux, scores pipeline (MinerU-Pipeline, Marker), formule du score global.
- https://github.com/datalab-to/marker/releases/tag/v2.0.0 — release Marker 2.0.0 du 20/07/2026, trois modes et scores olmOCR-bench (76,0 / 66,6 / 43,6 à 23,7 pages/s).
- https://github.com/datalab-to/surya/blob/master/MODEL_LICENSE — licence des poids Surya (AI Pubs Open RAIL-M modifiée, seuils de chiffre d'affaires).
- https://huggingface.co/PaddlePaddle/PaddleOCR-VL-1.6 — PaddleOCR-VL-1.6, 0,9 Md, Apache 2.0, publié le 28/05/2026, 96,3 sur OmniDocBench v1.6.
- https://huggingface.co/allenai/olmOCR-2-7B-1025 — olmOCR 2, Apache 2.0, fine-tune Qwen2.5-VL-7B, 82,4 sur olmOCR-Bench.
- https://allenai.org/blog/olmocr-2 — méthode d'entraînement RLVR et positionnement d'olmOCR 2.
- https://mistral.ai/news/ocr-4/ — Mistral OCR 4 (23/06/2026) : 4 $/1 000 pages, 2 $ en batch, Document AI à 5 $/1 000, conteneur auto-hébergeable, bboxes, classification de blocs, confiance par mot, 170 langues, 85,20 olmOCR-Bench.
- https://www.spheron.network/blog/best-open-source-ocr-vlm-self-host-gpu-cloud-2026/ — comparatif des VLM OCR auto-hébergeables (PaddleOCR-VL, DeepSeek-OCR 2, dots.ocr, GOT-OCR) et scores OmniDocBench v1.5.
- https://www.spheron.network/blog/deploy-deepseek-ocr-gpu-cloud/ — chiffres de débit et de coût par 1 000 pages pour DeepSeek-OCR auto-hébergé (A100, T4, RTX 4090/3090), relevés de mai 2026.
- https://aws.amazon.com/textract/pricing/ — tarifs officiels Textract (DetectDocumentText, Tables, Forms, Queries, Layout, AnalyzeExpense) avec paliers au-delà d'1 M de pages.
- https://cloud.google.com/document-ai/pricing — page tarifaire officielle Google Document AI (contenu non renvoyé lors de la consultation ; référence à reconfirmer).
- https://docuocr.com/blog/azure-document-intelligence-pricing — synthèse tierce des tarifs Azure Document Intelligence 2026 (Read, Layout, préconstruits, extraction personnalisée, classifieur, Query Fields) ; la page officielle Azure n'affichait pas les montants.
- https://learn.microsoft.com/en-us/azure/ai-services/content-understanding/document/analyzer-improvement — scores de confiance et grounding sur tous les types de champs dans les API Azure GA 2025-11-01 et préversion 2026-06-01.
- https://github.com/Unstructured-IO/unstructured — dépôt Unstructured : Apache 2.0, activité 2026, mention « 15 000 pages gratuites/mois puis 3 cents la page ».
- https://github.com/Unstructured-IO/unstructured/issues/3894 — signalement des dépendances AGPLv3 (ultralytics) sous une licence de dépôt Apache 2.0.
- https://unstructured.io/pricing — grille tarifaire Unstructured.
- https://developers.llamaindex.ai/python/cloud/llamaparse/faq/ — modèle de crédits LlamaParse (1 000 crédits = 1,25 $, coût par mode de parsing).
- https://www.eesel.ai/blog/llamaindex-pricing — synthèse tierce de la tarification LlamaIndex/LlamaParse 2026 (fourchette par page, palier gratuit).
- https://tika.apache.org/download.html — Apache Tika : 4.0.0 courante (Java 17+), 3.3.2 maintenue, distribution zip.
- https://cwiki.apache.org/confluence/display/TIKA/Tika+Roadmap+--+2.x,+3.x+and+Beyond — feuille de route Tika et fin de support de la branche 3.x.
- https://pdfmux.com/blog/pymupdf-vs-pdfplumber/ — comparatif 2026 vitesse/licence PyMuPDF (AGPL-3.0, ~180 p/s) contre pdfplumber (MIT, ~18 p/s), et usage mixte en production.
- https://arxiv.org/html/2606.13108v1 — PP-OCRv6 : banc de hallucination (93,2 % pour PP-OCRv6_medium contre 85,0 / 80,6 / 72,6 pour les VLM généralistes) et explication architecturale CTC contre autorégressif.
- https://openreview.net/forum?id=zyCjizqOxB — « Teaching VLMs to Admit Uncertainty in OCR from Lossy Visual Inputs » : les VLM hallucinent sans signaler d'incertitude sur entrées dégradées.
- https://arxiv.org/pdf/2604.26462 — étude empirique KYC : un pipeline multi-étages surpasse systématiquement l'envoi direct du PDF complet à un VLM.
- https://arxiv.org/pdf/2601.04426 — XGrammar-2 : moteur de génération structurée dynamique, latence par token, adoption comme backend par défaut.
- https://arxiv.org/pdf/2411.15100 — XGrammar : article fondateur, séparation tokens contexte-indépendants/dépendants, gains de 3× (JSON Schema) à 100× (CFG).
- https://blog.squeezebits.com/guided-decoding-performance-vllm-sglang — mesures de surcoût du décodage guidé sur vLLM et SGLang.
- https://arxiv.org/pdf/2501.10868 — JSONSchemaBench : évaluation rigoureuse de la conformité au schéma, limites des moteurs à automate fini sur les schémas récursifs.
- https://arxiv.org/pdf/2605.02363 — « When Correct Isn't Usable » : écart entre conformité au schéma et exploitabilité réelle de la sortie.
- https://arxiv.org/pdf/2506.17203 — scoring de confiance sur extraction structurée ; la confiance auto-déclarée mieux calibrée que la log-probabilité.
- https://arxiv.org/pdf/2606.07534 — PulseBench-Tab (avril 2026) : 1 820 tableaux annotés, 9 langues, 48,1 % avec cellules fusionnées, tailles de 2 à 1 183 cellules.
- https://unstructured.io/blog/agentic-table-parsing-a-composable-approach-to-complex-documents — parsing agentique de tableaux, gain rapporté de 7,5 % de TEDS corrigé.
- https://landing.ai/llms/confidence-scores-vs-visual-grounding-what-each-architecture-actually-tells-you — distinction entre score de confiance et ancrage visuel, et ce que chaque architecture peut réellement garantir.
- https://llms.reducto.ai/ — Reducto : architecture agentique multi-passes, revendication LongExtractBench (225 documents, ~358 pages, 99,6 % précision/rappel) — chiffres fournisseur.
- https://www.extend.ai/resources/multi-page-table-extraction-tools — panorama janvier 2026 des outils d'extraction de tableaux multi-pages et de leurs limites.
