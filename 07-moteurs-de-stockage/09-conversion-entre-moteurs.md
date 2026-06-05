🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.9 Conversion entre moteurs (`ALTER TABLE … ENGINE`)

> **Chapitre 7 — Moteurs de Stockage** · MariaDB 12.3 LTS

Le choix d'un moteur n'est pas définitif : on peut convertir une table d'un moteur à un autre. Cette opération est simple à écrire, mais ses conséquences — durée, blocage, perte de certaines garanties — méritent d'être comprises avant de la lancer en production.

## Changer de moteur : la syntaxe

La conversion se fait avec `ALTER TABLE … ENGINE = …` :

```sql
-- Convertir une table vers InnoDB
ALTER TABLE ma_table ENGINE = InnoDB;
```

Pour connaître le moteur actuel d'une table :

```sql
SHOW TABLE STATUS LIKE 'ma_table'\G        -- colonne « Engine »

-- ou, pour toute une base
SELECT TABLE_NAME, ENGINE
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'ma_base';
```

Pour que les **nouvelles** tables utilisent par défaut un moteur donné, on règle `default_storage_engine` (§7.1 et §7.2.4) plutôt que de convertir après coup.

## Ce qui se passe réellement : une reconstruction complète

Changer le moteur d'une table n'est jamais une opération « instantanée ». MariaDB doit créer une **nouvelle table dans le moteur cible** et y **recopier toutes les lignes** : c'est une opération **par copie** (`ALGORITHM=COPY`), car un changement de moteur ne peut pas se faire « sur place ». Il faut en tirer trois conséquences pratiques :

- **Durée** : elle est proportionnelle à la taille de la table et peut être **longue** sur de gros volumes.
- **Disponibilité** : pendant la copie, la table reste généralement **accessible en lecture, mais les écritures sont bloquées**. Sur une table active en production, l'impact peut être significatif.
- **Espace disque** : la copie nécessite de l'espace supplémentaire, de l'ordre de la **taille de la table**, le temps de l'opération.

À noter un usage voisin mais distinct : `ALTER TABLE t ENGINE = InnoDB` appliqué à une table **déjà InnoDB** ne change pas de moteur, mais **reconstruit** la table (défragmentation, mise à jour du format) — c'est d'ailleurs ce que fait `OPTIMIZE TABLE` sur InnoDB (§11.6.1).

## Convertir sans bloquer la production

Sur une grosse table active, une copie bloquante en écriture n'est pas acceptable. Deux approches permettent de limiter l'impact :

- les mécanismes de **modification de schéma en ligne** de MariaDB (*Online Schema Change*, §18.11), étant entendu qu'un changement de moteur reste une copie ;
- des outils externes comme **`pt-online-schema-change`** ou **`gh-ost`** (§16.8.3), qui construisent une copie « fantôme » de la table, y rejouent les modifications survenant pendant l'opération, puis basculent atomiquement — le tout avec un verrouillage minimal.

```bash
# Conversion en ligne d'une grosse table avec pt-online-schema-change
pt-online-schema-change --alter "ENGINE=InnoDB" --execute \
  D=ma_base,t=grosse_table
```

Dans tous les cas : **sauvegardez d'abord**, **testez sur une copie**, et préférez les **périodes de faible activité**.

## Ce qui change selon la direction de conversion

La conversion ne se contente pas de déplacer des données : selon le moteur d'arrivée, certaines garanties apparaissent ou disparaissent.

### Vers InnoDB (le cas le plus fréquent)

C'est la conversion la plus courante (migration d'anciennes tables MyISAM, par exemple). On **gagne** les transactions, l'**application** des clés étrangères, le verrouillage au niveau ligne et le MVCC. Deux points d'attention :

- **définissez une bonne clé primaire** : InnoDB organise la table par index *clustered* (§7.2), et une table sans clé primaire en reçoit une, cachée. Une clé compacte et croissante est préférable.
- les **clés étrangères** ignorées par MyISAM seront désormais **réellement vérifiées** : assurez-vous que les données sont cohérentes au préalable. Pour convertir un ensemble de tables liées par des FK, on peut suspendre temporairement les vérifications :

  ```sql
  SET foreign_key_checks = 0;
  -- ALTER TABLE … ENGINE = InnoDB;  (pour chaque table de l'ensemble)
  SET foreign_key_checks = 1;
  ```

### Depuis InnoDB vers un moteur non transactionnel (Aria, MyISAM)

On **perd** les transactions, le MVCC et — surtout — les **clés étrangères** : les moteurs non transactionnels ne les prennent pas en charge. Mais MariaDB ne les abandonne pas silencieusement pour autant. Tant qu'une table participe à une relation de clé étrangère — qu'elle soit du côté **enfant** ou du côté **parent** —, la conversion est **refusée** avec l'erreur `1217 — Cannot delete or update a parent row`, et `foreign_key_checks = 0` **ne lève pas** ce blocage. Il faut donc **supprimer explicitement les contraintes** au préalable :

```sql
ALTER TABLE enfant DROP FOREIGN KEY fk_pe;   -- retirer chaque FK concernée
ALTER TABLE enfant ENGINE = Aria;            -- la conversion passe alors
```

Et si l'on a vraiment besoin d'un moteur non transactionnel, on choisit **Aria** plutôt que MyISAM, puisqu'il est *crash-safe* (§7.4).

### Vers Memory

Les données doivent **tenir en RAM**, et le moteur Memory **ne prend pas en charge les colonnes `BLOB`/`TEXT`** : la conversion **échoue** si la table en contient. Rappelons aussi que les données sont **volatiles** (perdues au redémarrage). À réserver à des données temporaires (§7.10.1).

### Vers Archive

Après conversion, seules les opérations `INSERT` et `SELECT` sont possibles (ni `UPDATE` ni `DELETE`), avec une indexation très limitée. C'est le bon choix pour des **archives en ajout seul**, fortement compressées (§7.10.2).

### Vers ColumnStore

On perd index, clés étrangères et transactions (aucun index n'est d'ailleurs nécessaire). Pour de **gros volumes**, la voie efficace n'est pas une copie `ALTER` ligne par ligne mais le **chargement en masse** (`cpimport`, §7.5).

### Vers S3

La table devient **en lecture seule** : seul le fichier `.frm` reste en local, les données partant sur le stockage objet (§7.6). L'opération est **réversible** en reconvertissant la table vers InnoDB ou Aria.

## Convertir plusieurs tables en masse

Pour convertir de nombreuses tables (par exemple toutes les tables MyISAM héritées d'une base vers InnoDB), on **génère** les instructions `ALTER` à partir d'`INFORMATION_SCHEMA`, puis on les exécute :

```sql
SELECT CONCAT('ALTER TABLE `', TABLE_SCHEMA, '`.`', TABLE_NAME,
              '` ENGINE=InnoDB;') AS ddl
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'ma_base'
  AND ENGINE = 'MyISAM';
```

On récupère la colonne `ddl` produite, on la relit comme un script, et l'on traite ainsi les tables une à une (idéalement avec les précautions ci-dessus pour les grosses tables).

## Précautions et bonne démarche

- **Sauvegarder** avant toute conversion d'envergure.
- **Vérifier les dépendances de clés étrangères**, dans les deux sens (tables qui référencent et tables référencées).
- **Tester sur une copie**, estimer la **durée** et l'**espace disque** nécessaires.
- Opérer **hors des heures de pointe**, ou via un outil de **modification en ligne** pour les grosses tables.
- **Vérifier après coup** (`SHOW TABLE STATUS`) que le moteur, les index et les contraintes sont conformes aux attentes.

## Liens avec d'autres chapitres

- L'**architecture enfichable**, le **mélange de moteurs** et `default_storage_engine` sont en §7.1, la **configuration** InnoDB en §7.2.4.
- La **modification de schéma en ligne** est traitée en §18.11, et les outils **`gh-ost`/`pt-online-schema-change`** en §16.8.3.
- La reconstruction/optimisation des tables (`OPTIMIZE TABLE`) figure en §11.6.1.
- Le choix du moteur cible s'appuie sur la **comparaison** du §7.8 et les sections de chaque moteur (§7.2 à §7.7, §7.10).

## Ce qu'il faut retenir

- La conversion se fait avec **`ALTER TABLE … ENGINE = …`** ; pour les nouvelles tables, on règle plutôt `default_storage_engine`.
- C'est une **reconstruction complète par copie** : potentiellement **longue**, **bloquante en écriture**, et gourmande en **espace disque** — d'où l'intérêt des outils de conversion **en ligne** sur les grosses tables.
- La conversion **vers InnoDB** fait apparaître transactions et application des **clés étrangères** (prévoir une bonne clé primaire) ; la conversion **vers un moteur non transactionnel** est **refusée tant que des clés étrangères subsistent** (erreur 1217) — il faut les supprimer au préalable.
- Chaque moteur cible a ses contraintes : **Memory** (pas de `BLOB`/`TEXT`, volatil), **Archive** (ajout seul), **ColumnStore** (charger en masse), **S3** (lecture seule, réversible).
- **Sauvegardez, testez, planifiez** : vérifiez les dépendances FK et contrôlez le résultat après conversion.

⏭️ [Moteurs spécialisés](/07-moteurs-de-stockage/10-moteurs-specialises.md)
