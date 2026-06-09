🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.7 Rollback et contingence

Toute migration repose sur un pari : celui que la nouvelle version se comportera comme prévu. Les tests de compatibilité (§19.6) réduisent l'incertitude de ce pari, mais ne l'éliminent jamais totalement — un incident peut survenir en production, sur un cas que les tests n'avaient pas couvert, sous une charge que l'environnement de validation ne reproduisait pas. La question n'est donc pas seulement « comment migrer », mais « **que faire si la migration échoue après la bascule** ».

C'est l'objet de cette section. Une migration sans plan de retour arrière est une **porte à sens unique** : une fois franchie, on ne peut plus revenir. Or la décision de migrer ne devient véritablement réversible que si la réversibilité a été *préparée à l'avance*. Improviser un rollback en pleine crise, sous la pression d'un incident de production, est la pire des situations. La planification de la contingence consiste précisément à transformer cette porte à sens unique en une porte que l'on peut, si nécessaire, refranchir dans l'autre sens — proprement et rapidement.

---

## 19.7.1 Le défi spécifique d'un retour arrière sur une base de données

Le rollback d'une application sans état est trivial : on redéploie la version précédente du binaire, et le système retrouve exactement son comportement antérieur. Le rollback d'une base de données est d'une nature fondamentalement différente, et bien plus délicate, pour une raison unique : **la base détient un état qui évolue en permanence**.

Dès l'instant où la nouvelle version (12.3) reçoit des écritures de production, ces écritures n'existent que sur elle. Revenir à l'ancienne version (11.8) confronte alors à un dilemme :

- soit on restaure un état antérieur de la base, et l'on **perd les écritures** survenues entre-temps ;
- soit on cherche à **rapatrier ces écritures** vers l'ancienne version, opération techniquement risquée et parfois impossible.

Cette asymétrie — l'application revient en arrière sans coût, la base non — est le cœur du problème. Toute la stratégie de rollback consiste à **minimiser la fenêtre pendant laquelle des écritures non récupérables s'accumulent sur la nouvelle version**, et à se donner les moyens de récupérer ou de réconcilier ces écritures le cas échéant.

---

## 19.7.2 Définir les objectifs : RTO, RPO et fenêtre de rollback

Avant de choisir une stratégie, il faut fixer les objectifs quantitatifs qui la contraignent. Deux indicateurs, issus de la planification de reprise après incident, structurent toute décision de rollback :

- Le **RTO** (*Recovery Time Objective*) : la durée maximale d'interruption de service tolérée. Combien de temps l'organisation peut-elle rester sans base fonctionnelle pendant le retour arrière ?
- Le **RPO** (*Recovery Point Objective*) : la quantité maximale de données que l'on accepte de perdre, généralement exprimée en temps (par exemple « au plus 5 minutes d'écritures »).

Ces deux objectifs orientent directement le choix technique. Un RPO proche de zéro (aucune perte tolérée) impose un mécanisme de réplication inverse (§19.7.5) ; un RPO plus permissif autorise une simple restauration depuis sauvegarde. Un RTO court exige une procédure de rollback rapide, automatisée et répétée.

De ces objectifs découle la notion de **fenêtre de rollback** : la période pendant laquelle un retour arrière reste possible dans le respect du RPO. Cette fenêtre se referme au point de non-retour (§19.7.6).

---

## 19.7.3 La sauvegarde vérifiée, fondation de tout retour arrière

Quelle que soit la stratégie, tout rollback repose en dernier ressort sur une **sauvegarde antérieure à la migration**. C'est le filet de sécurité minimal et non négociable. Avant toute migration, une sauvegarde **complète, récente et — surtout — dont la restauration a été testée** doit exister.

Ce dernier point mérite insistance : une sauvegarde dont on n'a jamais vérifié la restauration n'est pas une sauvegarde, c'est une hypothèse. Le moment de découvrir qu'une sauvegarde est corrompue ou incomplète ne doit jamais être celui où l'on en a un besoin urgent.

> 🔗 Les stratégies, outils (Mariabackup, mariadb-dump) et la vérification des sauvegardes sont traités au chapitre 12. La §12.7 (*Tests de restauration et plan de reprise*) est particulièrement pertinente ici, ainsi que la §12.5.2 (*Point-in-time recovery*).

La combinaison **sauvegarde complète + binary logs** permet une restauration à un instant précis (PITR, §12.5.2) : on restaure la sauvegarde, puis on rejoue les journaux jusqu'au point souhaité. Ce mécanisme est central pour un rollback après migration in-place — sous une réserve importante développée plus bas (§19.7.9) : les journaux produits par la 12.3 ne se rejouent pas nécessairement sur une 11.8.

---

## 19.7.4 Stratégies de rollback selon le type de migration

La faisabilité et le coût d'un rollback dépendent étroitement de la méthode de migration retenue (§19.4).

| Type de migration | Mécanisme de rollback | Perte de données potentielle | Complexité du rollback |
|-------------------|------------------------|------------------------------|------------------------|
| **In-place** (sur place) | Restauration depuis sauvegarde antérieure | Écritures depuis la sauvegarde | Élevée |
| **Logique** (dump / reload) | Re-bascule vers l'ancienne instance préservée | Écritures sur la nouvelle instance | Modérée |
| **Par réplication** | Re-promotion de l'ancienne instance | Quasi nulle (si réplication inverse) | Faible (si préparée) |

**Migration in-place.** C'est le cas le plus contraignant pour le rollback, car **MariaDB ne prend pas en charge le downgrade**. La commande `mariadb-upgrade` (§19.4.1) modifie les tables système, et le format des fichiers de données évolue d'une version majeure à l'autre. On ne peut donc pas simplement redémarrer les binaires 11.8 sur un répertoire de données passé en 12.3. Le seul rollback fiable est la **restauration de la sauvegarde antérieure à la mise à niveau**, ce qui ramène à l'état de la base au moment de la sauvegarde. D'où une règle absolue : prendre une sauvegarde complète *immédiatement* avant la mise à niveau, et idéalement n'autoriser aucune écriture de production sur la 12.3 tant qu'elle n'est pas validée. Il faut aussi **conserver l'ancienne configuration** (`my.cnf`), notamment parce que la 12.3 a retiré des variables — `big_tables`, `large_page_size`, `storage_engine` (§11.2.3) — que l'ancienne version attend peut-être.

**Migration logique.** L'ancienne instance n'est pas modifiée par la migration : elle reste disponible, idéalement maintenue en lecture seule pendant la bascule. Le rollback consiste alors à **repointer les applications** vers cette ancienne instance — opération rapide. Le problème résiduel reste les écritures parties vers la 12.3 pendant la fenêtre d'exploitation, qui devront être réconciliées (§19.7.8).

**Migration par réplication.** C'est l'approche qui offre le meilleur rollback, détaillée ci-dessous. Elle s'appuie sur GTID (§13.4) pour faciliter la bascule et la re-bascule.

---

## 19.7.5 La réplication inverse comme filet de sécurité

Le motif le plus robuste pour un rollback à faible RPO consiste à **maintenir l'ancienne instance synchronisée avec la nouvelle pendant la fenêtre de rollback**, au moyen d'une réplication inverse.

Le principe : après la bascule, la nouvelle instance 12.3 devient la source de production, et l'on configure une réplication **12.3 → 11.8** afin que l'ancienne instance continue de recevoir, en quasi-temps réel, les écritures effectuées sur la nouvelle. Ainsi, en cas d'incident, le rollback se réduit à **re-promouvoir la 11.8** comme source : elle est déjà à jour, et la perte de données est réduite au délai de réplication — proche de zéro.

Cette technique est puissante, mais elle comporte des **limites importantes qu'il faut valider en pré-production** :

- La réplication d'une version **récente vers une version plus ancienne** n'est pas une configuration officiellement garantie. La 12.3, en tant que source, peut émettre dans son journal des événements que la 11.8 ne sait pas interpréter : SQL propre à la 12.3, nouveaux types de données, comportements liés à des fonctionnalités absentes de la 11.8.
- L'architecture du **nouveau binlog intégré à InnoDB** (§11.5.4) modifie la façon dont le flux de réplication est produit ; sa consommation par une 11.8 doit être explicitement testée.

Pour maximiser les chances de succès de la réplication inverse pendant la fenêtre de rollback, deux précautions s'imposent : **s'abstenir d'utiliser les fonctionnalités exclusives à la 12.3** tant que le rollback reste une option (afin que le flux demeure « compatible 11.8 »), et privilégier le **format de binlog `ROW`** (§11.5.2), qui limite les incompatibilités liées à l'interprétation des instructions. Cette discipline temporaire est le prix d'un rollback sûr.

> 🔗 La configuration de la réplication, les coordonnées de binlog et GTID sont traitées au chapitre 13 ; les mécanismes de failover et switchover en §13.8 et §14.6.

---

## 19.7.6 Le point de non-retour

Toute fenêtre de rollback se referme à un moment précis : le **point de non-retour** (PONR). Au-delà, le rollback devient soit impossible, soit synonyme d'une perte de données inacceptable au regard du RPO.

Ce point n'arrive pas par hasard : il correspond à une décision ou à un événement identifiable, par exemple le décommissionnement de la réplication inverse, l'utilisation de la première fonctionnalité 12.3 que la 11.8 ne pourrait pas reproduire, ou l'application d'un changement de schéma incompatible avec l'ancienne version.

L'erreur classique est de franchir ce point **sans s'en rendre compte**, par accumulation progressive. La bonne pratique est au contraire de le **matérialiser explicitement** dans le plan : décider, *en connaissance de cause*, du moment où l'on coupe le filet de sécurité — généralement une fois que la nouvelle version a été observée stable en production pendant une durée convenue. Tant que ce point n'est pas franchi, la discipline de compatibilité descendante (§19.7.5) doit être maintenue.

---

## 19.7.7 Critères de déclenchement et plan de contingence

**Définir les déclencheurs à l'avance.** La décision de rollback ne doit pas être prise à chaud, à l'instinct. Elle s'appuie sur des **critères objectifs définis avant la bascule**, miroirs des critères go/no-go établis pour les tests (§19.6.11) : par exemple un taux d'erreurs applicatives dépassant un seuil, une corruption de données détectée, un effondrement des performances au-delà d'une limite convenue, ou une incompatibilité critique bloquant une fonction métier essentielle.

Ces déclencheurs prémunissent contre un biais redoutable : le **coût irrécupérable** (*sunk cost*). Après des semaines de préparation, la tentation est forte de « pousser encore un peu » alors que les indicateurs sont au rouge, dans l'espoir que la situation se stabilise. Des critères prédéfinis retirent l'émotion et la pression du calendrier de l'équation : si le seuil est atteint, on déclenche, point.

**Documenter le plan dans un *runbook*.** Le plan de contingence prend la forme d'une procédure écrite, opérationnelle, qui ne laisse aucune place à l'improvisation. Un *runbook* de rollback complet précise au minimum :

- les **déclencheurs** (les seuils décrits ci-dessus) ;
- l'**autorité de décision** : qui a le pouvoir de déclencher le rollback, et qui est consulté ;
- la **procédure pas-à-pas** de retour arrière, suffisamment détaillée pour être exécutée par une personne qui n'a pas conçu la migration ;
- le **plan de communication** : qui prévenir, et dans quel ordre (équipes, parties prenantes, utilisateurs) ;
- les **vérifications post-rollback** confirmant le retour à un état sain ;
- une **estimation de durée**, à comparer au RTO pour s'assurer que la procédure tient dans le budget de temps imparti.

---

## 19.7.8 Répétition et réconciliation post-rollback

**Répéter le rollback, pas seulement la migration.** Un plan de rollback qui n'a jamais été exécuté reste une hypothèse. La procédure de retour arrière doit être **répétée en environnement de pré-production**, dans des conditions aussi proches que possible du réel, au même titre que la migration elle-même. Cette répétition valide la procédure, mesure sa durée effective (à confronter au RTO) et forme les personnes qui devront l'exécuter. Une équipe qui a déjà déroulé le rollback une fois l'exécutera bien plus sereinement le jour d'un incident.

**Réconcilier les écritures perdues.** Lorsqu'un rollback est effectivement déclenché, se pose la question des écritures qui ont eu lieu sur la 12.3 pendant la fenêtre d'exploitation et qui ne figurent pas dans l'instance restaurée. Selon la stratégie, ces écritures peuvent être récupérées via le binary log de la 12.3, les journaux applicatifs, ou un mécanisme de capture. Leur réconciliation vers l'instance 11.8 restaurée est souvent une opération manuelle ou semi-automatisée, qui doit elle aussi être anticipée dans le plan.

**Conduire un post-mortem.** Un rollback n'est pas un échec mais une décision de prudence qui a fonctionné. Il doit néanmoins être suivi d'une analyse rétrospective sans recherche de responsable, visant à comprendre la cause de l'incompatibilité ou de l'incident, à renforcer les tests (§19.6) en conséquence, et à corriger le plan avant une nouvelle tentative de migration.

---

## 19.7.9 Spécificités du rollback vers 11.8 depuis 12.3

Plusieurs caractéristiques de la 12.3 ont une incidence directe sur la faisabilité d'un retour arrière vers la 11.8 et méritent une attention particulière :

- **Pas de downgrade in-place.** Comme rappelé en §19.7.4, aucune procédure ne permet de redescendre une instance 12.3 vers 11.8 sans restauration de sauvegarde. C'est la contrainte structurante de tout rollback in-place.
- **Format du binlog** (§11.5.4). Le nouveau binlog intégré à InnoDB affecte aussi bien la réplication inverse (§19.7.5) que le PITR (§12.5.2) : la capacité d'une 11.8 à consommer ou rejouer ce flux doit être vérifiée, et non présumée.
- **Comportement transactionnel** (§6.9). Point rassurant pour ce sens de retour arrière : l'isolation par instantané (`innodb_snapshot_isolation`) est **active par défaut aussi bien en 11.8 qu'en 12.3** (depuis la 11.6.2). Un rollback `12.3 → 11.8` ne modifie donc **pas** ce comportement, et la logique applicative de gestion des conflits reste pertinente des deux côtés. Ce ne serait un sujet que pour un retour vers une version *antérieure* à 11.6.2.
- **Schéma exploitant des nouveautés 12.3.** Certaines structures créées en 12.3 ne « reviennent » pas proprement vers la 11.8. L'exemple concret est celui des **contraintes de clés étrangères homonymes dans des tables différentes** : leur portée est désormais limitée à la table en 12.3 (§18.12), si bien que deux contraintes de même nom — légales en 12.3 — sont **refusées par la 11.8** (`ERROR 1005`). C'est l'argument décisif pour s'abstenir d'utiliser les fonctionnalités exclusives à la 12.3 tant que la fenêtre de rollback reste ouverte.

> 🔗 La liste exhaustive des changements de comportement entre les deux versions est détaillée en §19.10 (*Migration 11.8 → 12.3*) ; elle constitue la grille de lecture des contraintes de rollback.

---

## Points clés à retenir

- Une migration sans plan de rollback est une **porte à sens unique** ; la réversibilité se prépare *avant* la bascule, jamais dans l'urgence d'un incident.
- Le rollback d'une base de données est intrinsèquement difficile car la base **détient un état qui évolue** : revenir en arrière implique de perdre ou de réconcilier les écritures faites sur la nouvelle version.
- **RTO** et **RPO** fixent les objectifs qui contraignent le choix de stratégie ; ils définissent la **fenêtre de rollback**, qui se referme au **point de non-retour** — lequel doit être franchi consciemment.
- Une **sauvegarde complète, récente et dont la restauration a été testée** est le filet de sécurité minimal et non négociable de toute migration.
- **MariaDB ne prend pas en charge le downgrade in-place** : le rollback d'une migration sur place passe obligatoirement par une restauration de sauvegarde, d'où l'importance de conserver l'ancienne configuration.
- La **réplication inverse** (12.3 → 11.8) offre le rollback à plus faible RPO, mais impose une discipline de compatibilité descendante et une validation préalable, car répliquer du récent vers de l'ancien n'est pas garanti.
- Définir des **critères de déclenchement objectifs** neutralise le biais du coût irrécupérable ; un **runbook documenté et répété** transforme la procédure en geste maîtrisé.

> 🔗 **Pour aller plus loin** : §19.6 *Tests de compatibilité* (les critères go/no-go qui alimentent les déclencheurs), §19.8 *Zero-downtime migrations* (stratégies de bascule minimisant la fenêtre de risque), §19.10 *Migration 11.8 → 12.3* (grille des changements de comportement), chapitre 12 *Sauvegarde et Restauration* (notamment §12.5.2 PITR et §12.7 PRA), chapitre 13 *Réplication* (§13.4 GTID, §13.8 failover/switchover) et §14.6 *Solutions de failover automatique*.

⏭️ [Zero-downtime migrations](/19-migration-compatibilite/08-zero-downtime-migrations.md)
