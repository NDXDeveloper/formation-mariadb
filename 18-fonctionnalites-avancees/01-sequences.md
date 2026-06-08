🔝 Retour au [Sommaire](/SOMMAIRE.md)

[← Retour au chapitre 18](README.md)

# 18.1 Sequences (`CREATE SEQUENCE`)

## Qu'est-ce qu'une séquence ?

Une **séquence** est un objet de base de données dont l'unique rôle est de produire, à la demande, une suite de nombres entiers selon des règles que l'on définit : valeur de départ, pas d'incrément, bornes, mise en cache et éventuel cyclage. Contrairement à l'attribut `AUTO_INCREMENT`, qui est attaché à une colonne précise d'une table, une séquence existe de façon autonome. Elle peut alimenter plusieurs tables, être interrogée sans qu'aucune ligne ne soit insérée, et offrir un contrôle bien plus fin sur la génération des valeurs.

Présentes dans MariaDB depuis la version 10.3, les séquences répondent à un standard SQL et à une attente forte des utilisateurs venant d'Oracle ou de PostgreSQL, où elles constituent le mécanisme d'identité de référence. Elles sont stockées en interne comme une table à une seule ligne (moteur InnoDB par défaut), ce qui les rend transparentes pour les outils de sauvegarde et robustes face aux arrêts brutaux.

## Séquence ou `AUTO_INCREMENT` ?

Avant d'entrer dans la syntaxe, il est utile de situer les deux mécanismes l'un par rapport à l'autre. Aucun n'est universellement supérieur : le choix dépend du besoin.

| Critère | `AUTO_INCREMENT` | Séquence |
|---|---|---|
| Portée | Liée à une colonne d'une table | Objet indépendant, partageable entre tables |
| Configuration | Minimale (incrément global du serveur) | Fine : pas, bornes, cache, cyclage |
| Sens de progression | Croissant uniquement | Croissant **ou** décroissant |
| Cyclage à la borne | Impossible | Possible (`CYCLE`) |
| Obtenir une valeur sans `INSERT` | Non | Oui (`NEXTVAL`) |
| Portabilité Oracle / PostgreSQL | Faible | Forte |
| Simplicité de mise en œuvre | Très simple | Plus verbeux |

En résumé, on privilégie `AUTO_INCREMENT` pour une simple clé technique mono-table, et une séquence dès que l'on a besoin de partager un compteur entre plusieurs tables, de réserver un identifiant avant l'insertion, de progresser autrement que par pas de 1, ou d'assurer la compatibilité avec un schéma Oracle ou PostgreSQL.

## Créer une séquence

### Syntaxe générale

```sql
CREATE [OR REPLACE] [TEMPORARY] SEQUENCE [IF NOT EXISTS] nom_sequence
  [ INCREMENT [=] increment ]
  [ MINVALUE [=] minvalue | NO MINVALUE ]
  [ MAXVALUE [=] maxvalue | NO MAXVALUE ]
  [ START [ WITH ] depart ]
  [ CACHE [=] cache | NOCACHE ]
  [ CYCLE | NOCYCLE ]
  [ AS type_entier ]
  [ ENGINE = nom_moteur ];
```

La forme la plus simple se contente du nom :

```sql
CREATE SEQUENCE seq_commande;
```

Cette séquence utilise alors l'ensemble des valeurs par défaut, qui correspondent à une séquence croissante classique :

| Clause | Rôle | Valeur par défaut (séquence croissante) |
|---|---|---|
| `INCREMENT` | Pas d'incrément à chaque appel | `1` |
| `START WITH` | Première valeur produite | égale à `MINVALUE`, soit `1` |
| `MINVALUE` | Borne basse | `1` |
| `MAXVALUE` | Borne haute | `9223372036854775806` (max d'un `BIGINT` signé, la dernière valeur étant réservée) |
| `CACHE` | Nombre de valeurs réservées en mémoire | `1000` |
| `CYCLE` / `NOCYCLE` | Reprise au début une fois la borne atteinte | `NOCYCLE` |

Quelques précisions sur les clauses les plus structurantes :

L'**`INCREMENT`** définit le pas. Une valeur positive donne une séquence croissante, une valeur négative une séquence décroissante. Il ne peut pas être nul.

Le **`CACHE`** est un levier de performance. Plutôt que de mettre à jour la table sous-jacente à chaque valeur produite, le serveur réserve d'un coup un bloc de `n` valeurs en mémoire. C'est rapide, mais cela a une contrepartie abordée plus loin (les valeurs réservées non consommées sont perdues en cas de redémarrage). `NOCACHE` impose l'écriture à chaque appel.

Le **`CYCLE`** autorise la séquence à repartir de sa borne basse (ou haute, pour une séquence décroissante) une fois la borne opposée atteinte. Par défaut (`NOCYCLE`), atteindre la borne provoque une erreur lors de l'appel suivant.

Par défaut, une séquence est de type `BIGINT`. La clause optionnelle `AS type` (introduite en MariaDB 11.5) permet de choisir un type entier plus petit (par exemple `SMALLINT UNSIGNED`) afin de contraindre l'amplitude des valeurs et de réduire l'occupation.

### Exemples

Une séquence de numérotation de factures démarrant à 1000, avec un petit cache :

```sql
CREATE SEQUENCE seq_facture
  START WITH 1000
  INCREMENT 1
  MINVALUE 1000
  MAXVALUE 999999
  CACHE 50
  NOCYCLE;
```

Une séquence décroissante :

```sql
CREATE SEQUENCE seq_decompte
  START WITH 100
  INCREMENT -1
  MINVALUE 1
  MAXVALUE 100;
```

Une séquence cyclique, par exemple pour répartir un travail sur cinq files :

```sql
CREATE SEQUENCE seq_file
  START WITH 1
  MINVALUE 1
  MAXVALUE 5
  CYCLE;
-- ... 4, 5, 1, 2, 3, 4, 5, 1, ...
```

> **Compatibilité Oracle.** Sous `sql_mode = ORACLE`, MariaDB accepte la phraséologie Oracle telle que `INCREMENT BY` en complément des formes natives. Ce point est repris dans la migration depuis Oracle (§19.2.1).

## Obtenir et manipuler les valeurs

### `NEXTVAL` et `NEXT VALUE FOR`

La fonction `NEXTVAL()` renvoie la valeur suivante et fait progresser la séquence :

```sql
SELECT NEXTVAL(seq_commande);   -- 1
SELECT NEXTVAL(seq_commande);   -- 2
SELECT NEXTVAL(seq_commande);   -- 3
```

La forme standard SQL `NEXT VALUE FOR` est strictement équivalente :

```sql
SELECT NEXT VALUE FOR seq_commande;   -- 4
```

### `PREVIOUS VALUE FOR` et `LASTVAL`

Ces deux expressions renvoient la **dernière valeur produite par `NEXTVAL` dans la session courante**, sans faire progresser la séquence. Elles sont locales à la connexion : si aucun `NEXTVAL` n'a encore été appelé dans la session, le résultat est `NULL`.

```sql
SELECT PREVIOUS VALUE FOR seq_commande;   -- 4
SELECT LASTVAL(seq_commande);             -- 4
```

C'est le mécanisme typique pour récupérer l'identifiant que l'on vient d'attribuer, par exemple afin de l'utiliser comme clé étrangère dans une ligne enfant insérée juste après.

### `SETVAL`

`SETVAL()` repositionne explicitement une séquence. Sa syntaxe est compatible avec PostgreSQL :

```sql
SETVAL(nom_sequence, valeur, [is_used, [round]])
```

Le paramètre `is_used` vaut `TRUE` (1) par défaut : la `valeur` est alors considérée comme déjà consommée, et le prochain `NEXTVAL` renverra la valeur suivante. À `FALSE` (0), le prochain `NEXTVAL` renverra exactement `valeur`. Le paramètre `round`, rarement utilisé, sert à indiquer le numéro de cycle pour les séquences `CYCLE`.

```sql
SELECT SETVAL(seq_facture, 5000);          -- prochain NEXTVAL : 5001
SELECT SETVAL(seq_facture, 5000, FALSE);   -- prochain NEXTVAL : 5000
```

Un point essentiel : **`SETVAL` ne fait jamais reculer une séquence**. Si la valeur demandée est inférieure à la position courante, la demande est ignorée. Ce comportement n'est pas une limitation arbitraire mais une garantie de cohérence en réplication, où des instructions peuvent arriver dans un ordre non strictement chronologique.

## Utiliser une séquence comme valeur par défaut

Une séquence peut alimenter directement la valeur par défaut d'une colonne, ce qui reproduit le confort de l'`AUTO_INCREMENT` tout en conservant la souplesse d'un objet partagé :

```sql
CREATE TABLE commande (
  id      BIGINT       NOT NULL DEFAULT NEXTVAL(seq_commande) PRIMARY KEY,
  client  VARCHAR(100),
  montant DECIMAL(10,2)
);

INSERT INTO commande (client, montant) VALUES ('Dupont', 149.90);
INSERT INTO commande (client, montant) VALUES ('Martin',  79.00);
```

Comme la séquence est indépendante, la même peut servir à plusieurs tables qui partageraient alors un espace d'identifiants commun — utile, par exemple, lorsque l'on veut garantir l'unicité d'un numéro à travers plusieurs entités.

## Inspecter et modifier une séquence

### Voir la définition et l'état

`SHOW CREATE SEQUENCE` restitue la définition :

```sql
SHOW CREATE SEQUENCE seq_commande\G
```

Comme une séquence est stockée sous forme de table, on peut aussi interroger directement son contenu pour lire son état interne :

```sql
SELECT * FROM seq_commande\G
```

Les colonnes exposées comprennent notamment `next_not_cached_value` (la prochaine valeur non encore mise en cache), `minimum_value`, `maximum_value`, `start_value`, `increment`, `cache_size`, `cycle_option` et `cycle_count`.

### `ALTER SEQUENCE`

On modifie les paramètres d'une séquence existante avec `ALTER SEQUENCE`, et on la réinitialise avec `RESTART` :

```sql
ALTER SEQUENCE seq_commande INCREMENT 5 MAXVALUE 1000000;
ALTER SEQUENCE seq_commande RESTART;            -- repart à START WITH
ALTER SEQUENCE seq_commande RESTART WITH 500;   -- repart à 500
```

### Supprimer une séquence

```sql
DROP SEQUENCE IF EXISTS seq_commande;
```

L'option `TEMPORARY` permet par ailleurs de créer une séquence éphémère, limitée à la session, et `OR REPLACE` / `IF NOT EXISTS` facilitent l'écriture de scripts ré-exécutables.

## Comportements à connaître et pièges

### Des trous sont normaux

La génération de valeurs **n'est pas transactionnelle**. Si une transaction appelle `NEXTVAL` puis effectue un `ROLLBACK`, la valeur consommée n'est pas restituée : un trou apparaît dans la numérotation. C'est exactement le comportement d'`AUTO_INCREMENT`, et c'est volontaire — restituer les valeurs imposerait une sérialisation qui ruinerait les performances.

La conséquence pratique est importante : **une séquence ne garantit pas une numérotation sans trou.** Pour un besoin réglementaire de numérotation strictement continue (certaines numérotations de factures, par exemple), il faut prévoir un mécanisme applicatif dédié plutôt que de s'appuyer sur une séquence ou un `AUTO_INCREMENT`.

### Cache et redémarrage

Le cache accélère la production de valeurs mais réserve un bloc à l'avance. Les valeurs réservées et non encore consommées sont **perdues lors d'un redémarrage du serveur**, ce qui crée également des trous. Le réglage du `CACHE` est donc un compromis : un grand cache maximise le débit mais accroît les sauts potentiels ; `NOCACHE` supprime ces sauts au prix d'une écriture à chaque appel. Pour une clé technique, des trous sont sans conséquence et un cache confortable est préférable.

### Réplication

Les séquences sont répliquées. Les opérations qui font avancer le compteur sont journalisées, et la propriété « jamais en arrière » de `SETVAL` garantit la convergence même lorsque des instructions parviennent dans le désordre au réplica. Le réglage du cache et le format du binlog peuvent influencer le détail du journal ; ces aspects rejoignent les considérations générales de réplication (Partie 6).

## Privilèges requis

L'utilisation des séquences s'appuie sur les privilèges habituels appliqués à la « table » qui les représente :

- `NEXTVAL` et `SETVAL` requièrent le privilège **`INSERT`** sur la séquence ;
- la lecture via `SELECT` ou `PREVIOUS VALUE FOR` requiert le privilège **`SELECT`** ;
- `CREATE SEQUENCE`, `ALTER SEQUENCE` et `DROP SEQUENCE` requièrent respectivement **`CREATE`**, **`ALTER`** et **`DROP`**.

## Cas d'usage typiques

Les séquences brillent dès que la génération d'identifiants doit être maîtrisée plutôt que subie. On les retrouve pour produire un **identifiant partagé** par plusieurs tables, pour **réserver un identifiant avant l'insertion** (réserver l'`id` d'une commande pour le journaliser ou le retourner à un service tiers avant même d'écrire la ligne), ou pour mettre en place des **identifiants distribués multi-nœuds** en attribuant à chaque nœud un `START` et un `INCREMENT` décalés, sur le même principe que `auto_increment_offset` et `auto_increment_increment`.

Elles jouent enfin un rôle clé lors d'une **migration depuis Oracle ou PostgreSQL** : les schémas issus de ces systèmes reposent massivement sur des séquences, et MariaDB permet de les transposer presque à l'identique. Ce point est développé dans les sections consacrées à la migration depuis Oracle (§19.2.1) et depuis PostgreSQL (§19.2.3).

## Points clés à retenir

- Une séquence est un **objet autonome** qui génère des entiers selon des règles configurables, là où `AUTO_INCREMENT` reste lié à une colonne.
- `NEXTVAL` (ou `NEXT VALUE FOR`) produit et avance ; `PREVIOUS VALUE FOR` / `LASTVAL` relisent la dernière valeur **de la session** ; `SETVAL` repositionne, sans jamais reculer.
- Une séquence peut servir de `DEFAULT` de colonne et être partagée entre plusieurs tables.
- La génération n'est **pas transactionnelle** et le cache crée des sauts : des **trous** sont à prévoir et acceptables pour une clé technique, mais inadaptés à une numérotation strictement continue.
- Le réglage du `CACHE` arbitre entre débit et continuité ; les séquences sont sûres en réplication.
- Atout majeur pour la **portabilité Oracle / PostgreSQL** et pour les scénarios d'identifiants partagés ou distribués.

⏭️ [Tables temporelles (System-Versioned Tables)](/18-fonctionnalites-avancees/02-system-versioned-tables.md)
