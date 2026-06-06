🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.2 Sauvegarde logique

> **Chapitre 12 — Sauvegarde et Restauration** · MariaDB 12.3 LTS

---

## Introduction

Une **sauvegarde logique** consiste à exporter le contenu d'une base de données non pas sous la forme des fichiers bruts du moteur de stockage, mais sous la forme des **instructions SQL** permettant de la reconstruire à l'identique. Au lieu de copier des octets sur disque, on produit une suite de commandes — créations de tables, insertions de lignes, définitions de vues, de procédures, de déclencheurs — qui, rejouées sur un serveur, recréent fidèlement la structure et les données d'origine.

Cette approche s'oppose à la sauvegarde *physique* (traitée en [section 12.3](03-sauvegarde-physique-mariabackup.md)), qui copie directement les fichiers de données du moteur. La différence est fondamentale et conditionne tous les compromis de la sauvegarde logique : parce qu'elle exprime les données dans un format indépendant de la représentation interne du moteur, elle offre une **portabilité** remarquable, mais au prix de performances moindres sur les gros volumes. C'est cette tension — souplesse contre rapidité — qui définit le domaine d'emploi privilégié de la sauvegarde logique.

Cette section présente le principe général, les avantages et les limites de l'approche logique, ainsi que la question centrale de la cohérence d'une sauvegarde réalisée sur une base en activité. Les outils qui la mettent en œuvre — `mariadb-dump`, ses options essentielles, et la solution parallélisée `mydumper`/`myloader` — sont détaillés dans les sous-sections suivantes.

---

## Principe de fonctionnement

Une sauvegarde logique procède par **lecture** des données et **sérialisation** sous forme de texte SQL. L'outil parcourt les objets de la base (tables, vues, routines…), interroge leur définition et leur contenu, puis génère un script reconstituant l'ensemble. Le résultat est un fichier — le plus souvent un unique fichier `.sql`, éventuellement compressé — qui ressemble, dans sa forme la plus simple, à ceci :

```sql
CREATE TABLE `clients` (
  `id` INT NOT NULL AUTO_INCREMENT,
  `nom` VARCHAR(100) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

INSERT INTO `clients` (`id`, `nom`) VALUES
  (1, 'Dupont'),
  (2, 'Martin');
```

La **restauration** consiste simplement à exécuter ce script sur un serveur cible : les instructions `CREATE` reconstruisent le schéma, puis les instructions `INSERT` rechargent les données, et le moteur reconstruit les index au fur et à mesure. Cette symétrie — sauvegarder, c'est *écrire* du SQL ; restaurer, c'est *exécuter* du SQL — explique à la fois la grande souplesse de l'approche et sa lenteur relative : la restauration ne se contente pas de remettre des fichiers en place, elle rejoue l'intégralité des opérations d'écriture et de construction d'index.

---

## Sauvegarde logique vs sauvegarde physique

Le choix entre approche logique et approche physique est l'un des arbitrages structurants de toute stratégie de sauvegarde. Le tableau ci-dessous en résume les différences essentielles.

| Critère | Sauvegarde logique | Sauvegarde physique |
|---------|--------------------|---------------------|
| Format produit | Instructions SQL (texte) | Copie des fichiers du moteur (binaire) |
| Portabilité | Élevée : entre versions, plateformes, voire moteurs | Limitée : même version, architecture et moteur |
| Lisibilité | Lisible et modifiable | Opaque (format interne) |
| Granularité | Fine : une table, une base, une sélection | Globale : instance ou tablespace |
| Vitesse de sauvegarde | Plus lente (parcours + sérialisation) | Plus rapide (copie de fichiers) |
| Vitesse de restauration | Lente (réexécution + reconstruction des index) | Rapide (remise en place des fichiers) |
| Taille de la sauvegarde | Volumineuse, mais très compressible | Proche de la taille réelle des données |
| Domaine privilégié | Petites et moyennes bases, migrations | Grandes bases, restauration rapide |
| Outils MariaDB | `mariadb-dump`, `mydumper`/`myloader` | Mariabackup ([12.3](03-sauvegarde-physique-mariabackup.md)) |

Les deux approches ne sont pas exclusives : de nombreux dispositifs de production les **combinent**, en utilisant la sauvegarde physique pour les restaurations complètes rapides et la sauvegarde logique pour la portabilité, les migrations ou la restauration sélective.

---

## Avantages de la sauvegarde logique

Le premier atout, et le plus déterminant, est la **portabilité**. Un export SQL ne dépend pas de la représentation binaire interne du moteur : il peut être restauré sur une version différente de MariaDB, sur une autre architecture matérielle, ou servir de support à une migration depuis MySQL ou vers un autre environnement. C'est ce qui fait de la sauvegarde logique l'outil de référence pour les **montées de version** et les **migrations**.

Vient ensuite la **lisibilité**. Le fichier produit étant du texte SQL, il peut être inspecté, comparé (par exemple dans un gestionnaire de versions pour suivre l'évolution d'un schéma), voire modifié avant restauration — pour adapter un nom de base, corriger une définition ou filtrer certaines données.

La **granularité** constitue le troisième avantage majeur : on peut sauvegarder ou restaurer sélectivement une seule table, une seule base, ou uniquement la structure sans les données. Cette finesse est précieuse pour restaurer un objet isolé qui aurait été supprimé par erreur, sans avoir à remettre en service l'instance entière.

Enfin, l'approche est **indépendante du moteur de stockage** : le même mécanisme exporte indifféremment des tables InnoDB, Aria ou autres, et facilite même la conversion d'un moteur à l'autre lors de la restauration.

---

## Limites et inconvénients

Ces qualités ont une contrepartie, concentrée sur les **performances** et la **volumétrie**. Sur une base de grande taille, la sauvegarde logique est sensiblement plus lente que la sauvegarde physique : elle suppose un parcours complet des données puis leur conversion en texte, ce qui mobilise du temps processeur et sollicite le serveur. La **restauration** est plus pénalisante encore, car elle réexécute toutes les insertions et reconstruit l'ensemble des index — une opération qui peut prendre, sur de très gros volumes, beaucoup plus de temps que la sauvegarde elle-même.

La **taille** du fichier produit est également plus importante que celle des données réelles, la représentation textuelle étant moins compacte que le format binaire ; cet inconvénient est toutefois largement atténué par la compression, à laquelle le texte SQL se prête très bien.

Pour ces raisons, la sauvegarde logique reste idéale pour les bases de taille **petite à moyenne**, mais montre ses limites au-delà : sur les bases volumineuses, on lui préfère généralement une approche physique, quitte à recourir à la parallélisation (`mydumper`/`myloader`, [12.2.3](02.3-mydumper-myloader.md)) pour repousser ce seuil.

---

## La question de la cohérence

Un point conceptuel mérite une attention particulière, car il conditionne la fiabilité de toute sauvegarde logique : comment obtenir une copie **cohérente** d'une base qui continue d'être modifiée pendant l'export ? Si l'on se contente de parcourir les tables les unes après les autres, des écritures peuvent survenir entre le début et la fin du parcours, produisant une sauvegarde dans un état incohérent — par exemple une commande enregistrée sans le client correspondant.

Deux mécanismes répondent à ce besoin selon le moteur de stockage. Pour les moteurs **transactionnels** comme **InnoDB**, on réalise l'export à l'intérieur d'une **transaction unique** en isolation `REPEATABLE READ` : grâce au MVCC, l'outil voit un instantané cohérent des données figé au démarrage de la transaction, et ce **sans bloquer** les écritures concurrentes. Pour les moteurs **non transactionnels**, en revanche, la cohérence ne peut être garantie qu'en **verrouillant** temporairement les tables le temps de la sauvegarde, ce qui suspend les écritures.

Cette distinction explique l'importance des options de cohérence (au premier rang desquelles `--single-transaction`), dont la mise en œuvre concrète est détaillée dans la sous-section [12.2.2 — Options essentielles](02.2-options-essentielles.md).

---

## Cas d'usage privilégiés

La sauvegarde logique s'impose naturellement dans plusieurs situations : les **migrations** et **montées de version**, où sa portabilité est irremplaçable ; la **restauration sélective** d'un objet précis (une table, une base) sans toucher au reste de l'instance ; le **versionnage de schéma** et l'export de structure seule ; le transfert de données entre environnements (production, recette, développement) ; et plus généralement la sauvegarde des bases de taille modérée pour lesquelles ses limites de performance ne se font pas sentir.

---

## Les outils de MariaDB pour la sauvegarde logique

MariaDB propose deux familles d'outils pour réaliser des sauvegardes logiques, présentés dans les sous-sections suivantes :

- [**12.2.1 — `mysqldump` / `mariadb-dump`**](02.1-mysqldump-mariadb-dump.md) : l'outil standard, livré avec MariaDB, qui produit un export SQL classique (avec notamment la prise en charge des *wildcards* via l'option `-L`).
- [**12.2.2 — Options essentielles**](02.2-options-essentielles.md) : les paramètres incontournables d'une sauvegarde fiable (`--single-transaction`, `--routines`, et autres) pour garantir cohérence et complétude.
- [**12.2.3 — `mydumper` / `myloader`**](02.3-mydumper-myloader.md) : une solution tierce exploitant le **parallélisme** pour accélérer sensiblement la sauvegarde et la restauration logiques sur les bases plus volumineuses.

---

## À retenir

- Une sauvegarde logique exporte la base sous forme d'**instructions SQL** (schéma + données), à l'inverse de la sauvegarde physique qui copie les fichiers du moteur.
- Ses points forts sont la **portabilité** (migrations, changements de version), la **lisibilité** et la **granularité** (restauration sélective).
- Ses limites sont les **performances** et la **taille** : elle convient surtout aux bases de taille petite à moyenne ; la restauration, qui réexécute le SQL et reconstruit les index, y est particulièrement lente.
- La **cohérence** d'une sauvegarde sur base active s'obtient via une transaction unique en `REPEATABLE READ` pour InnoDB (sans blocage), ou via un verrouillage pour les moteurs non transactionnels.
- MariaDB met à disposition `mariadb-dump` (standard) et `mydumper`/`myloader` (parallélisé), détaillés dans les sous-sections suivantes.

---

⏮️ [12.1 — Stratégies de sauvegarde](01-strategies-sauvegarde.md) · ⏭️ [12.2.1 — mysqldump / mariadb-dump](02.1-mysqldump-mariadb-dump.md)

⏭️ [mysqldump / mariadb-dump (support des wildcards -L)](/12-sauvegarde-restauration/02.1-mysqldump-mariadb-dump.md)
