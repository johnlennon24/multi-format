# multi-format — Cadre agentique d'extraction et de traitement de données

> Ce fichier est le point d'entrée de contexte pour tout agent (Claude Code ou autre)
> travaillant sur ce dépôt. Il décrit ce qu'est le projet, ce qu'il n'est pas, et les
> règles de travail. Les sous-répertoires ont leur propre `CLAUDE.md` qui précise le
> contexte local ; lis toujours le plus spécifique en complément de celui-ci.

## Ce qu'est le projet

`multi-format` est un **cadre agentique réutilisable** dont la mission est d'extraire
et de traiter automatiquement de la donnée depuis une ou plusieurs sources d'un
système d'information (SI).

Il est conçu pour être **redéployé sur plusieurs projets et plusieurs SI différents**.
La réutilisabilité n'est pas un objectif secondaire : c'est le critère qui arbitre les
décisions d'architecture. À chaque choix, la question à se poser est « est-ce que ça se
rebranche sur un autre SI sans réécriture ? ».

## Ce que le projet n'est pas

- Ce n'est pas un ETL de plus. Si le résultat final est un pipeline statique écrit à la
  main, le projet a échoué.
- Ce n'est pas une intégration dédiée à un SI particulier. **Aucun SI cible n'est défini
  à ce stade.** Toute source de données est un connecteur enfichable, jamais le cœur.
- Ce n'est pas un assistant de développement. Le choix retenu est un **runtime
  agentique** : les agents s'exécutent en production, pas seulement au moment d'écrire
  le code.

## Décisions structurantes déjà prises

Ces décisions viennent du cadrage initial avec le porteur du projet. Elles ne se
rediscutent pas sans lui.

| Sujet | Décision |
|---|---|
| Place de l'intelligence | **Runtime agentique** — des agents LLM exécutent réellement l'extraction et la transformation en production |
| Hébergement et souveraineté | **Non tranché** — l'état de l'art (axe 8) doit instruire la décision |
| GitOps | **Everything-as-code + CI/CD** — Git est la source de vérité (code, configuration, contrats de données, infrastructure) ; pas d'opérateur de réconciliation imposé à ce stade, mais l'architecture doit permettre d'y basculer plus tard |
| Familles de sources à couvrir | Bases SQL et NoSQL, APIs REST et SOAP, fichiers structurés (CSV, XML, JSON, Excel, Parquet), documents non structurés (PDF, courriels, scans) |
| Langage | Python |

## Points encore ouverts

À trancher avec le porteur du projet, et à ne pas présupposer :

- **Destination de la donnée** : le cadre s'arrête-t-il au chargement, ou couvre-t-il
  aussi la restitution ? Quelle cible (lac de données, entrepôt, base applicative, API) ?
- **Régime d'exécution** : traitement par lots planifié, événementiel, ou temps réel ?
  Quels ordres de grandeur de volumétrie ?
- **Contexte réglementaire** : données à caractère personnel ? Secteur régulé ?
- **Équipe** : contributeur unique ou plusieurs ? Cela dimensionne l'outillage qualité.

## Méthode de travail imposée

1. **L'état de l'art précède l'implémentation.** Aucune brique n'est choisie sans
   qu'un document de `docs/etat-de-l-art/` en justifie le choix face aux alternatives.
   Ne propose pas de code tant que la comparaison n'est pas faite et validée.
2. **La documentation est un livrable, pas un résidu.** Toute décision d'architecture
   donne lieu à un enregistrement dans `docs/adr/`.
3. **Tout est versionné dans Git**, y compris la configuration, les contrats de données
   et les prompts. Un prompt est un artefact de production : il se relit en revue et se
   teste en non-régression comme du code.
4. **Aucun superlatif dans la documentation.** Les affirmations sont sourcées ou
   signalées comme incertaines. On ne recopie pas un argumentaire marketing.

## Organisation du dépôt

```
docs/
  etat-de-l-art/   Comparatifs sourcés, un fichier par axe, plus une synthèse
  adr/             Enregistrements de décisions d'architecture (Architecture Decision Records)
```

La structure de code sera définie par l'axe 7 de l'état de l'art (GitOps et structure
de dépôt) et n'est volontairement pas figée ici.

## Conventions

- Documentation rédigée en **français**, en Markdown.
- Les liens externes cités doivent avoir été réellement consultés.
- Pas d'abréviation non explicitée à sa première occurrence.
