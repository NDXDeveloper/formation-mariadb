🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.12 — GitOps pour les bases de données

> **Positionnement.** Cette section conclut le chapitre 16 en en faisant la synthèse. L'Infrastructure as Code ([§16.1](01-infrastructure-as-code.md)), l'operator Kubernetes ([§16.5](05-mariadb-operator.md)), la CI/CD ([§16.7](07-cicd-bases-donnees.md)), la gestion des migrations ([§16.8](08-gestion-migrations.md)) et le *monitoring as code* ([§16.9.2](09.2-dashboards-grafana.md)) convergent vers un même modèle opérationnel : **GitOps**. Mais appliquer GitOps à une base de données soulève une difficulté que les applications sans état ne connaissent pas — et c'est précisément cette tension que cette section vise à éclairer.

## Qu'est-ce que GitOps ?

GitOps est un modèle opérationnel dans lequel **Git devient la source unique de vérité** de l'état désiré du système, et où un agent automatisé fait en sorte que l'état réel y corresponde en permanence. On ne se connecte plus à un serveur pour « faire » une modification : on modifie une description dans Git, et le système s'aligne tout seul.

L'initiative *OpenGitOps* en formalise quatre principes :

- **Déclaratif** : l'état désiré est décrit (ce que l'on veut), non scripté (comment l'obtenir).
- **Versionné et immuable** : cet état vit dans Git, avec tout l'historique, les revues et la traçabilité que cela implique.
- **Tiré automatiquement** (*pulled*) : un agent dans le cluster récupère l'état désiré depuis Git, plutôt que de le recevoir d'un système externe.
- **Réconcilié en continu** : l'agent compare sans cesse l'état réel à l'état désiré et corrige les écarts.

## GitOps n'est pas (tout à fait) la CI/CD : *push* contre *pull*

La CI/CD classique ([§16.7](07-cicd-bases-donnees.md)) fonctionne en mode **push** : un pipeline, déclenché par un commit, *pousse* activement les changements vers l'environnement cible (il exécute `kubectl apply`, lance un déploiement…). Le pipeline détient les accès à la production et agit dessus de l'extérieur.

GitOps inverse le flux en mode **pull** : un **contrôleur résidant dans le cluster** — typiquement **Argo CD** ou **Flux** — surveille le dépôt Git, détecte les écarts et applique lui-même les changements depuis l'intérieur. C'est ce qu'on appelle la **boucle de réconciliation**. L'avantage est double : les accès à la production ne sortent jamais du cluster (le pipeline externe n'a plus besoin de droits d'admin), et toute **dérive** est détectée, qu'elle vienne d'un commit ou d'une modification manuelle non autorisée.

```yaml
# Argo CD : déclaration de "quoi synchroniser depuis Git"
apiVersion: argoproj.io/v1alpha1  
kind: Application  
metadata:  
  name: mariadb-prod
spec:
  source:
    repoURL: https://git.exemple.com/infra/db.git
    path: clusters/prod
    targetRevision: main
  destination:
    namespace: data
  syncPolicy:
    automated:
      selfHeal: true
      prune: false          # ⚠️ crucial pour le stateful — voir plus bas
```

## La tension fondamentale : GitOps a été conçu pour le *stateless*

Voici le point le plus important de cette section, et celui que les présentations enthousiastes passent souvent sous silence.

La boucle de réconciliation repose sur deux hypothèses implicites, parfaitement vérifiées pour une application sans état : les opérations sont **idempotentes** (réappliquer l'état désiré ne casse rien) et **réversibles** (revenir à l'état précédent suffit à annuler un changement). Pour un `Deployment` Kubernetes, c'est vrai : si quelqu'un modifie manuellement le nombre de réplicas, le contrôleur le rétablit sans dommage ; si une nouvelle version pose problème, on *revert* le commit et l'ancienne image redéploie — l'état antérieur est intégralement retrouvé.

**Une base de données viole ces deux hypothèses.** Son « état » comprend les **données**, qui ne sont pas — et ne doivent jamais être — dans Git. On peut bien déclarer le *schéma* souhaité, mais réconcilier la base vers ce schéma peut signifier exécuter un `DROP COLUMN` : une **perte de données définitive**. Et surtout, *revenir en arrière n'est pas une opération Git* : annuler le commit qui supprimait la colonne ne **restaure pas** les données perdues. Les migrations de schéma sont, dans le cas général, **ordonnées et irréversibles** — l'inverse exact du modèle « cattle, not pets » sur lequel GitOps a été bâti.

Il serait donc dangereux de traiter naïvement une base comme un `Deployment`. La résolution de cette tension tient en un principe : **séparer ce qui se réconcilie de ce qui ne se réconcilie pas.**

| Dimension | Nature | Compatible réconciliation GitOps ? |
|-----------|--------|-------------------------------------|
| **Infrastructure** (réplicas, topologie, ressources, config moteur) | Déclarative, idempotente, réversible | ✅ Oui — terrain idéal |
| **Schéma & données** | Ordonné, irréversible, perte possible | ⚠️ Non — exige une discipline de migration |

## Terrain idéal : l'infrastructure de la base en GitOps

L'**infrastructure** de MariaDB se prête parfaitement à GitOps. Grâce au [`mariadb-operator`](05-mariadb-operator.md), tout l'état d'un cluster s'exprime de façon déclarative dans une ressource personnalisée (CRD), versionnée dans Git :

```yaml
# git: clusters/prod/mariadb.yaml — état désiré, versionné
# (manifeste simplifié ; schéma réel détaillé en §16.5)
apiVersion: k8s.mariadb.com/v1alpha1  
kind: MariaDB  
metadata:  
  name: mariadb-prod
spec:
  replicas: 3
  galera:
    enabled: true
  storage:
    size: 100Gi
  myCnf: |
    [mariadb]
    innodb_buffer_pool_size = 4G
```

L'operator agit ici comme le **moteur de réconciliation spécialisé** : Argo CD applique le manifeste, et l'operator façonne le cluster réel pour qu'il s'y conforme. Au-delà du cluster lui-même, l'operator expose d'autres objets déclaratifs — bases, utilisateurs, privilèges, sauvegardes planifiées ([§16.5.4](05.4-backups-automatises.md)) — qui deviennent autant de ressources gérables en GitOps. Combiné aux dashboards versionnés ([§16.9.2](09.2-dashboards-grafana.md)), on obtient une définition **entièrement déclarative** de la plateforme : topologie, configuration, accès, sauvegardes et supervision, le tout dans Git.

Ce périmètre est idempotent et réversible : modifier `innodb_buffer_pool_size` ou passer de 3 à 5 réplicas se réconcilie sans risque, et se *revert* proprement.

> 🆕 **Note 12.x — packaging Galera.** Depuis la série 12.x, Galera est livré dans un **paquet `mariadb-server-galera` séparé** ([§14.2.5](../14-haute-disponibilite/02.5-packaging-galera.md)), avec un impact sur les images officielles ([§16.3.1](03.1-images-officielles.md)) et le déploiement par l'operator ([§16.5.2](05.2-deploiement-galera.md)). Lors d'un déploiement Galera piloté en GitOps, il faut donc s'assurer que l'image et la version d'operator référencées dans Git intègrent bien ce changement de packaging.

## Le cas épineux : les migrations de schéma

C'est ici que la discipline remplace la réconciliation aveugle. Deux approches coexistent.

### Approche 1 — Le *migration runner* déclenché par GitOps

L'approche la plus répandue conserve les outils de migration vus en [§16.8](08-gestion-migrations.md) (**Flyway**, **Liquibase**), mais les exécute comme des **Jobs Kubernetes** orchestrés par le contrôleur GitOps. Avec Argo CD, on utilise pour cela un **hook de synchronisation** : un Job de migration s'exécute *avant* le déploiement applicatif (`PreSync`), garantissant que le schéma est à jour avant que le nouveau code ne tourne.

```yaml
apiVersion: batch/v1  
kind: Job  
metadata:  
  name: db-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync   # exécuté avant la synchro
spec:
  template:
    spec:
      containers:
        - name: flyway
          image: flyway/flyway
          args: ["migrate"]
      restartPolicy: Never
```

Dans ce modèle, Git versionne les **scripts de migration ordonnés** (et non un « état final » réconcilié) ; le runner les applique séquentiellement et de façon idempotente (chaque migration n'est jouée qu'une fois). C'est impératif sous le capot, mais déclenché et tracé par GitOps.

### Approche 2 — Le schéma déclaratif (*schema-as-code*)

Une approche plus récente rapproche réellement le schéma du modèle GitOps. Des outils comme **Atlas** (ou Liquibase dans son mode déclaratif) permettent de décrire le schéma *cible* dans Git, puis de **calculer automatiquement la migration** (un *diff* entre le schéma désiré et le schéma réel) — avec des **garde-fous** : détection des opérations destructrices, prévisualisation du plan (`plan`/`diff`), et possibilité d'imposer une revue humaine avant application. Certains de ces outils proposent leur propre operator Kubernetes, intégrable à la boucle GitOps.

Cette approche offre l'expérience la plus proche du « véritable » GitOps pour le schéma, mais elle **ne supprime pas** le problème de fond : un *diff* qui implique de supprimer une colonne reste une perte de données. Le garde-fou ne fait que rendre cette conséquence **visible et bloquante**, au lieu de l'exécuter silencieusement.

### La discipline transverse : *expand/contract* et *forward-only*

Quelle que soit l'approche, deux pratiques rendent les migrations compatibles avec un déploiement continu :

- **Forward-only** : on ne « revert » pas une migration. Pour annuler un changement, on écrit une *nouvelle* migration corrective — jamais on ne rejoue le passé à l'envers.
- **Expand / contract** (migrations en plusieurs phases) : on ajoute d'abord sans rien casser (phase *expand* : nouvelle colonne nullable, double écriture), on bascule le code, *puis* seulement on retire l'ancien (phase *contract*). C'est ce qui permet le zéro-interruption ([§19.8](../19-migration-compatibilite/08-zero-downtime-migrations.md)) et s'appuie sur l'ALTER non bloquant ([§18.11](../18-fonctionnalites-avancees/11-online-schema-change.md)).

## Les secrets dans un dépôt Git

GitOps impose une contrainte évidente : **on ne commite jamais un mot de passe en clair**. Or une base de données en a besoin (compte applicatif, réplication, exporter…). Quatre familles de solutions répondent à ce problème :

- **Sealed Secrets** : les secrets sont *chiffrés* avant d'entrer dans Git ; seul un contrôleur dans le cluster détient la clé pour les déchiffrer. Le dépôt ne contient que du chiffré.
- **External Secrets Operator** : le secret reste dans un coffre externe (Vault, AWS Secrets Manager…) ; Git ne contient qu'une *référence* déclarative, et l'operator injecte la valeur réelle dans le cluster.
- **SOPS** : chiffrement de fichiers (souvent adossé à une clé KMS), pris en charge nativement par Flux.
- **HashiCorp Vault** : gestion centralisée, voire secrets dynamiques à durée de vie limitée.

Le point commun : **Git ne voit jamais le secret en clair**, tout en restant la source de vérité de *quels* secrets existent et de *comment* ils sont injectés.

## Détection de dérive et garde-fous pour le *stateful*

La réconciliation automatique offre deux fonctionnalités puissantes — et redoutables sur une base de données.

Le **self-heal** rétablit automatiquement l'état désiré dès qu'une dérive est détectée. Utile, mais à manier avec prudence : un correctif d'urgence appliqué à chaud par un DBA (un index ajouté en pleine incident, cf. [§16.11](11-alerting-incident-response.md)) crée une dérive que le self-heal pourrait annuler. La parade est de **codifier après coup** ce correctif dans Git, plutôt que de le laisser en marge.

Le **prune** est bien plus dangereux : il supprime les ressources qui ne sont plus déclarées dans Git. Sur une application sans état, c'est anodin. Sur une base, **un prune mal maîtrisé peut supprimer un `StatefulSet` ou un `PersistentVolumeClaim`** — donc les données. La règle est absolue : **désactiver le prune automatique pour les ressources avec état** (`prune: false`, comme dans l'exemple Argo CD plus haut), et protéger les volumes (politiques de rétention `Retain`, finalizers). Sur ce périmètre, la prudence prime sur l'automatisme.

## Bénéfices et limites pour les bases de données

Les **bénéfices** sont réels et substantiels. Toute modification de la plateforme devient une *pull request* — donc **auditée, revue et tracée** : on sait qui a changé quoi, quand et pourquoi. L'environnement est **reproductible** (recréer un cluster identique = appliquer le dépôt), la **dérive est détectée**, et Git constitue une **source unique de vérité** partagée par toute l'équipe.

Mais les **limites**, propres au caractère stateful, doivent être intégrées dès la conception :

- **Le rollback d'infrastructure n'est pas le rollback des données.** Revenir sur un paramètre ou une topologie est trivial (revert + réconciliation) ; revenir sur une migration destructrice **ne se fait pas par Git** mais par une **restauration de sauvegarde** ([chapitre 12](../12-sauvegarde-restauration/05-restauration.md)). Confondre les deux est l'erreur la plus grave.
- **Le schéma n'est pas pleinement déclaratif** : il exige une discipline de migration (forward-only, expand/contract) que la réconciliation seule ne procure pas.
- **Le prune et le self-heal sont des armes à double tranchant** sur le stateful, à restreindre explicitement.

## Points clés à retenir

GitOps fait de **Git la source unique de vérité** d'un état désiré que des contrôleurs (Argo CD, Flux) réconcilient **en continu**, en mode *pull* — à distinguer du *push* de la CI/CD classique. Pour une base de données, il faut impérativement **séparer deux périmètres** : l'**infrastructure** (topologie, configuration, accès, sauvegardes, supervision) est idempotente et réversible — c'est le terrain idéal de GitOps, magnifiquement servi par le [`mariadb-operator`](05-mariadb-operator.md) ; le **schéma et les données**, en revanche, sont ordonnés et irréversibles, et relèvent non d'une réconciliation aveugle mais d'une **discipline de migration** (runner déclenché par hook ou schéma déclaratif à garde-fous, toujours *forward-only* et *expand/contract*). On y ajoute une gestion rigoureuse des **secrets** (jamais en clair dans Git) et des **garde-fous stateful** (prune désactivé, volumes protégés). Le bénéfice — auditabilité, reproductibilité, détection de dérive — est majeur, à condition de ne jamais oublier que *le rollback d'un schéma ne restaure pas les données : seule une sauvegarde le fait*.

---

**Fin du chapitre 16.** Ce chapitre a parcouru toute la chaîne DevOps appliquée à MariaDB : de l'Infrastructure as Code et la conteneurisation jusqu'à l'orchestration Kubernetes, la CI/CD, la supervision, l'observabilité, la réponse à incident et, ici, le modèle GitOps qui les fédère. La [Partie 9](../partie-09-integration-fonctionnalites-avancees.md) aborde ensuite l'**[intégration et le développement applicatif](../17-integration-developpement/README.md)**.

↩️ [Section précédente : 16.11 — Alerting et incident response](11-alerting-incident-response.md)

⏭️ [Partie 9 : Intégration, Développement et Fonctionnalités Avancées](/partie-09-integration-fonctionnalites-avancees.md)
