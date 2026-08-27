# ADR 0004 — Contrats de données et validation

- **Statut** : Proposé
- **Date** : 2026-08-27
- **Décideurs** : Repreneur du dépôt ; en attente d'arbitrage du porteur du projet
- **Axes de l'état de l'art concernés** : `05-contrats-qualite-tracabilite`, `00-synthese`

## Contexte

Un pipeline ETL classique est déterministe : on peut raisonner sa conformité par tests
de code. Un runtime agentique casse cette propriété — la sortie dépend du modèle, de sa
version, de la température, du prompt et du contenu source. Trois conséquences que l'ADR
0001 rend structurantes :

- **Le test de code ne suffit plus** : la conformité ne se garantit que par validation de
  la sortie à chaque exécution. La validation entre dans le chemin d'exécution nominal.
- **Les défaillances changent de nature** : l'agent échoue silencieusement et
  plausiblement (valeur bien typée mais fausse, colonnes confondues). Un contrat qui ne
  valide que le schéma (types, présence) ne détecte rien de tout cela ; il faut des règles
  sur les **valeurs** (plages, référentiels, cohérences croisées, réconciliation source).
- **La littérature 2026 distingue contrat définitionnel et applicatif** : la plupart des
  outils « data contract » stockent un YAML sans rien bloquer. Le seul contrat utile ici
  est **exécutable et bloquant**, placé entre l'agent et la cible, avec une voie de sortie
  explicite pour ce qu'il refuse.

Contrainte de licence (synthèse) : la chaîne doit rester auto-hébergeable et sous licence
permissive, un cadre destiné à plusieurs SI ne pouvant dépendre d'un composant qui
interdirait un usage managé pour des tiers.

## Options envisagées

### Option A — ODCS + Data Contract CLI + Pydantic/Instructor

Le contrat **ODCS** (YAML, Apache 2.0) est l'artefact pivot versionné en Git.
`datacontract-cli` (MIT, utilisable comme bibliothèque Python) l'exporte en JSON Schema
pour contraindre l'agent et l'exécute comme suite de tests. **Pydantic** + **Instructor**
valident et relancent l'appel en mémoire, avant tout accès à la cible. Chaîne
intégralement MIT/Apache 2.0, sans SaaS. Coûte l'écriture de la table de quarantaine et
des règles de réconciliation. Détail : axe 5.

### Option B — Adossement à dbt

Tests dbt + Elementary, versionnés en YAML. Coûte l'imposition d'un modèle d'exécution
(SQL post-chargement dans un entrepôt) que le projet n'a pas arbitré, et le moteur dbt
Fusion est en beta sous ELv2. Écartée par défaut ; réévaluable si la cible est un
entrepôt dbt. Détail : axe 5.

Autres candidats : **Soda Core v4** écarté (Elastic License 2.0 sur `main`, « Proprietary »
sur PyPI) ; **Great Expectations** écarté comme pivot (pas un format de contrat portable,
configuration lourde, gouvernance Fivetran trop récente) mais admis en complément ;
**Data Contract Specification** de datacontract.com dépréciée au profit d'ODCS.

## Décision

**Le cadre retient l'option A.**

1. **Artefact pivot unique.** Le contrat **ODCS** versionné en Git est la source unique
   dont on dérive : le JSON Schema qui contraint l'agent, les tests de qualité, le DDL de
   la cible et la documentation. Toute divergence entre ces artefacts dérivés est un bug
   de génération, pas une décision.

2. **Moteur.** `datacontract-cli` (MIT) est le moteur, appelé **comme bibliothèque
   Python** depuis le runtime (pas seulement en CI), pour exporter le JSON Schema et
   exécuter `test`.

3. **Validation en trois points**, tous nécessaires car ils ne détectent pas les mêmes
   fautes :
   - **À l'extraction** — sortie du LLM contrainte par le JSON Schema dérivé du contrat,
     puis validation Pydantic avec **retry automatique borné** (motif Instructor).
   - **Après transformation** — règles métier, référentiels, cohérences croisées, via
     `datacontract test`.
   - **Au chargement** — contraintes structurelles en base (types, clés, intégrité),
     seule barrière qu'un bug applicatif ne peut pas contourner.

4. **Quarantaine plutôt que rejet.** Tout enregistrement non conforme part dans une table
   de quarantaine avec sa charge utile brute, le verdict détaillé, le contexte d'inférence
   et un statut. Le rejeu est **idempotent** (clé déterministe dérivée de source + hash)
   et ouvre trois issues : rejeu automatique (modèle/prompt corrigé), correction humaine,
   abandon documenté. **Jamais d'écrasement en place** : les tentatives sont versionnées.

5. **Seuil de mise en quarantaine paramétrable.** Le *mécanisme* ci-dessus est gravé ; le
   *seuil* (strict, ou dégradé avec score de confiance) est un **paramètre par contrat**,
   tranché par le métier de chaque SI. Cet ADR ne fixe pas de seuil uniforme.

6. **Détection de dérive, deux phénomènes distincts.** Dérive *de source* (schéma amont
   modifié) : `datacontract changelog`/`lint` sur un schéma réimporté périodiquement.
   Dérive *de modèle* (même entrée, sortie différente) : non couverte par les outils data
   classiques — traitée par un **jeu de régression figé** rejoué en CI à chaque changement
   de modèle ou de prompt, plus un suivi statistique en production (taux de rejet, taux de
   nuls, distribution des valeurs par version de modèle).

7. **Réconciliation source↔cible** : comptages par lot, sommes de contrôle sur colonnes
   numériques, échantillon relu par un humain, unicité des clés — exprimés comme règles
   `sql` du contrat ODCS. `data-diff` (archivé depuis mai 2024) n'est pas utilisé.

8. **Renvois.** Le lignage et la provenance par valeur (facette OpenLineage) sont l'objet
   de l'ADR 0005. Le catalogue (DataHub) est une phase 2 hors périmètre de cet ADR.

## Justification

L'option A est retenue parce qu'elle est la seule chaîne du panel qui soit à la fois un
*format de contrat portable* (ODCS), un *moteur exécutable et bloquant* appelable en
processus (`datacontract-cli`) et une *boucle de validation-retry* adaptée au
non-déterminisme du LLM (Pydantic/Instructor) — le tout MIT/Apache 2.0 et
auto-hébergeable. Elle réalise le principe « un seul artefact déclaratif » qui découle
directement de l'ADR 0001 : le contrat sert à générer, valider et documenter.

La validation en trois points, plutôt qu'en un seul, répond au mode de défaillance propre
à l'agent : chaque point attrape une classe de fautes que les autres laissent passer
(format à l'extraction, sens après transformation, intégrité au chargement).

Le seuil de quarantaine est laissé paramétrable parce que l'axe 5 le qualifie
explicitement d'arbitrage métier : figer un seuil uniforme sur un SI non défini reviendrait
à décider à la place d'un métier qu'on ne connaît pas encore.

## Conséquences

### Ce que ça nous apporte

- Un contrat unique en Git dont tout le reste dérive, cohérent avec le GitOps.
- Une conformité validée à l'exécution, seule garantie possible en runtime agentique.
- Une détection des deux dérives (source et modèle), souvent confondues ailleurs.
- Une chaîne entièrement permissive et sans SaaS.

### Ce que ça nous coûte

- L'écriture et la maintenance de la table de quarantaine et de son rejeu idempotent.
- La constitution d'un **jeu de régression figé** pour la dérive de modèle : poste souvent
  sous-budgété (annotation), à provisionner.
- Les règles de réconciliation source↔cible à réimplémenter en règles `sql` ODCS.

### Ce que ça nous ferme

- L'adossement à dbt comme socle de qualité. Réouvrable si un SI cible a déjà un entrepôt
  dbt, par un ADR complémentaire.
- Soda Core v4 tant que sa licence (ELv2) n'est pas arbitrée juridiquement.

## Critères de réexamen

Cette décision devra être rediscutée si :

- **ODCS ou `datacontract-cli` changent de licence**, ou si `datacontract-cli` réintroduit
  une dépendance à une brique non permissive (critère licence bloquant, cf. synthèse).
- **La Data Contract Specification cesse d'être supportée** (fin 2026) sans que la
  migration ODCS soit achevée — sans objet ici puisqu'on démarre sur ODCS, mais à vérifier.
- **Un SI cible est un entrepôt dbt** : déclenche l'évaluation de l'option B en complément.
- **Le volume de quarantaine devient ingérable** sous le seuil retenu par un SI : signale
  un mauvais réglage du paramètre du point 5, à rediscuter avec le métier.
- **Le besoin d'un catalogue** apparaît (plusieurs connecteurs, plusieurs consommateurs) :
  ouvre la phase 2 (DataHub), hors de cet ADR.
