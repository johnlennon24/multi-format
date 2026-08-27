# Axe 7 — GitOps, CI/CD et structure de dépôt

> État de l'art arrêté au **27 août 2026**. Les versions citées ont été relevées en ligne à
> cette date sur les dépôts et documentations officiels (liens en fin de document). Ce qui
> n'a pas pu être confirmé est signalé comme tel.

## Périmètre et enjeu pour le projet

Le cadre sera **redéployé sur plusieurs SI**. La question d'ingénierie n'est donc pas « quel
CI choisir » mais « comment un même noyau sert N instances client sans que la N+1 impose de
réécrire le noyau, et sans que le correctif du noyau oblige à N migrations manuelles ».

Trois contraintes cadrent tout le reste :

1. **Socle d'hébergement non tranché.** Aucune décision ne doit supposer Kubernetes, ni un
   fournisseur de cloud, ni un magasin de secrets particulier. Tout ce qui touche au socle
   passe par une indirection.
2. **Runtime agentique.** Les prompts et les jeux d'évaluation sont des artefacts de
   production. Ils entrent dans le versionnement, la revue et la non-régression.
3. **GitOps au sens du client** (everything-as-code + CI/CD, sans opérateur de
   réconciliation). Il faut préserver l'option Argo CD / Flux sans la payer aujourd'hui.

## Séparer le noyau réutilisable des configurations spécifiques

Trois stratégies de distribution existent, et elles ne s'excluent pas.

| Stratégie | Ce qu'elle résout | Ce qu'elle ne résout pas |
|---|---|---|
| **Paquet Python versionné** (`mf-core==2.3.1`) | Diffusion des correctifs du noyau : `uv lock --upgrade-package` et c'est fait. Contrat de compatibilité explicite par SemVer. | Le squelette de dépôt d'instance (CI, arborescence, conventions) : un paquet ne crée pas de fichiers. |
| **Template de dépôt** (Copier) | Le squelette, et surtout sa **mise à jour** : Copier sait rejouer le diff du template sur un projet déjà généré (`copier update`) et déclarer des migrations entre versions de template. Cookiecutter ne le sait pas — c'est le critère qui tranche entre les deux. | La logique métier : un template qui contient du code devient une divergence de fork au bout de six mois. |
| **Plugins** (points d'entrée `importlib.metadata`) | L'extension par SI : un connecteur propriétaire vit chez le client, se déclare via `[project.entry-points."multiformat.connectors"]`, et le noyau le découvre sans le connaître. | La configuration et l'exploitation. |

**Recommandation : les trois, avec une frontière nette.**

- Le **noyau** est un ensemble de paquets Python versionnés, publiés depuis un dépôt unique
  organisé en workspace uv (lockfile unique à la racine, dépendances internes déclarées par
  `tool.uv.sources` avec `workspace = true`). Attention à la limite documentée : un workspace
  uv impose un `requires-python` commun (intersection des membres) et convient mal si des
  membres ont des exigences conflictuelles.
- Chaque **SI client a son propre dépôt d'instance**, généré par un template Copier hébergé
  dans le dépôt noyau. Ce dépôt ne contient **aucune ligne de logique** : de la configuration
  déclarative, des surcharges de prompts, des secrets chiffrés, des manifestes de déploiement
  et un pipeline qui inclut des composants CI du noyau.
- Tout ce qui est spécifique à un SI et pas exprimable en configuration devient un **plugin**,
  dans son propre paquet, jamais un `if client == "x"` dans le noyau.

La règle de tri, à appliquer en revue : *si la réponse à « est-ce vrai pour le prochain SI ? »
est non, ça ne va pas dans le noyau.*

## Structure de dépôt recommandée

**Dépôt 1 — le noyau (`multi-format`), monorepo workspace uv.**

```
multi-format/
├── pyproject.toml              # racine du workspace uv : [tool.uv.workspace] members
├── uv.lock                     # unique, versionné, source de vérité des versions
├── .python-version
├── packages/
│   ├── mf-core/                # modèle de domaine, ports, runtime agentique
│   │   └── src/mf_core/
│   │       ├── contracts/      # contrats de données : modèles Pydantic + JSON Schema généré
│   │       ├── ports/          # interfaces : SourceConnector, Sink, LlmClient, PromptStore
│   │       ├── runtime/        # boucle agentique, budgets jetons, reprise, journalisation
│   │       └── settings/       # schéma Pydantic Settings de la configuration d'instance
│   ├── mf-connectors-sql/      # un paquet par famille de sources (axe 1-3)
│   ├── mf-connectors-http/     #   REST + SOAP
│   ├── mf-connectors-files/    #   CSV, XML, JSON, Excel, Parquet
│   ├── mf-connectors-docs/     #   PDF, courriels, scans (axe 4)
│   ├── mf-prompts/             # registre de prompts + chargeur + tests de non-régression
│   └── mf-cli/                 # unique exécutable : `multiformat run|validate|plan`
├── prompts/                    # fichiers de prompts du noyau (voir section dédiée)
├── evals/                      # jeux d'évaluation de référence, indépendants du client
├── templates/instance/         # template Copier du dépôt d'instance (copier.yml + arbo)
├── ci/                         # composants CI réutilisables (GitLab) / workflows appelables
├── deploy/base/                # manifestes de base (Kustomize) ou chart Helm du noyau
└── docs/{etat-de-l-art,adr}/
```

**Dépôt 2 — une instance par SI (`mf-instance-<client>`), généré par Copier.**

```
mf-instance-<client>/
├── .copier-answers.yml         # lien vivant vers le template : permet `copier update`
├── pyproject.toml              # dépend de mf-core==2.3.* + connecteurs + plugins maison
├── uv.lock
├── config/
│   ├── base.yaml               # ce qui est vrai dans tous les environnements
│   ├── env/{dev,recette,prod}.yaml   # surcharges, uniquement des deltas
│   └── sources/*.yaml          # une source du SI = un fichier déclaratif
├── prompts/overrides/          # surcharges de prompts, mêmes règles que le noyau
├── evals/                      # cas métier du client, données anonymisées
├── secrets/{dev,recette,prod}.sops.yaml   # chiffré, versionné sans risque
├── deploy/overlays/{dev,recette,prod}/    # patchs Kustomize + image par digest
└── .gitlab-ci.yml              # ~20 lignes : inclut les composants CI du noyau
```

Une variante monorepo unique (noyau + instances dans le même dépôt) est tentable si le client
final est unique et interne. Elle est déconseillée ici : les instances client ont des droits
d'accès, des cycles de recette et souvent des contraintes de confidentialité qui ne se
mélangent pas dans un dépôt partagé.

## Configuration et environnements

**Retenu : YAML en couches, validé par un schéma Pydantic Settings au démarrage.**

- **Pydantic Settings** couvre nativement variables d'environnement, `.env`, répertoire de
  secrets (le format monté par Kubernetes et Docker Swarm), TOML, YAML, JSON, arguments CLI,
  et les magasins AWS Secrets Manager / AWS Parameter Store / Azure Key Vault / Google Secret
  Manager. L'ordre de priorité est personnalisable via `settings_customise_sources`. C'est le
  seul candidat qui donne **validation, typage et priorité de sources dans le même objet**.
- Ordre de priorité à figer : `variables d'environnement > secrets déchiffrés > config/env/<env>.yaml > config/base.yaml > défauts du noyau`.
- **Hydra** est écarté : puissant pour la composition et les balayages de paramètres, mais
  orienté recherche, et le rythme de publication paraît très ralenti (dernière version relevée
  1.3.5, datée du 5 août — l'année n'apparaissait pas sur la page consultée, probablement
  2024 ; à reconfirmer avant toute décision contraire).
- **Dynaconf** offre les couches d'environnement et une intégration Vault/Redis, mais sa
  validation est moins expressive qu'un modèle Pydantic, et la version affichée sur le site
  (3.3.5, datée du 28 mai 2024) suggère un rythme lent.
- **Règle non négociable** : `multiformat validate` charge et valide toute la configuration
  **sans effet de bord**, et c'est une étape bloquante de CI sur chaque environnement. Une
  instance ne doit jamais découvrir une clé manquante en production.

## Gestion des secrets

Le socle n'est pas tranché : il faut un choix qui ne dépende d'aucun socle et qui soit
migrable.

**Aujourd'hui — SOPS avec des clés `age`.** SOPS est un projet CNCF Sandbox, actif (v3.13.3 du
23 juillet 2026), qui chiffre valeur par valeur dans un YAML : le fichier reste lisible et
diffable en revue, seules les valeurs sont opaques. Il accepte `age`, PGP, AWS KMS, GCP KMS,
Azure Key Vault et HuaweiCloud KMS comme fournisseurs de clés — donc **on commence avec `age`
(zéro dépendance d'infrastructure) et on bascule vers le KMS du socle en changeant le
`.sops.yaml`, sans toucher aux fichiers ni au code**. Les artefacts de release sont signés
avec cosign et accompagnés d'une provenance SLSA et d'un SBOM.

**Ce qu'on écarte pour l'instant, et pourquoi :**

- **Sealed Secrets** : chiffrement lié au contrôleur du cluster cible, donc secret non
  portable d'un cluster à l'autre par construction, et rotation des clés de scellement gérée
  par le contrôleur. Ça suppose Kubernetes, ce qu'on ne peut pas supposer.
- **HashiCorp Vault / OpenBao** : la bonne cible à terme pour des secrets dynamiques et une
  rotation réelle. OpenBao (fork de Vault sous gouvernance Linux Foundation / OpenSSF, v2.6.2
  du 18 août 2026, publication régulière) est l'option à privilégier si la souveraineté pèse.
  C'est un service à exploiter : trop coûteux tant qu'on n'a pas de socle.
- **External Secrets Operator** : la brique de convergence si on atterrit sur Kubernetes ; il
  couvre plus de 40 fournisseurs et expose `ClusterSecretStore` / `PushSecret`. Il ne remplace
  pas SOPS, il consomme un magasin existant.

**Décision de conception qui rend tout ça interchangeable** : le code ne lit jamais un magasin
de secrets. Il lit des **variables d'environnement** (ou un répertoire de secrets). Ce qui les
remplit — `sops exec-env`, ESO, Vault Agent, le runner CI — est une affaire de déploiement.

## Chaîne CI/CD

**GitLab CI est le choix par défaut**, parce que le SI français cible l'impose souvent et
parce que les **CI/CD Components** donnent exactement ce dont on a besoin : des unités de
pipeline réutilisables, publiées dans un catalogue, versionnées en SemVer, paramétrées par des
`inputs` typés et validés. Le noyau publie ses composants ; chaque dépôt d'instance a un
`.gitlab-ci.yml` de vingt lignes qui les inclut **en épinglant un tag** (la documentation
GitLab déconseille explicitement `~latest` en production). Un portage GitHub Actions reste
possible via des *reusable workflows* ; Jenkins n'est retenu que sous contrainte client, son
modèle ne donne ni catalogue ni versionnement natif.

| Étape | Outil | Verdict |
|---|---|---|
| Format + lint | `ruff format --check`, `ruff check` (0.16.4, 20/08/2026 ; pas encore de 1.0) | **Bloquant** |
| Typage | `mypy --strict` sur `mf-core` | **Bloquant** sur le noyau, avertissement sur les connecteurs |
| Tests unitaires | pytest, LLM remplacé par une doublure déterministe | **Bloquant** |
| Validation de configuration | `multiformat validate` sur chaque environnement | **Bloquant** |
| Contrats de données | `datacontract lint` + `datacontract changelog` (détection de rupture) | **Bloquant** si rupture non accompagnée d'un incrément majeur |
| Tests de contrat des connecteurs | pytest + cassettes VCR | **Bloquant** |
| Non-régression de prompts | promptfoo sur cassettes | **Bloquant** |
| Build image | uv en multi-étapes, `uv sync --locked --no-editable`, `UV_COMPILE_BYTECODE=1`, `UV_LINK_MODE=copy`, cache mount | **Bloquant** |
| Signature + provenance | cosign, ou `actions/attest@v4` côté GitHub (`id-token: write`, `attestations: write`, vérifiable par `gh attestation verify`) | **Bloquant** |
| Analyse de vulnérabilités | Trivy/Grype sur l'image et le SBOM | **Avertissement** sauf CVE critique exploitable |
| Évaluation LLM complète | jeu d'évaluation sur modèle réel | **Nocturne**, jamais bloquant sur une fusion |
| Promotion d'environnement | job manuel, sur digest d'image | **Manuel** |

`ty` (Astral, 0.0.75 au 26/08/2026, fonctionnalités marquées « preview ») est très prometteur
sur la vitesse mais n'est pas mûr pour être bloquant : à réévaluer, pas à adopter.

**Empaquetage.** uv (0.12.6 au 25/08/2026) pour la résolution et le lock ; image finale en
multi-étapes. Sur les images minimales : `gcr.io/distroless/python3-debian13` existe et est
maintenue automatiquement contre Debian amont ; chez Chainguard, seul `python:latest` est
gratuit, les tags épinglés (donc les seuls utilisables en production reproductible) sont
commerciaux. Recommandation pragmatique : `python:3.13-slim` épinglé par digest pour démarrer,
distroless dès que la surface d'attaque devient un sujet d'audit. **Toute image est référencée
par digest, jamais par tag mobile, dans les overlays de déploiement.** Cible de maturité
raisonnable : SLSA Build L2 (provenance signée par la plateforme de build) dès le premier
déploiement, L3 (isolation du build, clés de signature inaccessibles aux étapes utilisateur)
en objectif.

## Versionner les prompts comme du code

**Git est la source de vérité du prompt, pas une console SaaS.** Un prompt vit dans
`prompts/<domaine>/<nom>.<n>.md` avec un en-tête YAML : identifiant, version SemVer, modèle
cible, paramètres d'échantillonnage, variables attendues. Le chargeur du noyau (`PromptStore`)
résout un identifiant vers un fichier du dépôt et **refuse de démarrer** si le prompt manque
ou si son empreinte ne correspond pas au verrou. Toute modification passe par une merge
request avec son diff de prompt et le résultat des tests de non-régression.

Convention de version : correctif = reformulation sans effet mesuré ; mineure = capacité
ajoutée, sortie compatible ; **majeure = changement de forme de sortie**, donc rupture de
contrat de données. Un changement majeur de prompt suit exactement la procédure d'un changement
majeur de schéma.

Les registres de prompts existants (MLflow Prompt Registry avec versions immuables et alias
`@production` ; Langfuse avec étiquettes `production`/`staging` et cache SDK côté client)
restent utiles **en lecture, pour l'observabilité et la comparaison de performance par
version**. Mais leur argument de vente — « déployer un prompt sans passer par l'ingénierie » —
est exactement ce que le mandat GitOps interdit. Si l'un des deux est adopté, c'est en miroir
poussé depuis la CI, jamais en source.

## Tester un système qui appelle un LLM, en CI

Trois cercles, à budgets et fréquences différents.

1. **Unitaire (chaque commit, coût nul)** : le port `LlmClient` est remplacé par une doublure
   scriptée. On teste l'orchestration, les budgets, la reprise sur erreur, le parsing — pas le
   modèle. C'est là que doit vivre la majorité des tests.
2. **Contrat, sur cassettes (chaque commit, coût nul)** : `pytest-recording` (VCR.py) enregistre
   les échanges HTTP. Il est en mode `none` par défaut, ce qui **fait échouer tout appel réseau
   imprévu** — précisément le garde-fou qu'on veut ; les cassettes se régénèrent avec
   `--record-mode=rewrite`, et `filter_headers` doit retirer l'en-tête d'authentification via la
   fixture `vcr_config`. Une cassette est un artefact versionné, relu en revue.
3. **Évaluation, sur modèle réel (nocturne et avant release)** : promptfoo (`promptfooconfig.yaml`,
   sortie JUnit XML, cache via `PROMPTFOO_CACHE_PATH`/`PROMPTFOO_CACHE_TTL`, seuils de réussite,
   intégrations GitHub Actions / GitLab CI natives) pour la non-régression de prompts ;
   Inspect AI (UK AISI, modèle *datasets / solvers / scorers*) si on a besoin d'évaluations
   agentiques avec bac à sable.

Sur le non-déterminisme : température 0 **ne suffit pas**. La cause dominante documentée est la
non-invariance par taille de lot côté serveur d'inférence — la même requête change de résultat
selon la charge concurrente. Conséquence directe : aucun test ne doit comparer une sortie LLM
caractère par caractère. On teste la conformité au schéma, la présence des champs, des
assertions sémantiques, et un **seuil de réussite agrégé** (par exemple 95 % du jeu), pas
l'égalité.

## Trajectoire vers un GitOps strict

Sept décisions à prendre maintenant, qui coûtent peu et rendent la bascule vers Argo CD
(v3.5.1 au 12/08/2026) ou Flux (v2.9.0, juin 2026, projet CNCF diplômé) mécanique :

1. **Aucun script de déploiement impératif.** La CI produit et pousse un artefact, elle
   n'applique pas d'état à la main.
2. **Manifestes déclaratifs dès le premier jour** dans `deploy/base` + `deploy/overlays/<env>`,
   même si le premier déploiement est un simple `docker compose` — la structure survit.
3. **Images par digest**, jamais par tag mobile. Un opérateur de réconciliation sans digest
   produit une dérive silencieuse.
4. **Séparation stricte build / deploy** : le commit qui change le code ne change pas l'overlay ;
   la promotion est un commit distinct qui ne modifie qu'un digest.
5. **Patron des manifestes rendus** : la CI écrit les manifestes générés dans une branche ou un
   répertoire d'environnement. C'est exploitable immédiatement, et c'est exactement ce que le
   Source Hydrator d'Argo CD (bêta depuis la 3.5.0) reprend en natif.
6. **Configuration applicative depuis un ConfigMap issu de Git**, jamais depuis des variables
   posées par le runner CI — sinon Git cesse d'être la source de vérité au moment précis où on
   en aurait besoin.
7. **Secrets déjà externalisés** derrière des variables d'environnement, chiffrés dans Git par
   SOPS : c'est ce qui permet d'insérer External Secrets Operator plus tard sans toucher au code.

## Recommandation

- **Deux niveaux de dépôts** : un monorepo noyau en workspace uv (paquets `mf-core` +
  connecteurs + plugins), et un dépôt d'instance par SI, généré et **maintenu à jour** par
  Copier (`copier update`) — c'est cette capacité de mise à jour, absente de Cookiecutter, qui
  fait le choix.
- **Trois canaux de réutilisation, pas un seul** : paquet versionné pour le code, template pour
  le squelette, points d'entrée pour l'extension propre à un SI.
- **Configuration** : YAML en couches (`base` + `env/<env>`) validé par Pydantic Settings, avec
  une commande `validate` bloquante en CI.
- **Secrets** : SOPS + `age` maintenant, changement de fournisseur de clés vers le KMS du socle
  plus tard, ESO ou OpenBao si et quand Kubernetes / la souveraineté tranchent.
- **CI** : GitLab CI avec composants versionnés publiés par le noyau ; ruff et mypy bloquants,
  cassettes VCR et promptfoo bloquants, analyse de vulnérabilités en alerte, évaluation LLM
  complète en nocturne. Images par digest, signées, avec provenance.
- **Prompts** : dans Git, versionnés en SemVer, chargés par empreinte, testés en
  non-régression. Un registre SaaS n'est accepté qu'en miroir descendant.

**Points de décision ouverts, à trancher avec le porteur :** (a) GitLab ou GitHub, qui décide de
la forme exacte des composants CI ; (b) dépôts d'instance chez nous ou chez le client, qui
détermine la stratégie de publication du noyau (index privé, ou dépendance Git épinglée) ;
(c) niveau SLSA visé, qui conditionne le durcissement des runners ; (d) budget mensuel
d'évaluation LLM, qui fixe la taille des jeux nocturnes ; (e) adoption ou non d'un registre de
prompts en miroir, à réexaminer une fois l'axe observabilité instruit.

## Sources

- https://docs.astral.sh/uv/concepts/projects/workspaces/ — workspaces uv : lockfile unique, `tool.uv.sources` `workspace = true`, `requires-python` commun par intersection, cas où uv déconseille les workspaces.
- https://github.com/astral-sh/uv/releases — uv 0.12.6 publiée le 25 août 2026.
- https://docs.astral.sh/uv/guides/integration/docker/ — recommandations officielles Docker : multi-étapes, `--locked`, `--no-install-project`, `--no-editable`, `UV_COMPILE_BYTECODE`, `UV_LINK_MODE=copy`, cache mount.
- https://copier.readthedocs.io/en/stable/comparisons/ — tableau Copier / Cookiecutter / Yeoman : mises à jour de template et migrations propres à Copier.
- https://pydantic.dev/docs/validation/latest/concepts/pydantic_settings/ — sources gérées (env, dotenv, secrets dir, TOML/YAML/JSON, CLI, AWS/Azure/GCP) et `settings_customise_sources`.
- https://hydra.cc/docs/intro/ — positionnement d'Hydra, version documentée 1.3.
- https://github.com/facebookresearch/hydra/releases — dernière version relevée 1.3.5, datée du 5 août (année non affichée sur la page consultée) ; rythme de publication très ralenti.
- https://www.dynaconf.com/ — formats, environnements en couches, `Validator`, intégrations Vault/Redis ; version affichée 3.3.5 (28 mai 2024).
- https://github.com/getsops/sops — fournisseurs de clés (AWS/GCP/Azure/HuaweiCloud KMS, age, PGP), statut CNCF Sandbox.
- https://github.com/getsops/sops/releases — v3.13.3 du 23 juillet 2026 ; artefacts signés cosign, provenance SLSA, SBOM.
- https://openbao.org/ — OpenBao, fork de Vault sous gouvernance Linux Foundation / OpenSSF.
- https://github.com/openbao/openbao/releases — v2.6.2 du 18 août 2026 ; v2.6.0 (14 juillet 2026) : sealing de namespace, plugins KMS, framework de workflows.
- https://external-secrets.io/latest/ — plus de 40 fournisseurs, `ClusterSecretStore`, `PushSecret`.
- https://github.com/bitnami-labs/sealed-secrets — modèle `kubeseal`, rotation des clés de scellement, portées strict / namespace-wide / cluster-wide, non-portabilité entre clusters.
- https://docs.gitlab.com/ci/components/ — composants CI/CD : catalogue, versionnement SemVer, `~latest` déconseillé en production, `inputs` typés, structure `templates/`.
- https://docs.github.com/en/actions/how-tos/secure-your-work/use-artifact-attestations/use-artifact-attestations — `actions/attest@v4`, permissions `id-token`/`attestations: write`, `gh attestation verify`.
- https://slsa.dev/spec/v1.1/levels — Build L1 (provenance existante), L2 (provenance signée par la plateforme), L3 (isolation du build, secrets de signature inaccessibles).
- https://github.com/GoogleContainerTools/distroless — images `gcr.io/distroless/python3-debian13` et `-debian12`, suivi automatisé de Debian amont.
- https://images.chainguard.dev/directory/image/python/versions — `python:latest` gratuit, tags épinglés sur accès commercial.
- https://github.com/astral-sh/ruff/releases — Ruff 0.16.4 du 20 août 2026, pas de version 1.0 annoncée.
- https://docs.astral.sh/ty/ — ty, vérificateur de types Astral, positionnement et absence de déclaration de stabilité.
- https://github.com/astral-sh/ty/releases — ty 0.0.75 du 26 août 2026, fonctionnalités « preview ».
- https://python-semantic-release.readthedocs.io/en/latest/ — version 10.6.1, intégration CI, support monorepo documenté.
- https://github.com/datacontract/datacontract-cli — `lint`, `test`, `export`, `changelog` (détection de rupture), `import` ; support ODCS, dbt, Avro, JSON Schema ; images signées cosign.
- https://datacontract.com/ — page consultée : seul le titre a été renvoyé, aucun contenu exploitable ; à reconsulter.
- https://mlflow.org/docs/latest/genai/prompt-registry/ — versions immuables, alias mutables (`@production`), TTL de cache 60 s sur alias, API `mlflow.genai.register_prompt` / `load_prompt`.
- https://langfuse.com/docs/prompt-management/overview — étiquettes production/staging, déploiement sans passer par l'ingénierie, cache SDK côté client, liaison prompt ↔ traces.
- https://langfuse.com/self-hosting — auto-hébergement Docker, fonctionnalités marquées EE nécessitant une clé de licence, dépendances PostgreSQL + ClickHouse + Redis + stockage S3.
- https://www.promptfoo.dev/docs/integrations/ci-cd/ — `promptfooconfig.yaml`, action `promptfoo/promptfoo-action@v1`, GitLab CI, `--fail-on-error`, `PROMPTFOO_CACHE_PATH`/`_TTL`, sorties JSON/HTML/JUnit XML.
- https://github.com/kiwicom/pytest-recording — VCR.py sous pytest : mode `none` par défaut, `--record-mode=rewrite`, `--block-network`, `filter_headers` via la fixture `vcr_config`.
- https://inspect.aisi.org.uk/ — Inspect AI (UK AISI et Meridian Labs) : datasets, solvers, scorers, bacs à sable Docker/Kubernetes, CLI `inspect eval`.
- https://thinkingmachines.ai/blog/defeating-nondeterminism-in-llm-inference/ — cause du non-déterminisme : non-invariance par taille de lot, pas la concurrence flottante ; noyaux invariants par batch comme remède.
- https://github.com/argoproj/argo-cd/releases — Argo CD v3.5.1 du 12 août 2026 ; nouveautés de la 3.5.0.
- https://argo-cd.readthedocs.io/en/latest/user-guide/source-hydrator/ — patron des manifestes rendus, `drySource`/`syncSource`, statut bêta depuis la 3.5.0.
- https://fluxcd.io/blog/ — Flux v2.9.0 (juin 2026), projet CNCF diplômé depuis novembre 2022, système de plugins CLI, intégration OpenBao.
