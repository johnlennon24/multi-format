# ADR 0001 — Frontière entre ce qui est décidé par LLM et ce qui est figé en code

- **Statut** : Proposé
- **Date** : 2026-08-27
- **Décideurs** : Repreneur du dépôt ; en attente d'arbitrage du porteur du projet
- **Axes de l'état de l'art concernés** : `06-patterns-agentiques-data` (central), `00-synthese`, `04-documents-non-structures`, `05-contrats-qualite-tracabilite`

## Contexte

Le cadrage initial retenait un **runtime agentique intégral** : des agents LLM
exécutant réellement l'extraction et la transformation en production. L'état de l'art
(axe 6, et ouverture de la synthèse) réfute cette cible sur des mesures, et converge
vers une position plus étroite. C'est la décision qui commande tous les autres ADR :
coût, reproductibilité et surface d'attaque en dépendent. Elle doit donc être tranchée
en premier.

Trois faits cadrent le problème :

- **Le coût est piloté par l'architecture, pas par le modèle.** Un même travail varie
  d'un facteur 139× selon l'échafaudage (arXiv 2608.08654). Compiler la décision du
  modèle vers des fonctions déterministes revient de 5,4 à 10,7× moins cher (SemBaker,
  arXiv 2608.06677).
- **Les erreurs par élément se composent.** À 97 % de justesse par ligne — valeur
  optimiste — un lot d'un million de lignes contient 30 000 erreurs, non déterministes
  donc non reproductibles. Aucun taux par ligne acceptable individuellement ne survit
  au volume.
- **Les échecs sont majoritairement silencieux.** Un résultat « plausible mais faux »
  est pire qu'une exception (arXiv 2603.25764 : 100 % de patchs soumis, 44 % résolus,
  échecs sémantiques silencieux dominants).

Le projet est par ailleurs en GitOps (Git source de vérité) : un traitement qui rappelle
un LLM sur chaque enregistrement n'est ni rejouable à l'identique, ni couvrable par des
tests de non-régression, ni auditable.

## Options envisagées

### Option A — Runtime agentique intégral

Le LLM voit et traite chaque enregistrement en production. Détail et mesures : axe 6.
Apporte une souplesse maximale et une architecture simple à décrire. Coûte le facteur de
prix par volume, la non-reproductibilité des décisions prises au vol, la composition des
erreurs, et une surface d'attaque par injection indirecte sur chaque ligne lue.

### Option B — Décider par LLM, exécuter par code

Le LLM produit des **artefacts** — mapping, requête, parseur, jeu de règles. Ce sont ces
artefacts, versionnés en Git et testés, qui touchent le volume. Le LLM ne traite jamais
chaque ligne. Apporte le coût compilé, la reproductibilité par construction, et une
surface d'attaque réduite (le modèle produit un artefact soumis à validation, il n'agit
pas sur la donnée). Coûte l'effort de compilation décision→artefact et la nécessité de
gérer le cycle de vie des artefacts (échantillonnage, validation sur jeu disjoint,
détection de dérive). Détail et mesures : axe 6.

## Décision

**Le cadre retient l'option B, en régime strict.**

1. **Principe directeur.** Le LLM produit des artefacts versionnés (mapping, requête,
   parseur, règle de validation). Ce sont ces artefacts, relus et testés, qui traitent
   le volume. Le LLM ne touche jamais chaque enregistrement.

2. **Régime strict, exceptions énumérées.** Le principe est contraignant en revue. La
   seule dérogation admise à ce jour est celle des **documents non structurés** (PDF,
   courriels, scans) : pas de schéma source à mapper, l'inférence par unité est
   irréductible (axe 4). Là, le LLM *est* l'extracteur, et la validation reste
   déterministe en aval. **Toute autre dérogation future exige un nouvel ADR** ; elle ne
   peut pas être introduite au fil du code.

3. **Nature des artefacts.** Deux formes sont admises :
   - **Déclarative** — mapping, requête, règles sous forme de configuration ou de SQL
     commité, que le cadre interprète. Disponible sans condition.
   - **Code généré** — le LLM écrit du code (par exemple un parseur pour un format que
     nul artefact déclaratif ne couvre), exécuté ensuite au volume. **Cette forme est
     autorisée dans son principe, mais son exécution effective est subordonnée à
     l'acceptation préalable de l'ADR sur le bac à sable d'exécution** (isolation réseau
     par défaut, image reproductible, limites CPU/mémoire/temps). Tant que cet ADR n'est
     pas accepté, le code généré n'est pas exécuté en production. La raison est à l'axe 6
     (« code as action ») : le code généré dépend du contenu des données lues, c'est donc
     formellement du code fourni par un tiers non fiable (injection indirecte).

4. **Grille normative.** Le tableau ci-dessous, repris de l'axe 6, a valeur de règle
   opposable en revue. Il tranche, pour chaque situation, qui de l'agent ou du code
   déterministe est responsable.

| Situation | Agent LLM | Code déterministe |
|---|---|---|
| Découvrir et décrire une source inconnue | Oui | Non |
| Proposer un mapping source → cible | Oui, sur échantillon | Non |
| Appliquer le mapping au volume | Non | Oui |
| Écrire une requête d'extraction | Oui, en assistance | Oui, à l'exécution |
| Exécuter une requête sur la production | Non | Oui |
| Écrire un parseur pour un format tordu | Oui | Non |
| Parser un volume de fichiers avec ce parseur | Non | Oui |
| Extraire des champs d'un document non structuré | Oui, sous contrainte de schéma | Non |
| Valider type, cardinalité, contrainte | Non | Oui |
| Décider si un lot est conforme | Non | Oui |
| Diagnostiquer et classer un échec technique | Oui | Non |
| Réparer un incident technique idempotent | Oui, borné | Oui |
| Réparer en modifiant la sémantique de la donnée | Jamais | Jamais sans humain |
| Détecter une dérive de schéma | Oui, pour qualifier | Oui, pour détecter |
| Ordonnancer, réessayer, gérer les dépendances | Non | Oui |
| Résumer ou qualifier librement un contenu | Oui | Non |

5. **Hors périmètre de cet ADR.** Le point de contrôle humain sur les artefacts (revue
   au commit) n'est pas tranché ici : il relève d'un ADR ultérieur sur la CI et la
   gouvernance. Cet ADR pose la frontière, pas le processus de validation qui l'encadre.

## Justification

L'option B est retenue parce que les trois contraintes du contexte — coût, composition
des erreurs, reproductibilité GitOps — la désignent indépendamment, et parce qu'elle
offre en prime la meilleure parade structurelle au risque principal du cadre, l'injection
de prompt indirecte (synthèse, section risque) : un modèle qui produit un artefact
validé a une surface d'attaque très inférieure à un modèle qui agit sur la donnée.

Le régime strict, plutôt que souple, découle du constat de la synthèse selon lequel la
gouvernance protège mieux ce cadre que la technique : une frontière qui s'érode au fil du
code sans laisser de trace rendrait la règle inopérante là où elle a le plus de valeur.

Graver la grille comme norme, plutôt que la laisser indicative, rend la règle
directement applicable en revue : un désaccord se trouve dans le tableau, il ne se
rediscute pas à chaque cas.

## Conséquences

### Ce que ça nous apporte

- Un coût dominé par du code déterministe, pas par des appels LLM au volume.
- Des traitements reproductibles et testables en non-régression, cohérents avec le GitOps.
- Une surface d'attaque réduite face à l'injection indirecte.
- Une règle de partage agent/code opposable dès la revue.

### Ce que ça nous coûte

- L'effort de compiler chaque décision du modèle en artefact validé (échantillonnage
  informé, jeu de contrôle disjoint, assertions déterministes).
- **Une dépendance dure et assumée** : l'autorisation du code généré (point 3) reste
  inerte tant que l'ADR bac à sable n'est pas accepté. Ce choix ouvre une capacité dont
  le garde-fou n'est pas encore instruit ; il faut le traiter avant toute mise en
  production de code généré, sous peine d'exécuter du code d'origine non fiable sans
  isolation.
- Un trou temporaire assumé sur la revue humaine des artefacts, renvoyée à un ADR
  ultérieur ; d'ici là, la responsabilité de valider un artefact avant son exécution au
  volume n'est pas formellement attribuée.

### Ce que ça nous ferme

- Le runtime agentique intégral (option A). Réouvrable seulement si les mesures qui le
  disqualifient (coût, composition des erreurs) cessent d'être vraies — ce qui
  supposerait un changement d'ordre de grandeur documenté, acté par un nouvel ADR.
- L'ajout d'exceptions au fil de l'eau : toute nouvelle exception passe par un ADR.

## Critères de réexamen

Cette décision devra être rediscutée si l'un de ces événements survient :

- **L'ADR bac à sable est rejeté ou durablement bloqué.** Alors le point 3 (code généré)
  doit être révisé : soit on se limite aux artefacts déclaratifs, soit on rouvre la
  question de l'isolation.
- **Un écart d'ordre de grandeur** sur le coût par élément ou la justesse par ligne (par
  exemple un modèle rendant l'exécution par ligne compétitive en prix et reproductible),
  documenté sur source primaire.
- **Un besoin métier avéré** qui ne rentre dans aucune case de la grille et qu'aucun
  artefact ne couvre — déclencheur d'un ADR d'exception, pas d'une dérogation locale.
- **Une évolution du périmètre du cadre** (destination de la donnée, régime d'exécution)
  qui déplacerait la frontière décision/exécution.
