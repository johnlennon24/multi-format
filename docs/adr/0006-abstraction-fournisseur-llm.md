# ADR 0006 — Abstraction du fournisseur de LLM

- **Statut** : Proposé
- **Date** : 2026-08-27
- **Décideurs** : Repreneur du dépôt ; en attente d'arbitrage du porteur du projet
- **Axes de l'état de l'art concernés** : `08-souverainete-llm-conformite`, `00-synthese`

## Contexte

Le cadre fait tourner des agents LLM en production : le modèle voit la donnée métier
réelle. Trois conséquences (axe 8) : le fournisseur devient un **sous-traitant RGPD**, une
**dépendance d'exploitation** (disponibilité, tarif, dépréciation de modèle,
reproductibilité), et comme **aucun SI cible n'est défini**, le cadre doit supporter les
trois postures d'hébergement (SaaS externe, auto-hébergé, hybride) **sans réécriture**.

La **posture** elle-même dépend du SI (données personnelles, secteur, volumétrie,
compétence GPU) et sera tranchée par déploiement à l'ADR 0010. Le présent ADR ne décide
pas la posture : il décide l'**abstraction** qui rend ce choix réversible. L'objectif est
énoncé par l'axe 8 : *le fournisseur est une ligne de configuration versionnée, jamais un
`import` Python*.

Contrainte GitOps (ADR 0001, méthode du dépôt) : changer de posture ou de modèle doit être
un changement de fichier versionné, prouvé par le rejeu d'une suite d'évaluation.

## Options envisagées

### Option A — Port interne + adaptateur OpenAI-compatible unique + LiteLLM + `models.yaml`

Trois couches (axe 8) : une interface maison `ModelPort` qu'utilise seul le code métier ;
un adaptateur nominal unique, client HTTP **OpenAI-compatible** (couvre les API SaaS,
vLLM, SGLang, llama.cpp, Ollama) ; une passerelle **LiteLLM en proxy auto-hébergé**
(routage, *fallbacks*, budgets, suivi de coût). Un `models.yaml` versionné décrit chaque
profil. Coûte l'écriture et l'entretien de l'abstraction. Apporte la réversibilité totale
de la posture et du fournisseur.

### Option B — Couplage direct à un SDK fournisseur

Le code métier importe `openai`/`anthropic`/`mistralai`. Apporte un démarrage plus rapide.
Coûte un chantier de réécriture le jour où l'on change de posture — exactement le risque
que l'axe 8 identifie comme le plus coûteux. Écartée.

Passerelle managée (**OpenRouter**) comme couche unique : écartée pour un client contraint
car elle **ajoute un maillon à la chaîne de sous-traitance RGPD**.

## Décision

**Le cadre retient l'option A.**

1. **Port interne `ModelPort`.** Le code métier ne connaît qu'une interface maison
   exprimée dans notre vocabulaire, par exemple `extract(document, schema) -> objet
   validé` : elle prend un **schéma Pydantic** (ADR 0004) et rend un objet **validé côté
   cadre**. Aucun module métier n'importe un SDK de fournisseur.

2. **Adaptateur nominal unique.** Un client HTTP **OpenAI-compatible**, plus petit
   dénominateur commun réel du marché, couvrant à la fois les API SaaS et les serveurs
   auto-hébergés (vLLM, SGLang, llama.cpp, Ollama).

3. **Passerelle LiteLLM en proxy auto-hébergé** pour le routage, les *fallbacks*, les
   budgets par clé/équipe et le suivi de coût. Le mode proxy est préféré au SDK : il sort
   la configuration fournisseur du code applicatif. OpenRouter est écarté (maillon RGPD
   supplémentaire).

4. **`models.yaml` versionné.** Chaque profil décrit : fournisseur, **identifiant de
   modèle épinglé à une version datée**, fenêtre de contexte, capacités, coût unitaire.
   Changer de posture ou de modèle = éditer ce fichier et **rejouer la suite
   d'évaluation** — le test qui prouve que l'abstraction tient.

5. **Traitement de ce qui n'est pas portable** (détail : axe 8) :
   - **Sortie structurée** : ne jamais faire confiance au fournisseur ; la garantie est la
     validation Pydantic en sortie avec retry (ADR 0004). La contrainte de décodage côté
     serveur (vLLM/SGLang) est une optimisation, isolée dans l'adaptateur.
   - **Cache de prompt** : sémantiques incompatibles isolées dans l'adaptateur ; discipline
     imposée de **préfixe stable en tête** (consignes + schéma), variable en queue, qui
     profite à tous les backends.
   - **Appels d'outils** : normalisés via LiteLLM ; nombre d'outils exposés par appel
     limité.
   - **Fenêtre de contexte** : déclarée par modèle ; le cadre **découpe**, ne suppose jamais.
   - **Tokenizer** : jamais de budget en tokens codé en dur ; toujours le comptage du
     fournisseur.
   - **Raisonnement** : niveau abstrait (`bas`/`moyen`/`haut`) traduit par l'adaptateur.

6. **Construire pour l'auto-hébergement dès le départ, même en démarrant en SaaS.** Le
   port, la validation Pydantic systématique et le `models.yaml` coûtent peu au début ;
   partir couplé à une API et vouloir rapatrier ensuite est le chantier à éviter.

7. **Sécurité et reproductibilité, par défaut** (axe 8) : journalisation du **contenu
   désactivée par défaut** (seuls identifiants de corrélation et compteurs de tokens),
   **rétention nulle** exigée comme critère d'éligibilité fournisseur exprimé dans la
   configuration, identifiants de modèle **épinglés datés**, jeu d'évaluation *golden*
   rejoué en CI, et modèle réellement utilisé **journalisé** dans les métadonnées de chaque
   extraction (cf. facette d'inférence, ADR 0005).

8. **Renvoi.** La posture d'hébergement (a/b/c) est décidée par SI à l'ADR 0010 ; cet ADR
   la rend réversible sans réécriture.

## Justification

L'option A est retenue parce qu'elle est la seule qui rend la posture d'hébergement et le
fournisseur **réversibles**, ce que le contexte impose absolument puisque aucun SI n'est
défini. Elle réalise la règle de l'axe 8 (« le fournisseur est une ligne de configuration »)
et s'aligne sur le GitOps : le `models.yaml` versionné plus le rejeu d'évaluation
transforment un changement de fournisseur en opération testée, pas en pari.

L'adaptateur OpenAI-compatible unique évite le piège d'un adaptateur par fournisseur : il
couvre déjà le SaaS et l'auto-hébergé. La validation côté cadre, indépendante du
fournisseur, est ce qui rend la sortie structurée fiable quelle que soit la posture — la
contrainte côté serveur n'étant qu'une optimisation quand elle est disponible.

## Conséquences

### Ce que ça nous apporte

- Une posture d'hébergement et un fournisseur interchangeables par configuration.
- Une surface RGPD maîtrisée (pas de maillon managé imposé, rétention nulle par critère).
- La reproductibilité par identifiants épinglés et jeu *golden* en CI.
- L'alignement avec l'auto-hébergement souverain sans engagement prématuré.

### Ce que ça nous coûte

- L'écriture et l'entretien du port, de l'adaptateur et de la couche d'isolation des
  fonctionnalités non portables (cache, raisonnement, tokenizer).
- L'exploitation d'un proxy LiteLLM auto-hébergé.
- La discipline de prompt (préfixe stable) et la tenue du `models.yaml` à jour.

### Ce que ça nous ferme

- Le couplage direct à un SDK fournisseur (gain de court terme contre dette de réécriture).
- Les passerelles managées (OpenRouter) pour les clients contraints ; réouvrable pour un
  client non régulé qui accepte le maillon supplémentaire.

## Critères de réexamen

Cette décision devra être rediscutée si :

- **Le format OpenAI-compatible cesse d'être le dénominateur commun** du marché (un
  fournisseur structurant divergeant sans compatibilité) : imposerait un second adaptateur.
- **LiteLLM change de licence** ou de modèle de distribution d'une façon incompatible avec
  l'auto-hébergement (critère licence bloquant, cf. ADR 0008).
- **La posture d'hébergement tranchée à l'ADR 0010** révèle une contrainte non couverte par
  l'abstraction (par exemple une exigence de cache pilotable absente en auto-hébergé).
- **Les conventions de sortie structurée ou d'appel d'outils se normalisent** entre
  fournisseurs : pourrait simplifier la couche d'isolation.
