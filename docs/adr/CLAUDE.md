# Contexte local — `docs/adr/`

Ce répertoire contient les **enregistrements de décisions d'architecture**
(*Architecture Decision Records*, ADR). Un ADR acte un choix ; il ne le réexplore pas.

## Répartition avec l'état de l'art

Les deux répertoires sont complémentaires et ne doivent pas se recouvrir :

- `docs/etat-de-l-art/` **décrit le paysage** : quelles solutions existent, ce qu'elles
  valent, ce qu'elles coûtent. Le ton y est comparatif.
- `docs/adr/` **acte une décision** : ce qu'on retient, à quelle date, pour quelles
  raisons, et ce qu'on accepte de perdre. Le ton y est tranché.

En cas de contradiction entre les deux, **l'ADR fait foi** : il est postérieur et
engage le projet.

## Convention de nommage

`NNNN-titre-en-kebab-case.md`, numérotation à quatre chiffres, jamais réutilisée même
si un ADR est abandonné. Exemple : `0003-choix-du-moteur-orchestration.md`.

## Statuts

Un ADR porte l'un de ces statuts, indiqué en tête de fichier :

| Statut | Signification |
|---|---|
| `Proposé` | Rédigé, en attente d'arbitrage |
| `Accepté` | En vigueur |
| `Rejeté` | Étudié puis écarté — on le conserve, la raison du refus a de la valeur |
| `Déprécié` | N'est plus pertinent, sans remplaçant |
| `Remplacé par NNNN` | Une décision ultérieure a pris le relais |

Un ADR accepté ne se modifie pas et ne se supprime pas. Pour revenir sur une décision,
on écrit un nouvel ADR qui remplace le précédent, et on met à jour le statut de
l'ancien. L'historique des décisions fait partie de la documentation.

## Gabarit

Voir `0000-gabarit.md`. Toute décision structurante doit y passer, en particulier :
choix du socle agentique, du moteur d'orchestration, de la brique d'ingestion, de la
posture d'hébergement du modèle, et de la frontière entre ce qui est décidé par un LLM
et ce qui est figé en code.
