# Synthèse transverse et architecture cible

> Document de synthèse des huit axes de l'état de l'art. Daté du 27 août 2026.
> Il ne remplace pas les axes : il les met en tension et propose une architecture.
> Les décisions qui en découlent seront actées dans `docs/adr/`.

## Le résultat le plus important : le runtime agentique intégral n'est pas la bonne cible

Le cadrage initial retenait un **runtime agentique** — des agents LLM exécutant
réellement l'extraction et la transformation en production. Les huit axes convergent,
indépendamment les uns des autres, vers une conclusion plus nuancée que nous devons
poser franchement plutôt que l'enfouir.

Le principe qui ressort de l'axe 6 est **« décider par LLM, exécuter par code »** : le
modèle produit des *artefacts* — un mapping, une requête, un parseur, un jeu de règles
— et ce sont ces artefacts, versionnés en Git, qui touchent le volume de données. Trois
arguments mesurés le soutiennent :

1. **Le coût.** Compiler la décision du modèle vers des fonctions Python déterministes
   revient de 5,4 à 10,7 fois moins cher (SemBaker, arXiv 2608.06677).
2. **La composition des erreurs.** À 97 % de justesse par ligne, un million de lignes
   produit 30 000 erreurs — et elles ne sont pas reproductibles d'une exécution à
   l'autre. Aucun taux par ligne acceptable individuellement ne survit au volume.
3. **La reproductibilité.** Un artefact versionné se relit en revue, se teste en
   non-régression et se rejoue à l'identique. Une décision prise au vol dans un appel
   LLM, non.

Ce n'est pas un renoncement à l'ambition agentique : c'est son déplacement. L'agent
reste en production et garde l'initiative — il découvre les sources, propose les
mappings, écrit les parseurs, diagnostique les échecs. Ce qu'il cesse de faire, c'est
de traiter les enregistrements un par un. La frontière décision/exécution devient le
véritable objet d'architecture du cadre, et c'est aussi ce qui le rend défendable en SI
d'entreprise.

**Une exception assumée** : les documents non structurés (axe 4). Là, il n'existe pas
d'artefact déterministe à compiler — le modèle *est* l'extracteur. C'est le seul endroit
du cadre où un LLM voit chaque unité de donnée, et c'est aussi le seul où son coût par
unité se justifie.

## Architecture cible proposée

| Couche | Retenu | Raison dominante |
|---|---|---|
| Socle agentique | **Pydantic AI** (MIT) | Agnostique du fournisseur, mode `'test'` hors-ligne rendant la CI déterministe, délègue la durabilité à de vrais moteurs |
| Exécution durable | **DBOS** par défaut, Temporal à grande échelle | Se greffe sur un Postgres existant : zéro composant nouveau, licence MIT |
| Orchestration | **Différée**, Airflow 3.3 si backfill avéré | Deux couches distinctes, mais la seconde n'est pas nécessaire dès la V1 |
| Extract-Load | **dlt** (Apache 2.0) | Seule *bibliothèque* du panel ; les concurrents sont des plateformes à héberger |
| Documents | Pipeline étagé, VLM en second rideau | Voir axe 4 |
| Contrats | **ODCS** + `datacontract-cli` + Pydantic | Déclaratif, versionnable en Git, chaîne intégralement Apache 2.0 / MIT |
| Lignage | **OpenLineage** + facettes propres | Le standard existe mais ne couvre pas le cas agentique (voir plus bas) |
| Accès LLM | Port interne + adaptateur OpenAI-compatible + LiteLLM | Le fournisseur devient un paramètre de configuration |
| Distribution | Monorepo noyau + instances générées par **Copier** | `copier update` repropage les évolutions du noyau aux SI déjà déployés |
| Secrets | **SOPS** + clés `age` | Migrable vers le KMS du socle par simple changement de `.sops.yaml` |

## Trois constats transverses

### La licence élimine plus de candidats que la technique

C'est le fait le plus frappant de l'étude. Sur les huit axes, les disqualifications
tiennent bien plus à la gouvernance qu'aux capacités : Soda Core bascule d'Apache 2.0
vers Elastic 2.0, Estuary est sous BSL jusqu'en 2029, Restate est sous BSL 1.1, Windmill
sous AGPLv3, Great Expectations passe sous Fivetran, Fivetran fusionne avec dbt Labs,
Prefect rachète Dagster Labs, AutoGen et Semantic Kernel entrent en maintenance.

**Conséquence pour un cadre destiné à durer sur plusieurs SI** : le critère licence doit
être bloquant en revue, au même titre qu'un test qui échoue, et chaque dépendance
structurante doit avoir une porte de sortie documentée. Un ADR par brique, avec ses
critères de réexamen, n'est pas de la bureaucratie ici : c'est la seule protection
contre une bascule de licence qui rendrait le cadre indistribuable.

### Le lignage agentique n'existe pas encore comme standard

OpenLineage couvre le lignage colonne à colonne et les assertions de qualité, mais il
lui manque exactement les trois choses dont ce projet a besoin : l'ancrage
source→valeur (savoir qu'un montant vient de tel endroit de la page 7 d'un PDF donné),
la version de prompt comme entité de premier rang, et le score de confiance par champ.

C'est une lacune, mais c'est surtout **une opportunité de différenciation** : ces trois
facettes sont spécifiables dans le cadre et constituent une part réelle de sa valeur
réutilisable. C'est ce qui distingue un cadre d'un assemblage d'outils.

### On auto-héberge pour la souveraineté, jamais pour économiser

L'analyse de coût de l'axe 8 est sans ambiguïté : face aux petits modèles d'API, le
volume d'équilibre est de l'ordre de 22,5 milliards de tokens par mois, un débit
physiquement inatteignable sur deux GPU. La bascule n'existe que face au haut de gamme.
L'auto-hébergement se justifie par la souveraineté, la reproductibilité (des poids figés
sont versionnables) et la conformité — pas par le budget.

## Risque principal : l'injection de prompt indirecte

Les entrées du cadre sont des documents et des données produits par des tiers. Un PDF
peut contenir des instructions destinées au modèle. Ce risque n'est pas théorique dès
lors que l'agent a accès à des sources de données et à des capacités d'écriture.

Le principe « décider par LLM, exécuter par code » en est d'ailleurs la meilleure
parade structurelle : un modèle qui produit un artefact soumis à revue et à validation
de schéma a une surface d'attaque très inférieure à un modèle qui agit directement sur
la donnée. À compléter par le décodage contraint (vLLM), la validation systématique
côté cadre, et le cloisonnement des droits d'écriture.

## MCP : oui pour le plan de contrôle, non pour le transport de données

L'écosystème MCP n'est pas assez stable pour qu'on programme directement contre lui : le
dépôt officiel a archivé ses serveurs PostgreSQL et SQLite, et les implémentations
plafonnent (troncature des réponses, limite du nombre d'outils). MCP a sa place derrière
une interface `Source` interne, jamais comme fondation. Le cadre ne doit pas devenir
l'otage d'un protocole de deux ans.

## Points de décision restant ouverts

Aucun ne bloque le démarrage, mais chacun doit être tranché avant l'implémentation de la
brique concernée.

| Question | Impacte | Qui tranche |
|---|---|---|
| Destination de la donnée : le cadre s'arrête-t-il au chargement ? | Périmètre | Porteur du projet |
| Régime d'exécution et volumétrie | Orchestration, coût | Porteur du projet |
| Présence d'un Postgres / d'un Kubernetes au socle | DBOS contre Temporal contre Restate | Par SI |
| Données personnelles et secteur régulé | Posture d'hébergement | Par SI |
| Acceptabilité des licences BSL et ELv2 | Plusieurs briques | Juridique |
| Compétence GPU et MLOps disponible | Faisabilité de l'auto-hébergement | Par SI |
| Choix du bac à sable d'exécution du code généré | Sécurité | Non instruit, à traiter |
| Outillage d'évaluation et de non-régression des prompts | CI | Non instruit, à traiter |

## Limites de cet état de l'art

À lire avant de s'appuyer sur ces documents :

- **Quota de recherche épuisé.** Le budget de recherche web de la session a été atteint
  en cours d'étude ; plusieurs axes ont terminé leur collecte par accès direct aux
  sources primaires. La couverture est inégale d'un axe à l'autre.
- **Volume hétérogène.** L'axe 4 fait 533 lignes, les autres 136 à 323, l'étude ayant
  été resserrée en cours de route. Le déséquilibre ne reflète pas l'importance relative.
- **Aucun post-mortem industriel nommé n'a été trouvé** sur l'usage d'agents en
  production pour l'ingénierie de données. C'est en soi une information : le recul
  public sur ces architectures est faible, ce qui plaide pour la prudence et pour une
  V1 volontairement modeste.
- **Points explicitement non recoupés**, signalés dans chaque axe : notamment le report
  éventuel du volet haut risque de l'AI Act par un « AI Omnibus » (à vérifier au JOUE),
  et plusieurs chiffres de benchmark ou de tarif marqués « non trouvé » plutôt
  qu'estimés.

## Ce qu'il est raisonnable de faire ensuite

1. Trancher les points de décision qui ne dépendent pas d'un SI cible, et les acter en
   ADR — en priorité la frontière décision/exécution, qui commande tout le reste.
2. Instruire les deux sujets non couverts : bac à sable d'exécution, et évaluation des
   prompts.
3. Poser le squelette du noyau selon l'arborescence de l'axe 7, sans implémenter de
   connecteur.
4. Valider l'architecture sur **un** cas de bout en bout, volontairement étroit, avant
   toute généralisation.
