🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.9 Migration System-Versioned Tables (format timestamp, héritage 11.8)

Les tables à versionnement système (*system-versioned tables*, §18.2) conservent automatiquement l'historique de toutes les modifications de leurs lignes. Elles reposent sur un détail d'implémentation discret mais crucial pour la migration : un **marqueur de « ligne active »** stocké dans la colonne `row_end`. La valeur de ce marqueur a changé avec l'extension de la plage des `TIMESTAMP` introduite en 11.8 (§11.12), et ce changement a des conséquences concrètes — parfois coûteuses — lors d'une migration.

Cette section explique ce qu'est ce marqueur, pourquoi sa valeur a évolué, ce que la migration implique réellement, et surtout *quel chemin de migration est concerné* — car la réponse n'est pas la même selon que l'on vient de la 11.8 ou d'une version antérieure. C'est précisément le sens de la mention « héritage 11.8 » : le comportement n'est pas nouveau en 12.3, il en est hérité, ce qui change radicalement la nature de la préoccupation selon le point de départ.

---

## 19.9.1 Rappel : le marqueur de ligne active des tables temporelles

Une table à versionnement système possède deux colonnes de période, `row_start` et `row_end`, qui délimitent l'intervalle de validité de chaque version d'une ligne (§18.2.1). Lorsqu'une ligne est insérée, elle reçoit un `row_start` égal à l'instant courant et un `row_end` fixé à une valeur spéciale représentant **« encore en vigueur »** — un instant volontairement situé dans un futur très lointain, qui sert de marqueur d'infini.

```sql
SELECT a, row_start, row_end FROM employes;
-- Une ligne dont row_end vaut le marqueur d'infini est la version ACTIVE.
-- Une ligne dont row_end vaut un instant passé est une version HISTORIQUE.
```

Quand la ligne est modifiée, sa version courante est déplacée vers l'historique avec un `row_end` fixé à l'instant de la modification, et une nouvelle version active est créée avec, de nouveau, le marqueur d'infini. C'est la comparaison de `row_end` à ce marqueur qui distingue, dans toute requête (`FOR SYSTEM_TIME AS OF`, sélection des lignes courantes), ce qui est actif de ce qui est historique.

La valeur exacte de ce marqueur est le **maximum représentable** par un `TIMESTAMP` — et c'est là que réside l'enjeu de migration.

---

## 19.9.2 Le changement de marqueur : de 2038 à 2106

Historiquement, le type `TIMESTAMP` de MariaDB était limité à une valeur maximale de `2038-01-19 03:14:07 UTC` — la fameuse limite de l'« an 2038 » (Y2038), héritée du codage sur 32 bits signés des horodatages Unix. Le marqueur d'infini des tables à versionnement système valait donc :

```
2038-01-19 03:14:07.999999 UTC   (= FROM_UNIXTIME(2147483647.999999))
```

À partir de la 11.8, MariaDB a **étendu la plage des `TIMESTAMP` jusqu'à 2106** (§11.12) en interprétant le champ de 32 bits comme non signé plutôt que signé. La nouvelle valeur maximale devient `2106-02-07 06:28:15 UTC`, et le marqueur d'infini des tables temporelles devient en conséquence :

```
2106-02-07 06:28:15.999999 UTC   (= FROM_UNIXTIME(4294967295.999999))
```

> 📌 Les valeurs affichées peuvent différer de ces valeurs UTC selon le fuseau horaire de la session ; la référence canonique reste exprimée en UTC.

Trois propriétés importantes accompagnent ce changement :

- Il **ne modifie pas le format de stockage** physique : le champ reste codé sur 32 bits, désormais lus comme non signés. Une table créée avec la nouvelle plage reste lisible par un ancien serveur *tant que ses valeurs restent dans l'ancienne plage* — ce qui, pour une ligne active, n'est justement pas le cas (son `row_end` vaut 2106). Ce point est déterminant pour le rollback (§19.9.6).
- Il n'est pris en charge que sur les **plateformes 64 bits**.
- Il impose que, dans les tables à versionnement système, le `row_end` des lignes passe d'un marqueur **basé sur 2038** à un marqueur **basé sur 2106**.

---

## 19.9.3 Ce que la migration implique : la réécriture des tables versionnées

C'est ici que la migration cesse d'être anodine. Faire passer le marqueur d'infini de 2038 à 2106 ne se fait pas d'un simple changement de métadonnées : **toutes les lignes — et tous les index — des tables à versionnement système doivent être réécrites** pour adopter la nouvelle plage.

L'impact est purement volumétrique, mais il peut être considérable. Pour une table temporelle comportant un historique de plusieurs millions, voire dizaines de millions de lignes, cette réécriture peut **prendre un temps significatif** et constituer, à elle seule, le poste le plus long de toute l'opération de migration. À l'inverse, une base *sans* table à versionnement système n'est pas concernée et migre en quelques secondes pour cette partie.

Le point opérationnel à retenir est donc clair : **le coût de cette étape est proportionnel au volume d'historique conservé**, et non au nombre de tables ni à la complexité du schéma. Une estimation de durée doit être établie en fonction de la taille des partitions historiques (§18.2.3) avant de planifier la fenêtre de migration.

---

## 19.9.4 Héritage 11.8 : quel chemin de migration est concerné ?

C'est la question décisive, et la réponse dépend entièrement du point de départ :

| Point de départ | Conversion 2038 → 2106 déclenchée ? | Coût |
|-----------------|--------------------------------------|------|
| **11.8 → 12.3** | **Non** — déjà effectuée lors du passage à 11.8 | Nul pour ce point |
| **Pré-11.8 (10.x, 11.4) → 12.3** | **Oui** — réécriture intégrale des tables versionnées | Proportionnel au volume d'historique |

La 11.8 ayant déjà introduit la plage étendue, **les tables à versionnement système d'une instance 11.8 sont déjà au format 2106**. Le passage de 11.8 à 12.3 — qui est le scénario de migration central de cette formation (§19.10) — **ne redéclenche donc pas cette conversion** : pour ce point précis, la migration 11.8 → 12.3 est gratuite.

En revanche, une migration *directe* depuis une version antérieure à 11.8 (10.6, 10.11, 11.4…) vers la 12.3 effectuera la conversion lors de la mise à niveau, avec le coût volumétrique décrit ci-dessus. C'est exactement ce que signifie « héritage 11.8 » : le comportement a été introduit en 11.8 et la 12.3 en hérite tel quel. Comprendre cette distinction évite deux erreurs symétriques — surestimer le coût d'une migration 11.8 → 12.3 (en croyant la conversion nécessaire), ou le sous-estimer lors d'une migration depuis une version plus ancienne (en l'ignorant).

---

## 19.9.5 Planifier la conversion des tables volumineuses

Lorsque le chemin de migration *déclenche* la conversion (départ pré-11.8), plusieurs leviers permettent d'en maîtriser le coût.

**Purger l'historique inutile en amont.** La quantité de lignes à réécrire étant le facteur déterminant, réduire le volume d'historique *avant* la migration raccourcit d'autant l'opération. L'instruction `DELETE HISTORY` supprime les versions historiques antérieures à une date donnée :

```sql
-- Purger l'historique antérieur à une date, AVANT la migration
DELETE HISTORY FROM commandes BEFORE SYSTEM_TIME '2024-01-01 00:00:00';
```

> ⚠️ La date passée à `BEFORE SYSTEM_TIME` doit être franchement **dans le passé**. Une date postérieure au `row_end` des lignes actives entraînerait la suppression et le basculement en historique des lignes courantes elles-mêmes — un effet manifestement indésirable. On purge l'historique ancien, jamais le présent.

**Déporter la conversion hors du chemin critique.** La migration par réplication (§19.8.4) est ici particulièrement avantageuse : la conversion des tables temporelles s'effectue sur la nouvelle instance 12.3 *pendant qu'elle rattrape la production par réplication*, sans interrompre le service assuré par l'ancienne instance. Le long temps de réécriture est ainsi absorbé en arrière-plan, et n'entre pas dans la fenêtre de bascule (§19.8.5).

**Dimensionner la fenêtre pour un upgrade in-place.** Si la migration se fait sur place, la réécriture s'exécute durant la mise à niveau (typiquement via `mariadb-upgrade`, §19.4.1) et doit être intégrée à la durée de la fenêtre de maintenance. D'où l'importance d'une estimation préalable fondée sur le volume d'historique.

---

## 19.9.6 Impact sur le rollback et le downgrade

Le changement de marqueur a une conséquence directe sur la réversibilité, qui prolonge la discussion de la §19.7.

Une fois converties au format 2106, les tables à versionnement système contiennent, pour leurs lignes actives, des valeurs de `row_end` **situées hors de l'ancienne plage des `TIMESTAMP`** (au-delà de 2038). Or un serveur antérieur à 11.8 ne sait pas interpréter ces valeurs. C'est pourquoi, dès qu'une base comporte des tables à versionnement système, **le downgrade cesse d'être trivial** : on ne peut pas simplement redescendre l'instance vers une version qui ignore la plage étendue, sous peine de lire incorrectement les lignes actives.

Deux conséquences pratiques en découlent :

- **Pour le scénario 11.8 ↔ 12.3** (la migration centrale de la formation), ce point précis *n'aggrave pas* le rollback : les deux versions utilisent déjà le format 2106, donc le marqueur d'infini n'est pas un obstacle au retour vers 11.8. Les difficultés de rollback entre ces deux versions tiennent à d'autres facteurs (nouveau binlog InnoDB §11.5.4 s'il est activé, portée des noms de contraintes FK §18.12), traités en §19.7 — l'isolation par instantané, elle, étant identique des deux côtés, n'en fait pas partie.
- **Pour un retour vers une version pré-11.8**, en revanche, le marqueur 2106 est un blocage réel. La voie de rollback sûre redevient la **restauration de la sauvegarde antérieure** à la migration (§19.7.3) ; la réplication inverse est ici d'autant plus fragile que la version cible ne comprend pas le format des données qu'on chercherait à lui envoyer.

La règle générale énoncée en §19.7 — *restaurer une sauvegarde plutôt que tenter un downgrade* — s'applique donc avec une acuité particulière aux bases comportant des tables temporelles.

---

## 19.9.7 Vérifications et cas limites

**Identifier les tables concernées.** Avant toute migration, il faut recenser les tables à versionnement système, par exemple via `SHOW CREATE TABLE` (la clause `WITH SYSTEM VERSIONING` y figure) ou par interrogation d'`INFORMATION_SCHEMA` (§9.7.1). Ce sont les seules tables pour lesquelles la conversion 2038 → 2106 entre en jeu.

**Vérifier le format après migration.** Une fois la conversion effectuée, une vérification simple confirme que les lignes actives portent bien le nouveau marqueur :

```sql
-- Le maximum de row_end doit refléter 2106, et non plus 2038
SELECT MAX(row_end) FROM employes;
```

**Plateformes 32 bits.** L'extension n'étant prise en charge que sur 64 bits, une migration vers une plateforme 32 bits ne bénéficierait pas de la plage étendue — configuration aujourd'hui marginale, mais à exclure explicitement lors de l'inventaire matériel.

**Chemins de mise à niveau par dump/reload.** Les montées de version par sauvegarde logique et rechargement des tables à versionnement système ont historiquement présenté des cas limites (échecs de conversion sur certains chemins). La validation post-migration (§19.6) doit donc inclure une vérification explicite de l'intégrité et du format de ces tables, et ne pas la présumer acquise.

**Interaction de `DELETE HISTORY` avec le marqueur.** Comme rappelé en §19.9.5, le comportement de `DELETE HISTORY BEFORE SYSTEM_TIME` dépend de la position de la date fournie par rapport au `row_end` des lignes actives ; cette subtilité doit être connue de toute personne effectuant la maintenance de l'historique avant ou après migration.

---

## Points clés à retenir

- Les tables à versionnement système marquent leurs lignes actives par un `row_end` égal au **maximum représentable d'un `TIMESTAMP`**, qui sert de marqueur d'« infini ».
- L'extension de la plage `TIMESTAMP` (11.8, §11.12) fait passer ce maximum de **2038-01-19** à **2106-02-07**, sans changer le format de stockage, et **uniquement sur les plateformes 64 bits**.
- La conversion impose de **réécrire toutes les lignes et index** des tables versionnées : son coût est proportionnel au **volume d'historique** et peut être long ; les bases sans table temporelle ne sont pas concernées.
- **« Héritage 11.8 » signifie que la conversion n'est pas redéclenchée par une migration 11.8 → 12.3** (déjà faite) ; elle ne se produit que lors d'une migration *directe depuis une version antérieure à 11.8*.
- Pour maîtriser le coût : **purger l'historique inutile** avant migration et, si possible, **déporter la conversion** sur l'instance cible via une migration par réplication (§19.8.4).
- Côté rollback : le format 2106 rend le **downgrade non trivial** vers les versions pré-11.8 (valeurs hors de l'ancienne plage), ce qui renforce la règle « restaurer une sauvegarde plutôt que downgrader » (§19.7) ; entre 11.8 et 12.3, ce point n'est en revanche pas un obstacle.

> 🔗 **Pour aller plus loin** : §18.2 *Tables temporelles (System-Versioned Tables)* (fonctionnement détaillé, requêtes temporelles, partitionnement de l'historique), §11.12 *Extension TIMESTAMP 2038→2106*, §19.4 *Stratégies de mise à jour* (in-place vs logique), §19.6 *Tests de compatibilité* (validation post-migration), §19.7 *Rollback et contingence*, §19.8 *Zero-downtime migrations* (déport de la conversion par réplication), et §19.10 *Migration 11.8 → 12.3* pour la synthèse du chemin de migration central.

⏭️ [Migration 11.8 → 12.3 : changements de comportement (variables retirées, scope des noms de contraintes FK, packaging Galera, binlog InnoDB optionnel)](/19-migration-compatibilite/10-migration-11-8-vers-12-3.md)
