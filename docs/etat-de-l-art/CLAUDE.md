# Contexte local — `docs/etat-de-l-art/`

Ce répertoire contient l'**état de l'art** réalisé avant toute implémentation, en
application de la règle posée dans le `CLAUDE.md` racine : aucune brique technique
n'est retenue sans un document qui la compare à ses alternatives.

## Découpage

L'étude est découpée en huit axes, un fichier par axe, plus une synthèse transverse.

| Fichier | Axe |
|---|---|
| `01-frameworks-agentiques.md` | Socle d'orchestration des agents LLM eux-mêmes |
| `02-ingestion-extract-load.md` | Briques qui vont chercher la donnée dans les sources |
| `03-orchestration.md` | Orchestration et exécution durable |
| `04-documents-non-structures.md` | Extraction depuis PDF, courriels, scans |
| `05-contrats-qualite-tracabilite.md` | Garde-fous : contrats, validation, lignage |
| `06-patterns-agentiques-data.md` | **Axe central** : patterns agentiques appliqués à la donnée |
| `07-gitops-ci-cd.md` | Structure de dépôt, CI/CD, versionnement des prompts |
| `08-souverainete-llm-conformite.md` | Hébergement, choix du modèle, RGPD et AI Act |
| `00-synthese.md` | Synthèse transverse et architecture cible recommandée |

L'axe 6 est le cœur de l'étude : c'est lui qui distingue un véritable cadre agentique
d'un ETL classique. L'axe 5 en est le contrepoids : il définit les garde-fous qui
rendent le non-déterminisme d'un LLM acceptable en production.

## Structure imposée à chaque document d'axe

Pour que les axes restent comparables entre eux :

1. Périmètre et enjeu pour le projet
2. Panorama des solutions (une sous-section par solution)
3. Grille comparative (tableau homogène)
4. Ce que ça implique pour un runtime agentique
5. Recommandation (options classées, et ce qui est écarté avec la raison)
6. Sources (URLs réellement consultées)

Certains axes ajoutent une section d'analyse dédiée à leur tension centrale.

## Règles de rédaction

- **Sourcer ou signaler l'incertitude.** Une version, une date, un tarif ou un
  résultat de benchmark doit venir d'une source consultée et liée. À défaut, écrire
  explicitement « non trouvé » — jamais une estimation présentée comme un fait.
- **Rester critique.** Les limites et les échecs documentés d'un outil ont plus de
  valeur que ses arguments de vente.
- **Dater les constats.** L'écosystème bouge vite ; un comparatif non daté devient
  faux sans prévenir.

## Cycle de vie

Ces documents sont datés et destinés à être **révisés**, pas figés. Une décision qui
en découle est enregistrée dans `docs/adr/`, qui devient alors la référence : l'état
de l'art explique le paysage, l'ADR acte le choix.
