# multi-format

Cadre agentique réutilisable pour l'extraction et le traitement automatiques de données
depuis les sources d'un système d'information : bases SQL et NoSQL, APIs REST et SOAP,
fichiers structurés, documents non structurés.

Le cadre est conçu pour être redéployé sur plusieurs SI. Aucun SI cible n'est défini :
les sources sont des connecteurs enfichables, jamais le cœur.

## Où en est le projet

**Phase actuelle : état de l'art terminé, décisions d'architecture à prendre.**

L'étude préalable est achevée — huit axes comparatifs et une synthèse, environ 2 100
lignes sourcées, dans [`docs/etat-de-l-art/`](docs/etat-de-l-art/). Aucune ligne de code
n'a encore été écrite, et c'est délibéré : la méthode retenue veut qu'aucune brique ne
soit choisie sans qu'un document en justifie le choix face à ses alternatives.

La prochaine étape est la conversion de cet état de l'art en décisions engageantes,
sous forme d'ADR. **Le détail de ce qui reste à faire est dans [`TODO.md`](TODO.md).**

### Le résultat principal de l'état de l'art

Les huit axes convergent vers un principe directeur : **décider par LLM, exécuter par
code**. L'agent produit des artefacts versionnés — mapping, requête, parseur, règles —
et ce sont ces artefacts, non le modèle, qui traitent le volume. Trois raisons mesurées :
le coût, la non-reproductibilité des décisions prises au vol, et la composition des
erreurs (à 97 % de justesse par ligne, un million de lignes produit 30 000 erreurs).

L'agent reste en production et garde l'initiative ; il cesse simplement de traiter les
enregistrements un par un. Seule exception : les documents non structurés, où il
n'existe pas d'artefact déterministe à compiler.

## Organisation du dépôt

| Chemin | Contenu |
|---|---|
| [`CLAUDE.md`](CLAUDE.md) | Contexte du projet, décisions actées, méthode de travail |
| [`TODO.md`](TODO.md) | État d'avancement et prochaines étapes |
| [`docs/etat-de-l-art/`](docs/etat-de-l-art/) | Les huit axes comparatifs et la synthèse |
| [`docs/adr/`](docs/adr/) | Décisions d'architecture (gabarit posé, aucune décision actée) |

Commencer par [`docs/etat-de-l-art/00-synthese.md`](docs/etat-de-l-art/00-synthese.md) :
la synthèse met les axes en tension et propose une architecture cible.

## Méthode

- L'état de l'art précède l'implémentation.
- Toute décision structurante donne lieu à un ADR, avec ses critères de réexamen.
- Git est la source de vérité : code, configuration, contrats de données et prompts.
- Les affirmations sont sourcées, ou explicitement signalées comme incertaines.
