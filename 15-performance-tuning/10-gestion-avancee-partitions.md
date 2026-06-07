🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15.10 Gestion avancée des partitions

Les opérations courantes de gestion des partitions — ajout, suppression, fusion — ont été présentées au fil de la section 15.9, dans le contexte de chaque méthode de partitionnement. Cette section les complète par les opérations avancées qui permettent de piloter finement le cycle de vie d'une table partitionnée : la conversion entre une partition et une table autonome, la réorganisation et la reconstruction des partitions, la maintenance ciblée, et la suppression du partitionnement. La pièce maîtresse, à laquelle le titre fait référence, est la **conversion partition↔table**, qui sous-tend les principaux scénarios d'archivage et de chargement en masse.

## Pourquoi convertir entre partition et table ?

L'idée fondatrice est simple : une partition et une table autonome ont la **même forme physique**. Une partition de la table `ventes` contenant les données de 2023 a exactement la structure d'une table indépendante `archives_2023`. Convertir l'une en l'autre revient donc, dans le cas idéal, à une opération de métadonnées — un simple changement d'étiquette — et non à une recopie de lignes.

Cette équivalence ouvre deux scénarios majeurs. Le premier est l'**archivage** : on détache une partition ancienne pour en faire une table autonome que l'on pourra ensuite compresser, déplacer vers un stockage froid (voir le moteur S3, §7.6) ou simplement conserver à part, sans alourdir la table active. Le second est le **chargement en masse** : on prépare un lot de données dans une table séparée — où l'on peut construire les index et valider le contenu à loisir, sans toucher à la table de production — puis on la rattache comme nouvelle partition d'un seul geste, rendant les données visibles instantanément.

## EXCHANGE PARTITION : l'échange historique

La commande `ALTER TABLE … EXCHANGE PARTITION … WITH TABLE …` existe dans MariaDB depuis très longtemps. Elle **échange le contenu** d'une partition et d'une table autonome :

```sql
ALTER TABLE ventes EXCHANGE PARTITION p2023 WITH TABLE archives_2023;
```

Plusieurs conditions doivent être réunies : la table `ventes` doit être partitionnée et la table `archives_2023` ne doit pas l'être ; cette dernière ne peut pas être une table temporaire ; et les deux tables doivent être **par ailleurs identiques** en structure (mêmes colonnes, mêmes index). Toute ligne présente dans la table autonome doit en outre satisfaire la définition de la partition cible — contrainte vérifiée ligne à ligne, sauf si l'on précise l'option `WITHOUT VALIDATION`. L'échange lui-même est une opération de métadonnées, donc rapide quel que soit le volume.

Comme `EXCHANGE` échange les contenus, l'usage typique met en jeu un côté vide : on échange avec une table vide pour **extraire** une partition, ou l'on échange une table pleine avec une partition vide pour **y charger** des données. C'est d'ailleurs sur cette commande que reposait, avant la version 10.7, la seule façon de convertir une partition en table autonome — une recette en plusieurs étapes peu intuitive :

```sql
-- Recette historique pour extraire une partition en table autonome
CREATE TABLE archives_2023 LIKE ventes;        -- copie aussi le partitionnement  
ALTER TABLE archives_2023 REMOVE PARTITIONING;  -- on le retire  
ALTER TABLE ventes EXCHANGE PARTITION p2023 WITH TABLE archives_2023;  
ALTER TABLE ventes DROP PARTITION p2023;        -- la partition est désormais vide  
```

Cette séquence présente un défaut majeur : étant composée de plusieurs instructions distinctes, elle n'est **pas atomique**. Une panne ou un arrêt du serveur en cours de route peut laisser la table dans un état intermédiaire incohérent. C'est précisément ce que les opérations modernes de conversion sont venues corriger.

## CONVERT PARTITION TO TABLE et CONVERT TABLE TO PARTITION

MariaDB propose désormais deux commandes dédiées, qui réalisent la conversion en une seule instruction **atomique** : l'opération aboutit entièrement ou est intégralement annulée, même en cas de crash du serveur en cours d'exécution.

La conversion d'une partition vers une table autonome est disponible depuis **MariaDB 10.7** :

```sql
-- Extraire la partition p2023 en table autonome, en une seule instruction
ALTER TABLE ventes CONVERT PARTITION p2023 TO TABLE archives_2023;
```

La conversion inverse — rattacher une table autonome comme nouvelle partition — est disponible depuis **MariaDB 11.4** :

```sql
-- Rattacher une table autonome comme nouvelle partition
ALTER TABLE ventes
  CONVERT TABLE import_2026 TO PARTITION p2026 VALUES LESS THAN ('2027-01-01');
```

Les deux commandes sont donc présentes dans MariaDB 12.3. Lors de la conversion d'une table vers une partition, chaque ligne est **validée** pour s'assurer qu'elle respecte la définition de la partition. Cette vérification peut être lente sur de grandes tables ; on peut la désactiver avec l'option `WITHOUT VALIDATION`, l'option par défaut `WITH VALIDATION` effectuant le contrôle.

```sql
-- Sans validation ligne à ligne : plus rapide, mais sous la responsabilité de l'appelant
ALTER TABLE ventes
  CONVERT TABLE import_2026 TO PARTITION p2026 VALUES LESS THAN ('2027-01-01')
  WITHOUT VALIDATION;
```

L'usage de `WITHOUT VALIDATION` n'est judicieux que lorsque l'on a la **garantie** que toutes les lignes de la table source respectent les bornes de la partition. Une ligne hors borne insérée sans validation produirait une incohérence entre le contenu réel et la définition de la partition, susceptible de fausser l'élagage et les requêtes ultérieures.

## EXCHANGE ou CONVERT : que choisir ?

Pour une conversion pure entre partition et table, les commandes `CONVERT` constituent l'approche moderne et recommandée : elles tiennent en une seule instruction, sont atomiques et donc sûres face aux pannes, et expriment directement l'intention. La recette `EXCHANGE` en plusieurs étapes n'a plus de raison d'être pour ce cas d'usage sur une version récente.

`EXCHANGE PARTITION` conserve néanmoins son utilité propre : celle d'**échanger des contenus en place** sans détacher ni recréer de partition — par exemple pour substituer en bloc le contenu d'une partition par celui d'une table préparée à l'avance, la partition restant en place dans la table. C'est aussi la seule option disponible sur les versions antérieures à 10.7 / 11.4. Le choix se résume donc ainsi : `CONVERT` pour transformer une partition en table ou inversement, `EXCHANGE` pour permuter des contenus.

## Réorganiser et reconstruire les partitions

Au-delà de la conversion, plusieurs opérations agissent sur les partitions existantes sans les transformer en tables.

`REORGANIZE PARTITION`, déjà rencontrée aux §15.9.1 et §15.9.2, permet de **scinder ou fusionner** des partitions en redistribuant leurs lignes, tout en préservant les données. C'est l'outil de la fenêtre glissante (extraction d'une période d'une partition `MAXVALUE`) comme de la refonte d'une nomenclature `LIST`.

```sql
-- Scinder la partition fourre-tout pour ouvrir une nouvelle période
ALTER TABLE ventes REORGANIZE PARTITION pmax INTO (
    PARTITION p2026 VALUES LESS THAN ('2027-01-01'),
    PARTITION pmax  VALUES LESS THAN (MAXVALUE)
);
```

`REBUILD PARTITION` **reconstruit** une partition, ce qui défragmente ses données et ses index — l'équivalent ciblé d'un `OPTIMIZE` au niveau d'une partition. Elle est utile après de nombreuses suppressions ou mises à jour ayant fragmenté l'espace.

```sql
ALTER TABLE ventes REBUILD PARTITION p2025;
```

`TRUNCATE PARTITION`, enfin, **vide** rapidement le contenu d'une partition sans la supprimer, contrairement à `DROP PARTITION` qui retire à la fois les données et la définition de la partition.

```sql
ALTER TABLE ventes TRUNCATE PARTITION p2023;
```

On rappellera que `DROP PARTITION` ne s'applique qu'aux partitionnements `RANGE` et `LIST` ; sur `HASH` et `KEY`, c'est `COALESCE PARTITION` qui réduit le nombre de partitions (§15.9.3).

## Maintenance ciblée par partition

Les commandes de maintenance habituelles peuvent être restreintes à une ou plusieurs partitions, ce qui réduit considérablement la durée et l'impact des opérations sur les très grandes tables. On indique une liste de partitions, ou le mot-clé `ALL` pour les traiter toutes.

```sql
ALTER TABLE ventes ANALYZE PARTITION p2025, p2026;  
ALTER TABLE ventes OPTIMIZE PARTITION p2025;  
ALTER TABLE ventes CHECK PARTITION ALL;  
```

Cibler la seule partition active plutôt que la table entière permet, par exemple, de recalculer les statistiques de la période courante sans parcourir des années d'historique — un gain net en fenêtre de maintenance.

## Supprimer le partitionnement

Lorsqu'un partitionnement n'apporte plus de bénéfice, ou que l'on souhaite revenir à une table simple, `ALTER TABLE … REMOVE PARTITIONING` retire tout le partitionnement en **fusionnant l'ensemble des partitions** en une seule table non partitionnée, sans affecter les données.

```sql
ALTER TABLE ventes REMOVE PARTITIONING;
```

À la différence de `DROP PARTITION`, qui détruit les données de la partition visée, `REMOVE PARTITIONING` conserve l'intégralité des lignes : seule la structure de partitionnement disparaît.

## Opérations en ligne et impact sur la production

Le caractère bloquant de ces opérations varie fortement, et c'est un facteur décisif en production. `EXCHANGE PARTITION` et les commandes `CONVERT` s'apparentent à des opérations de métadonnées et restent rapides ; la conversion `CONVERT TABLE … TO PARTITION` avec validation fait toutefois exception, puisqu'elle lit chaque ligne pour la contrôler. À l'inverse, `REORGANIZE` et `REBUILD` recopient physiquement les données concernées et peuvent donc être lourdes sur de gros volumes.

Un point important doit être souligné : en MariaDB, les **opérations spécifiques aux partitions** (`ADD`, `DROP`, `REBUILD`, `REORGANIZE`, `COALESCE`… `PARTITION`) **ne prennent pas en charge la clause `ALGORITHM`/`LOCK`**, et donc pas le mode `ALTER ONLINE TABLE` (qui équivaut à `LOCK = NONE`). Une tentative en ce sens est rejetée :

```sql
-- ❌ Échoue : les opérations de partition n'acceptent pas LOCK=NONE
ALTER ONLINE TABLE ventes REBUILD PARTITION p2025;
-- ERROR 1846 (0A000): LOCK=NONE/SHARED/EXCLUSIVE is not supported.
-- Reason: Partition specific operations do not yet support LOCK/ALGORITHM. Try LOCK=DEFAULT
```

Ces opérations s'exécutent donc avec le verrouillage par défaut (`LOCK = DEFAULT`). Leur impact reste néanmoins variable : celles qui relèvent des métadonnées (`DROP`/`TRUNCATE PARTITION`, quasi instantanées ; `EXCHANGE`/`CONVERT`) bloquent peu, tandis que celles qui recopient des données (`REBUILD`, `REORGANIZE`) doivent être réservées à une fenêtre de maintenance. Le mode en ligne demeure en revanche disponible pour les modifications de schéma **non spécifiques aux partitions** d'une table partitionnée — ajout de colonne ou d'index, par exemple :

```sql
-- ✅ Fonctionne : modification non spécifique aux partitions, en ligne
ALTER ONLINE TABLE ventes ADD COLUMN remise DECIMAL(5,2);
```

Le choix de l'approche conditionne donc l'impact : privilégier les conversions atomiques et la maintenance ciblée, et réserver les réorganisations et reconstructions lourdes aux fenêtres adaptées. Ces considérations rejoignent celles du changement de schéma non bloquant, traité au §18.11.

## Automatisation du cycle de vie

La plupart de ces opérations gagnent à être automatisées plutôt qu'exécutées à la main. MariaDB ne fournit pas de mécanisme intégré pour, par exemple, « ajouter une partition pour le mois prochain » ou « archiver les partitions de plus d'un an », mais la combinaison d'une procédure stockée et d'un événement planifié (`EVENT`, §8.4) permet de mettre en place une maintenance régulière : création anticipée des partitions à venir, conversion des partitions expirées en tables d'archives, recalcul des statistiques de la partition courante. C'est la voie naturelle pour industrialiser la fenêtre glissante d'une table chronologique.

## En conclusion

La gestion avancée des partitions transforme le partitionnement d'un choix de conception figé en un véritable cycle de vie maîtrisable : on extrait, on rattache, on réorganise et on entretient les partitions au gré des besoins, avec des opérations désormais atomiques et, pour beaucoup, exécutables en ligne. Cette maîtrise reste néanmoins cantonnée à un serveur unique. Lorsque la volumétrie ou la charge dépassent ce qu'un seul nœud peut absorber, il faut distribuer les données entre plusieurs serveurs — c'est l'objet du §15.11 (Sharding et distribution horizontale), qui prolonge la logique du partitionnement à l'échelle d'une infrastructure.

⏭️ [Sharding et distribution horizontale](/15-performance-tuning/11-sharding-distribution.md)
