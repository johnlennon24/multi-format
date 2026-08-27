# TODO — où on en est, et ce qui vient ensuite

> Dernière mise à jour : 27 août 2026.
> Ce fichier est un état d'avancement, pas un backlog détaillé. Les décisions actées
> vivent dans `docs/adr/`, le matériau qui les justifie dans `docs/etat-de-l-art/`.

## Fait

- [x] Cadrage du projet : runtime agentique, GitOps everything-as-code, quatre familles
      de sources, aucun SI cible. Consigné dans le `CLAUDE.md` racine.
- [x] Squelette documentaire : `CLAUDE.md` racine et par répertoire, gabarit d'ADR,
      `.gitignore` (pas de secrets, pas de données métier dans le dépôt).
- [x] **État de l'art complet, 8 axes + synthèse** (`docs/etat-de-l-art/`, ~2 100 lignes).

## L'étape en cours : convertir l'état de l'art en décisions

L'état de l'art décrit le paysage ; il ne décide de rien. L'étape présente consiste à
écrire les ADR qui engagent le projet. Ils sont ordonnés par dépendance : le premier
commande tous les autres.

### Décisions qui ne dépendent d'aucun SI cible

Elles peuvent être prises tout de suite.

- [ ] **ADR 0001 — Frontière entre ce qui est décidé par LLM et ce qui est figé en code.**
      C'est la décision structurante du projet. L'état de l'art converge vers « décider
      par LLM, exécuter par code », ce qui nuance le choix initial de runtime agentique
      intégral. À trancher explicitement avant tout le reste : coût, reproductibilité et
      surface d'attaque en dépendent.
      → axe 6, et section d'ouverture de la synthèse.
- [ ] **ADR 0002 — Socle agentique.** Pydantic AI est recommandé (MIT, agnostique du
      fournisseur, mode `'test'` hors-ligne qui rend la CI déterministe).
      → axe 1.
- [ ] **ADR 0003 — Brique d'Extract-Load.** dlt recommandé, comme bibliothèque appelée
      en processus et non comme plateforme à héberger.
      → axe 2.
- [ ] **ADR 0004 — Contrats de données et validation.** ODCS + `datacontract-cli` +
      Pydantic, avec validation en trois points (extraction, transformation, chargement)
      et une quarantaine rejouable.
      → axe 5.
- [ ] **ADR 0005 — Lignage agentique.** Adopter OpenLineage et **spécifier les trois
      facettes manquantes** : ancrage source→valeur, version de prompt, confiance par
      champ. Aucun standard ne les couvre : c'est une part réelle de la valeur du cadre.
      → axe 5.
- [ ] **ADR 0006 — Abstraction du fournisseur de LLM.** Port interne, adaptateur
      OpenAI-compatible unique, `models.yaml` versionné à identifiants épinglés datés.
      Objectif : le fournisseur est un paramètre de configuration, pas une décision
      d'architecture.
      → axe 8.
- [ ] **ADR 0007 — Structure de dépôt et distribution.** Monorepo noyau + instances
      générées par Copier (`copier update` permet de repropager les évolutions du noyau
      aux SI déjà déployés — ce que Cookiecutter ne sait pas faire).
      → axe 7.
- [ ] **ADR 0008 — Politique de licences.** Rendre le critère licence bloquant en revue,
      et documenter une porte de sortie par dépendance structurante. Sur les huit axes,
      la gouvernance a éliminé plus de candidats que la technique.
      → synthèse, section « constats transverses ».

### Décisions qui dépendent du SI cible

À instruire maintenant, à trancher par déploiement. Elles ne bloquent pas la V1 du noyau.

- [ ] **ADR 0009 — Moteur d'exécution durable.** DBOS si un Postgres existe déjà (zéro
      composant nouveau, MIT), Temporal à grande échelle, Restate si la licence BSL 1.1
      est acceptable. Les axes 1 et 3 divergent : à arbitrer une seule fois.
- [ ] **ADR 0010 — Posture d'hébergement du modèle.** SaaS, auto-hébergé ou hybride,
      selon les données personnelles, le secteur et la volumétrie. Rappel de l'axe 8 :
      on auto-héberge pour la souveraineté et la reproductibilité, pas pour économiser.
- [ ] **ADR 0011 — Couche d'orchestration.** Différable. Airflow 3.3 seulement si un
      besoin réel de backfill ou de lignage inter-flux est avéré.

## Sujets non instruits par l'état de l'art

Deux trous identifiés et assumés. À combler avant l'implémentation concernée.

- [ ] **Bac à sable d'exécution du code généré par l'agent.** Sujet de sécurité, non
      couvert. Bloquant dès que l'agent écrit du code exécuté en production.
- [ ] **Évaluation et non-régression des prompts.** Un prompt est un artefact de
      production : il doit se tester en CI. L'axe 7 pose le principe, pas l'outillage.

## Questions ouvertes pour le porteur du projet

Aucune ne bloque l'écriture des ADR 0001 à 0008.

- [ ] La donnée va où ? Le cadre s'arrête-t-il au chargement, ou couvre-t-il la
      restitution ?
- [ ] Régime d'exécution : par lots planifiés, événementiel, temps réel ? Quelles
      volumétries anticipées ?
- [ ] Données à caractère personnel, secteur régulé ?
- [ ] Contributeur unique ou équipe ? Cela dimensionne l'outillage qualité.

## Hygiène du dépôt

- [ ] **Le jeton d'accès personnel GitHub est en clair dans `.git/config`** (il est dans
      l'URL du remote). Le révoquer, en recréer un, et remettre un remote propre.
- [ ] Aucun commit à ce jour. Créer une branche pour la documentation plutôt que de
      pousser sur `main`, conformément à l'approche GitOps retenue.
- [ ] Ajouter la chaîne CI minimale une fois l'ADR 0007 tranché.

## Après les décisions

Dans l'ordre, et pas avant que les ADR 0001 à 0008 soient actés :

1. Poser le squelette du noyau selon l'arborescence de l'axe 7, sans aucun connecteur.
2. Implémenter **un** cas de bout en bout, volontairement étroit.
3. Ne généraliser qu'ensuite. Le recul public sur ces architectures est faible — aucun
   post-mortem industriel nommé n'a été trouvé — ce qui plaide pour une V1 modeste.
