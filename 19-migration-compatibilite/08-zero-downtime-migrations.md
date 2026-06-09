🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.8 Zero-downtime migrations

Pour de nombreux systèmes — commerce en ligne, services financiers, plateformes SaaS — une fenêtre de maintenance avec interruption de service n'est plus acceptable. Migrer doit alors se faire *pendant que la base sert le trafic de production*, sans que l'utilisateur final ne perçoive de coupure. C'est l'objet des **migrations sans interruption** (*zero-downtime migrations*).

Cette section décrit les techniques qui rendent cet objectif atteignable, en s'appuyant sur les briques déjà présentées : les méthodes de mise à niveau (§19.4), la réplication (chapitre 13), la haute disponibilité (chapitre 14) et les changements de schéma en ligne (§18.11). Elle prolonge directement la §19.7 : l'architecture de migration par réplication qui permet un rollback à faible perte est la *même* qui permet une bascule sans interruption — ce sont deux facettes d'un seul dispositif.

---

## 19.8.1 Ce que « zéro interruption » signifie réellement

Le terme « zéro interruption » est en partie un idéal. En pratique, une migration de version comporte presque toujours une **fenêtre de bascule** (*cutover*) très brève — de l'ordre de quelques secondes — pendant laquelle les écritures sont suspendues ou re-routées. L'objectif réaliste n'est donc pas l'absence physique de toute transition, mais une **interruption imperceptible pour l'utilisateur** : une reconfiguration de connexions plutôt qu'une panne, une latence ponctuelle plutôt qu'une page d'erreur.

Il est utile de distinguer deux problèmes que la migration sans interruption doit résoudre, car ils relèvent de techniques différentes :

- les **changements de schéma** (`ALTER TABLE`) sur une instance en service, sans verrouiller les tables ;
- la **migration de version ou de plateforme** (passage à une nouvelle instance 12.3), sans arrêter le service — le problème de la bascule.

Les deux sont traités séparément ci-dessous, puis réunis par la couche de routage qui réduit la fenêtre perçue à néant.

---

## 19.8.2 Changements de schéma en ligne

Un `ALTER TABLE` naïf sur une grande table peut la verrouiller pendant toute la durée de sa reconstruction — de quelques minutes à plusieurs heures — bloquant l'application. La migration sans interruption suppose donc des modifications de schéma **non bloquantes**.

MariaDB propose nativement des algorithmes d'`ALTER TABLE` en ligne (§18.11), contrôlés par les clauses `ALGORITHM` et `LOCK` :

```sql
-- Ajout de colonne quasi instantané : modification des seules métadonnées
ALTER TABLE commandes ADD COLUMN statut VARCHAR(20) DEFAULT 'en_attente',
  ALGORITHM=INSTANT;

-- Reconstruction en place autorisant les écritures concurrentes
ALTER TABLE commandes ADD INDEX idx_client (client_id),
  ALGORITHM=INPLACE, LOCK=NONE;
```

`ALGORITHM=INSTANT` traite certaines opérations (ajout de colonne, par exemple) en modifiant uniquement les métadonnées, sans toucher aux données — l'opération est quasi immédiate. `ALGORITHM=INPLACE` reconstruit l'objet sans copie complète de la table et, avec `LOCK=NONE`, autorise les lectures *et* les écritures pendant l'opération.

Lorsque l'algorithme natif ne suffit pas — opération non prise en charge en ligne, besoin d'un contrôle fin du débit, ou volonté de pouvoir interrompre l'opération — on recourt à des **outils externes** (§16.8.3) :

```bash
# pt-online-schema-change : table fantôme, triggers de synchronisation, copie par lots
pt-online-schema-change --alter "ADD INDEX idx_client (client_id)" \
  D=boutique,t=commandes --execute

# gh-ost : approche sans triggers, pilotée par la lecture du binlog
gh-ost --database="boutique" --table="commandes" \
  --alter="ADD INDEX idx_client (client_id)" --execute
```

Ces outils créent une table fantôme, y recopient les données par lots tout en répercutant les écritures concurrentes (par triggers pour `pt-online-schema-change`, par lecture du binlog pour `gh-ost`), puis effectuent un renommage atomique en fin de course. La copie progressive et régulée évite tout verrou long.

> 🔗 Côté réplication, l'**Optimistic ALTER TABLE** (§13.10) réduit le retard induit par une modification de schéma sur les réplicas, et `innodb_alter_copy_bulk` (§15.6) accélère la construction d'index lors d'une copie de table.

---

## 19.8.3 Découpler le schéma du déploiement : le motif expand/contract

Certains changements de schéma ne sont sans interruption que s'ils sont **découplés du déploiement applicatif**. Modifier simultanément la base et le code suppose en effet que les deux versions soient compatibles à l'instant précis de la bascule — condition rarement tenable sans coupure.

Le motif **expand/contract** (ou *parallel change*) résout cela en décomposant un changement en étapes rétrocompatibles, chacune déployable indépendamment :

1. **Expand** — ajouter le nouvel élément (colonne, table) sans retirer l'ancien. L'application existante continue de fonctionner, ignorant le nouvel élément.
2. **Migrate** — déployer une version de l'application qui écrit à la fois dans l'ancien et le nouveau format, puis remplir rétroactivement (*backfill*) les données existantes.
3. **Contract** — une fois les lectures basculées sur le nouveau format et l'ancien devenu inutilisé, le supprimer.

À chaque étape, l'ancienne et la nouvelle version de l'application coexistent sans conflit. Ce découplage est ce qui rend possible une évolution de schéma continue, sans qu'aucune des étapes n'exige l'arrêt du service.

---

## 19.8.4 Migration de version par réplication : le motif de référence

Pour migrer vers une nouvelle version (11.8 → 12.3) sans interruption, le motif canonique est la **migration par réplication**. Plutôt que de mettre à niveau l'instance en place — ce qui impose un arrêt et interdit le downgrade (§19.7.4) —, on construit une instance neuve et on l'amène à l'état de la production par réplication.

La séquence d'ensemble est la suivante :

1. **Provisionner une instance 12.3 neuve**, configurée comme cible de production (en tenant compte des variables retirées, §11.2.3, et du packaging Galera séparé le cas échéant, §14.2.5).
2. **Établir une réplication 11.8 → 12.3** : la 12.3 devient un réplica de la production 11.8 et applique ses écritures en continu, jusqu'à rattraper son retard.
3. **Valider** la nouvelle instance pendant qu'elle réplique, à l'aide de la campagne de tests de la §19.6 (comparaison des résultats, des plans, de la cohérence).
4. **Basculer** (cutover) : voir §19.8.5.

Un point technique important éclaire pourquoi cette direction est sûre. La réplication s'effectue ici du **plus ancien vers le plus récent** : la 12.3, en tant que réplica, consomme le flux de binlog produit par la 11.8 (format classique). C'est la direction prise en charge et sans difficulté. La direction inverse — 12.3 → 11.8 — est celle qui pose problème (§19.7.5), car la 12.3 émet alors son nouveau binlog intégré à InnoDB (§11.5.4) et peut produire des événements que la 11.8 ne sait pas interpréter. Autrement dit, **la bascule sans interruption emprunte la direction facile ; le filet de rollback emprunte la direction fragile** — d'où l'attention particulière qu'exige ce dernier.

L'usage de GTID (§13.4) tout au long facilite le suivi des positions, la promotion et l'éventuelle mise en place de la réplication inverse pour le rollback.

---

## 19.8.5 La séquence de bascule (cutover)

La bascule est le moment critique : c'est la seule fenêtre où une interruption, même brève, peut survenir. L'objectif est de la réduire à quelques secondes en suivant une séquence rigoureuse :

1. **Réduire le retard de réplication à un niveau proche de zéro** en amont, en s'assurant que la 12.3 suit la 11.8 sans accumulation (réplication parallèle, optimistic ALTER §13.10).
2. **Figer les écritures sur l'ancienne source** en la passant en lecture seule :

   ```sql
   -- Sur l'ancienne source 11.8
   SET GLOBAL read_only = ON;
   ```

   Toute écriture privilégiée résiduelle doit également être quiescée, afin qu'aucune transaction n'échappe à la réplication.
3. **Attendre que le réplica 12.3 applique les derniers événements** et vérifier l'alignement des positions :

   ```sql
   -- Sur le réplica 12.3
   SHOW REPLICA STATUS\G   -- Seconds_Behind_Source = 0, positions/GTID alignés
   ```

   La confirmation que la 12.3 a tout rattrapé (§13.7.2) est la condition d'une bascule sans perte.
4. **Promouvoir la 12.3** comme nouvelle source : arrêter la réplication entrante et la rendre accessible en écriture.
5. **Repointer les applications** vers la 12.3 — directement ou, mieux, via la couche de routage (§19.8.6) qui rend ce changement transparent.
6. **Mettre en place la réplication inverse 12.3 → 11.8** comme filet de rollback (§19.7.5), avant de rouvrir les écritures :

   ```sql
   -- Sur l'ancienne 11.8, pour la maintenir synchronisée (fenêtre de rollback)
   CHANGE MASTER TO MASTER_HOST='nouvelle-12.3', MASTER_USE_GTID=slave_pos;
   START SLAVE;
   ```
7. **Rouvrir les écritures** sur la 12.3, désormais source de production.

La durée perçue de l'interruption se résume aux étapes 2 à 5 — quelques secondes si le retard de réplication était maîtrisé.

---

## 19.8.6 La couche de routage : réduire la fenêtre perçue à néant

Même une bascule de quelques secondes peut être rendue **imperceptible** par une couche de routage placée entre les applications et la base. C'est elle qui transforme un « repointage » manuel et visible en une transition transparente.

**MaxScale** (§14.4) joue ce rôle nativement : il achemine les requêtes (read/write split, §14.4.2) et masque l'identité physique de la source aux applications, qui se connectent toujours au même point d'entrée. Lors de la bascule, c'est MaxScale qui redirige le trafic vers la nouvelle source, sans reconfiguration côté application.

Surtout, les fonctionnalités de **Transaction Replay et Connection Migration** (§14.10) permettent aux connexions et même aux transactions en cours de **survivre au changement de backend** : une transaction interrompue par la bascule peut être rejouée transparemment sur la nouvelle source, et les sessions migrées sans rupture. C'est ce mécanisme qui rapproche le plus la migration d'un véritable « zéro interruption » du point de vue de l'utilisateur.

Les alternatives **ProxySQL** et **HAProxy** (§14.9) offrent des capacités comparables de repointage transparent et de routage, ProxySQL ajoutant un pooling de connexions et un routage par règles.

L'orchestration de la promotion elle-même peut s'appuyer sur les solutions de failover automatique (§14.6), qui automatisent la détection, la promotion et le repointage selon une procédure éprouvée.

---

## 19.8.7 Cas particulier des clusters Galera

Pour un cluster Galera (§14.2), la situation appelle une nuance importante. Une mise à niveau **de version mineure ou de correctif** peut souvent se faire en **rolling upgrade**, nœud par nœud : on retire un nœud du cluster, on le met à niveau, on le réintègre, et l'on répète — le service restant assuré par les autres nœuds.

En revanche, **faire cohabiter des versions majeures différentes au sein d'un même cluster Galera n'est pas pris en charge**. Un saut 11.8 → 12.3 ne peut donc pas se faire par simple rolling upgrade in-place du cluster. Le motif adapté consiste alors à **construire un nouveau cluster 12.3 distinct** et à le synchroniser depuis l'ancien par réplication asynchrone entre clusters, éventuellement parallélisée (§13.11), avant de basculer le trafic — c'est-à-dire une transposition, à l'échelle d'un cluster, du motif de migration par réplication décrit plus haut.

Le packaging séparé `mariadb-server-galera` introduit en 12.3 (§14.2.5) est par ailleurs à intégrer dans le provisionnement du nouveau cluster, notamment dans les images conteneurisées et les déploiements via operator (§16.5.2).

---

## 19.8.8 Le rôle de l'application et le déploiement progressif

Une migration sans interruption n'est jamais purement une affaire d'infrastructure : l'application doit y participer.

Elle doit d'abord **tolérer la brève fenêtre de bascule** — idéalement par une logique de reconnexion et de réessai des requêtes, à défaut en s'appuyant sur la couche de routage (§19.8.6) qui absorbe l'interruption. Une application qui échoue brutalement à la première erreur de connexion réintroduit une coupure visible que toute l'architecture cherchait à éviter.

Elle doit ensuite, pendant la transition, **se comporter correctement avec la version cible**. Les changements de comportement de la version cible doivent être gérés *avant* la bascule, sous peine de transformer une migration réussie au niveau base en incident applicatif. Pour un passage depuis la 11.8, l'isolation par instantané (§6.9) n'est pas un de ces changements — elle est déjà active des deux côtés (depuis la 11.6.2) ; en revanche, si l'on migre depuis un socle plus ancien ou depuis MySQL, l'application doit désormais capter les erreurs de conflit d'instantané et rejouer la transaction.

Enfin, le **déploiement progressif** (*canary*) réduit le risque : router d'abord une fraction du trafic vers la 12.3, observer son comportement réel, puis augmenter graduellement la part jusqu'à la bascule complète. Cette montée en charge progressive limite l'exposition en cas de problème non détecté par les tests, et s'articule naturellement avec le plan de contingence (§19.7).

---

## 19.8.9 Spécificités 12.3 pour les migrations sans interruption

Plusieurs caractéristiques de la 12.3 conditionnent la conduite d'une migration sans interruption :

- **Direction de réplication** (§11.5.4). La réplication 11.8 → 12.3, base de la bascule sans interruption, est la direction sûre (la 12.3 consomme un binlog classique). Le nouveau binlog intégré à InnoDB ne devient une contrainte que pour la réplication inverse servant au rollback (§19.7).
- **Réplication parallèle entre clusters Galera** (§13.11). Elle accélère la synchronisation d'un nouveau cluster 12.3 depuis un ancien cluster, rendant praticable la migration de cluster à cluster décrite en §19.8.7.
- **Optimistic ALTER TABLE** (§13.10) et **`innodb_alter_copy_bulk`** (§15.6). Ces optimisations réduisent le retard de réplication induit par les modifications de schéma et accélèrent les reconstructions, deux facteurs qui rétrécissent la fenêtre de bascule.
- **Packaging Galera séparé** (§14.2.5). À prendre en compte dans le provisionnement de la nouvelle infrastructure, en particulier conteneurisée (§16.3.1) et via operator (§16.5.2).
- **Comportement transactionnel** (§6.9). L'isolation par instantané est déjà active par défaut en 11.8 comme en 12.3 (depuis la 11.6.2) : un passage `11.8 → 12.3` n'y change rien. La vigilance s'impose surtout si la source précède la 11.6.2 ou provient de MySQL — l'application doit alors gérer les erreurs de conflit *avant* la bascule, faute de quoi la migration réussit côté base mais échoue côté service.

---

## Points clés à retenir

- « Zéro interruption » désigne en pratique une **bascule de quelques secondes rendue imperceptible**, et non l'absence physique de toute transition ; la couche de routage est ce qui en réduit la perception à néant.
- Deux problèmes distincts à résoudre : les **changements de schéma en ligne** (Online DDL natif §18.11, ou outils `gh-ost`/`pt-online-schema-change` §16.8.3) et la **bascule de version**, réunis par le proxy.
- Le motif **expand/contract** découple l'évolution de schéma du déploiement applicatif, permettant à l'ancienne et à la nouvelle version de coexister à chaque étape.
- La **migration par réplication** (11.8 → 12.3) est le motif de référence : c'est la direction *sûre* de réplication, et c'est la même architecture qui sert le rollback (§19.7).
- La **séquence de bascule** — lecture seule sur la source, attente du rattrapage, promotion, repointage — concentre toute l'interruption dans une fenêtre de quelques secondes.
- **MaxScale** avec **Transaction Replay et Connection Migration** (§14.10) rapproche le plus la migration d'une transparence totale ; ProxySQL et HAProxy (§14.9) sont des alternatives.
- Pour Galera, **les versions majeures ne cohabitent pas dans un même cluster** : un saut 11.8 → 12.3 se fait de cluster à cluster par réplication (§13.11), non par rolling upgrade in-place.
- L'**application** doit participer : réessais, tolérance à la lecture seule, gestion de l'isolation par instantané (§6.9), et déploiement progressif.

> 🔗 **Pour aller plus loin** : §19.4 *Stratégies de mise à jour et upgrade paths*, §19.6 *Tests de compatibilité* (validation avant bascule), §19.7 *Rollback et contingence* (le filet de sécurité indissociable de la bascule), §18.11 *Online Schema Change*, §16.8.3 *gh-ost et pt-online-schema-change*, chapitre 13 *Réplication* (§13.4 GTID, §13.8 failover/switchover, §13.10 Optimistic ALTER, §13.11 réplication parallèle entre clusters Galera), et chapitre 14 *Haute Disponibilité* (§14.4 MaxScale, §14.9 ProxySQL/HAProxy, §14.10 Transaction Replay/Connection Migration).

⏭️ [Migration System-Versioned Tables (format timestamp, héritage 11.8)](/19-migration-compatibilite/09-migration-system-versioned-tables.md)
