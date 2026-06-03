🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.6 Insertion de données (INSERT, INSERT INTO SELECT, LOAD DATA)

> **Chapitre 2 : Bases du SQL** · Niveau : Débutant  
> Version de référence : **MariaDB 12.3 LTS**

Une fois les tables créées, il faut les **remplir**. MariaDB offre plusieurs voies pour insérer des données, de l'ajout d'une ligne unique au chargement massif d'un fichier : `INSERT` sous ses différentes formes, `INSERT ... SELECT` pour recopier le résultat d'une requête, et `LOAD DATA` pour importer un fichier en masse. Cette section présente aussi la gestion des **doublons** et quelques réflexes de performance pour les gros volumes.

## `INSERT` : la forme de base

L'instruction `INSERT` ajoute une ou plusieurs lignes. La forme recommandée précise explicitement la liste des colonnes :

```sql
INSERT INTO client (nom, email, pays)
VALUES ('Dupont', 'dupont@example.com', 'FR');
```

On peut insérer **plusieurs lignes en une seule instruction**, ce qui est nettement plus efficace que d'enchaîner des `INSERT` séparés :

```sql
INSERT INTO client (nom, email, pays) VALUES
    ('Martin', 'martin@example.com', 'FR'),
    ('Smith',  'smith@example.com',  'US'),
    ('Rossi',  'rossi@example.com',  'IT');
```

Les colonnes non mentionnées reçoivent leur valeur par défaut (`DEFAULT`), `NULL` si elles sont nullables, ou une valeur générée s'il s'agit d'une colonne `AUTO_INCREMENT`. Omettre la liste des colonnes est possible (`INSERT INTO client VALUES (...)`) mais **déconseillé** : il faut alors fournir toutes les valeurs dans l'ordre exact des colonnes, ce qui rend le code fragile à la moindre évolution du schéma. MariaDB propose en outre une syntaxe alternative avec `SET` :

```sql
INSERT INTO client SET nom = 'Durand', email = 'durand@example.com';
```

Lorsqu'une colonne `AUTO_INCREMENT` génère une valeur, la fonction `LAST_INSERT_ID()` permet de la récupérer immédiatement après l'insertion :

```sql
INSERT INTO client (nom, email) VALUES ('Leroy', 'leroy@example.com');
SELECT LAST_INSERT_ID();   -- identifiant auto-généré de la dernière insertion
```

## `INSERT ... SELECT`

`INSERT ... SELECT` insère dans une table le **résultat d'une requête** portant sur une autre (ou la même). C'est l'outil idéal pour recopier, filtrer ou transformer des données existantes :

```sql
INSERT INTO client_archive (id, nom, email)
SELECT id, nom, email
FROM client
WHERE actif = FALSE;
```

La requête `SELECT` peut comporter des jointures, des expressions et des fonctions, ce qui permet de transformer les données au passage. Le nombre et le type des colonnes sélectionnées doivent correspondre à ceux de la liste cible.

## Gérer les doublons et les conflits

Par défaut, insérer une ligne qui violerait une contrainte d'unicité (clé primaire ou `UNIQUE`) provoque une **erreur**. Trois mécanismes permettent d'adapter ce comportement.

`INSERT IGNORE` **ignore** les lignes problématiques (doublon, certaines violations) et poursuit, en émettant de simples avertissements au lieu d'erreurs :

```sql
INSERT IGNORE INTO client (nom, email) VALUES ('Martin', 'martin@example.com');
```

`INSERT ... ON DUPLICATE KEY UPDATE` réalise un **« upsert »** : si la ligne existe déjà (selon une clé), elle est **mise à jour en place** plutôt que réinsérée. On référence la valeur qui aurait été insérée via `VALUES(colonne)` :

```sql
INSERT INTO stock (produit_id, quantite) VALUES (42, 10)
ON DUPLICATE KEY UPDATE quantite = quantite + VALUES(quantite);
```

`REPLACE` enfin : si une ligne de même clé existe, elle est **supprimée puis recréée**. Attention, ce n'est pas une simple mise à jour — la suppression peut déclencher des actions de clé étrangère en cascade, des triggers `DELETE`/`INSERT` et faire avancer le compteur `AUTO_INCREMENT`. Pour mettre une valeur à jour, `ON DUPLICATE KEY UPDATE` est généralement préférable.

```sql
REPLACE INTO config (cle, valeur) VALUES ('theme', 'sombre');
```

## `LOAD DATA` : chargement en masse depuis un fichier

Pour importer un grand volume depuis un fichier texte (CSV typiquement), `LOAD DATA INFILE` est **bien plus rapide** qu'une multitude d'`INSERT`. On y décrit le format du fichier — séparateurs de champs et de lignes, caractère d'encadrement — et l'on peut sauter une ligne d'en-tête :

```sql
LOAD DATA LOCAL INFILE 'clients.csv'
INTO TABLE client
FIELDS TERMINATED BY ',' ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 LINES                       -- ignore la ligne d'en-tête
(nom, email, pays);
```

Le mot-clé `LOCAL` indique que le fichier se trouve **côté client**, et non sur le serveur. Pour des raisons de sécurité, le chargement local doit être autorisé (variable `local_infile`), tandis que le chargement côté serveur est restreint aux emplacements définis par `secure_file_priv`. Une liste de colonnes et une clause `SET` permettent de mapper et de transformer les champs au passage. À noter qu'une instruction sœur, `LOAD XML INFILE`, fait de même pour des fichiers XML, et que l'outil en ligne de commande `mariadb-import` encapsule `LOAD DATA` pour un usage en script.

## Performance des insertions massives

Trois réflexes accélèrent fortement l'insertion de gros volumes. D'abord, **regrouper les lignes** dans des `INSERT` multi-lignes, qui réduisent le nombre d'allers-retours et d'analyses de requêtes. Ensuite, **encadrer les insertions par une transaction** : sous InnoDB, chaque instruction en mode *autocommit* déclenche une écriture sur disque, alors qu'un `COMMIT` unique en fin de lot mutualise ce coût.

```sql
START TRANSACTION;
INSERT INTO mesure (capteur_id, valeur) VALUES (...);
INSERT INTO mesure (capteur_id, valeur) VALUES (...);
-- ... nombreuses insertions ...
COMMIT;
```

Enfin, pour les volumes vraiment importants, **`LOAD DATA`** reste la voie la plus rapide. Les optimisations de chargement et de construction d'index en masse sont approfondies au chapitre 15 (notamment `innodb_alter_copy_bulk`, §15.6), et l'import/export à des fins de sauvegarde au chapitre 12.

## En résumé

`INSERT` ajoute des lignes — idéalement avec une liste de colonnes explicite et, pour plusieurs lignes, sous forme multi-lignes ; `LAST_INSERT_ID()` récupère l'identifiant auto-généré. `INSERT ... SELECT` recopie et transforme des données depuis une requête. Pour les conflits de clé, `INSERT IGNORE` passe outre, `ON DUPLICATE KEY UPDATE` réalise un *upsert* en place, et `REPLACE` supprime puis réinsère (à manier avec prudence). Enfin, `LOAD DATA` charge un fichier en masse, et le couple **insertions groupées + transaction** constitue le réflexe de base pour de bonnes performances sur de gros volumes.

---

← Section précédente : [2.5 Contraintes](05-contraintes.md) · [Sommaire du chapitre](README.md) · Section suivante : [2.7 Requêtes de sélection simples](07-requetes-selection-simples.md) →

⏭️ [Requêtes de sélection simples (SELECT, WHERE, ORDER BY, LIMIT)](/02-bases-du-sql/07-requetes-selection-simples.md)
