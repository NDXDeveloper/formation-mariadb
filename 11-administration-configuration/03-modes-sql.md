🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.3 — Modes SQL (`sql_mode`)

> Troisième section du chapitre 11. Après les fichiers de configuration (§11.1) et les variables (§11.2), on s'arrête sur une variable particulière, `sql_mode`, qui agit comme un interrupteur global sur deux dimensions essentielles : la **rigueur** avec laquelle MariaDB valide les données, et sa **compatibilité** avec d'autres SGBD.

---

## Qu'est-ce que `sql_mode` ?

`sql_mode` est une variable système qui gouverne la manière dont MariaDB interprète et contrôle le SQL. Deux missions s'y croisent. D'une part, elle règle le **niveau d'exigence** vis-à-vis des données : faut-il refuser une valeur invalide ou tenter de l'ajuster silencieusement ? D'autre part, elle permet d'**émuler le comportement** d'autres bases — au premier rang desquelles Oracle — pour faciliter migrations et portabilité.

Comprendre `sql_mode` est doublement utile : c'est un garde-fou pour l'intégrité des données, et c'est aussi la source de surprises classiques, notamment lors d'une migration où un mode plus strict fait soudain échouer des requêtes qui passaient auparavant.

---

## Comment le mode SQL se définit

La valeur de `sql_mode` est une **chaîne d'options séparées par des virgules, sans espaces** ; les noms d'options sont insensibles à la casse. C'est une variable système ordinaire : elle existe en portée globale et de session, et elle est dynamique (modifiable à chaud). Les notions vues en 11.2 s'appliquent donc directement — une session hérite de la valeur globale à la connexion, et un changement de session n'affecte que le client courant.

On la consulte avec la syntaxe `@@` et on la modifie avec `SET` (cf. 11.2.1) :

```sql
-- Consulter
SELECT @@sql_mode;          -- valeur de session
SELECT @@global.sql_mode;   -- valeur globale

-- Modifier la session courante
SET SESSION sql_mode = 'STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION';

-- Modifier la valeur globale (pour les nouvelles connexions)
SET GLOBAL sql_mode = 'TRADITIONAL';
```

Pour la rendre persistante, on l'inscrit dans un fichier de configuration (cf. 11.1), ou on la passe via l'option de démarrage `--sql-mode` :

```ini
[mariadbd]
sql_mode = STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
```

---

## Le mode strict : le réglage le plus structurant

Le cœur de `sql_mode` est le **mode strict**, actif dès lors que l'un des deux drapeaux `STRICT_TRANS_TABLES` ou `STRICT_ALL_TABLES` est présent. Il détermine la réaction du serveur face à une valeur invalide ou manquante dans une instruction qui modifie des données (`INSERT`, `UPDATE`) ou dans un `CREATE TABLE`.

En **mode strict** — le comportement par défaut de MariaDB depuis la version 10.2.4 — une valeur incorrecte provoque une **erreur** et l'instruction est rejetée. En **mode permissif**, MariaDB **ajuste silencieusement** la valeur : il tronque une chaîne trop longue, ramène un nombre hors limites à la borne la plus proche, ou insère une valeur par défaut. L'exemple canonique est l'insertion d'une chaîne trop longue :

```sql
-- Colonne définie en VARCHAR(5)
INSERT INTO produits (code) VALUES ('ABCDEFG');

-- Mode strict (défaut) : l'instruction échoue
--   ERROR 1406 (22001): Data too long for column 'code' at row 1

-- Mode permissif : la valeur est tronquée en 'ABCDE', avec un simple avertissement
```

La différence entre les deux drapeaux stricts tient aux tables non transactionnelles : `STRICT_TRANS_TABLES` privilégie la cohérence (en évitant les mises à jour partielles), tandis que `STRICT_ALL_TABLES` est plus rigoureux mais peut laisser une table non transactionnelle partiellement modifiée si l'erreur survient en milieu d'instruction. Sur des tables InnoDB transactionnelles, la question ne se pose pas : l'instruction est annulée proprement. À noter que le mot-clé `IGNORE` dans une requête permet, ponctuellement, de retomber sur un comportement « avertissement » même en mode strict.

L'enjeu est l'**intégrité des données** : le mode strict transforme une corruption discrète (une valeur tronquée à l'insu de l'application) en une erreur visible et traçable. C'est pourquoi on recommande très généralement de le conserver actif. Comme la valeur effective peut être modifiée par la distribution dans ses fichiers de configuration, le bon réflexe reste de **vérifier `@@sql_mode` sur l'instance réelle** plutôt que de présumer du défaut compilé.

---

## Des modes utiles à connaître

Au-delà du mode strict, `sql_mode` rassemble de nombreux drapeaux. En voici une sélection parmi les plus courants.

| Mode | Effet |
|------|-------|
| `STRICT_TRANS_TABLES` | Rejette les valeurs invalides/manquantes pour les tables transactionnelles |
| `STRICT_ALL_TABLES` | Idem pour toutes les tables (risque de mise à jour partielle sur les non transactionnelles) |
| `ERROR_FOR_DIVISION_BY_ZERO` | Erreur sur une division par zéro dans `INSERT`/`UPDATE` (au lieu d'un `NULL`) |
| `NO_ZERO_DATE` / `NO_ZERO_IN_DATE` | Interdit les dates `'0000-00-00'` ou comportant une partie nulle |
| `NO_ENGINE_SUBSTITUTION` | Erreur si le moteur demandé est indisponible, au lieu d'une substitution silencieuse |
| `ONLY_FULL_GROUP_BY` | Rejette les `GROUP BY` ambigus (colonnes non agrégées absentes du `GROUP BY`) — cf. 3.2 |
| `PIPES_AS_CONCAT` | `||` devient l'opérateur de concaténation de chaînes (au lieu du OU logique) |
| `ANSI_QUOTES` | Les guillemets `"` délimitent un identifiant (au lieu d'une chaîne) |

Ces drapeaux se combinent librement. `ONLY_FULL_GROUP_BY`, par exemple, oblige à écrire des regroupements sans ambiguïté, ce qui rejoint les bonnes pratiques évoquées au chapitre 3 sur `GROUP BY` et `HAVING`.

---

## Les modes combinés

Plutôt que d'énumérer chaque drapeau, on peut utiliser des **valeurs combinées** qui en activent plusieurs d'un coup.

`TRADITIONAL` fait se comporter MariaDB comme un serveur SQL traditionnel et exigeant ; il regroupe `STRICT_TRANS_TABLES`, `STRICT_ALL_TABLES`, `NO_ZERO_IN_DATE`, `NO_ZERO_DATE`, `ERROR_FOR_DIVISION_BY_ZERO`, `NO_AUTO_CREATE_USER` et `NO_ENGINE_SUBSTITUTION` (sous MariaDB, `NO_AUTO_CREATE_USER` en fait partie, contrairement à MySQL 8 qui l'a retiré). C'est un raccourci pratique quand on veut la rigueur maximale en matière d'intégrité.

`ANSI` rapproche MariaDB du standard SQL, en activant notamment `ANSI_QUOTES` et `PIPES_AS_CONCAT`. Il est utile pour écrire un SQL plus portable d'un SGBD à l'autre.

---

## Émuler un autre SGBD : le mode `ORACLE` et ses cousins

La seconde grande mission de `sql_mode` est l'émulation. Le mode le plus important à cet égard — et le plus présent dans cette formation — est **`ORACLE`**.

```sql
-- Activer la compatibilité Oracle pour la session
SET SESSION sql_mode = 'ORACLE';
```

Activer le mode Oracle modifie en profondeur l'interprétation du SQL : prise en charge d'une partie de la syntaxe PL/SQL, d'alias de types de données, et de comportements propres à Oracle. C'est précisément ce mode qui ouvre la porte aux nombreuses fonctionnalités de compatibilité Oracle abordées ailleurs dans la formation — la syntaxe `( + )` pour les jointures externes (§3.3.5), les fonctions `TO_DATE`, `TRUNC`, `TO_NUMBER` (§3.7.1), les tableaux associatifs (§8.1.4), `SYS_REFCURSOR` (§8.5.2). La migration depuis Oracle (§19.2.1) s'appuie largement sur ce levier.

MariaDB propose d'autres modes d'émulation visant des SGBD comme PostgreSQL, MS SQL Server ou DB2, mais leur portée est plus limitée et leur usage plus rare ; `ORACLE` est de loin le plus abouti et le plus employé.

---

## `OLD_MODE` : émuler d'anciennes versions

Il existe une variable voisine mais distincte, `old_mode`. Là où `sql_mode` sert à émuler le comportement d'**autres serveurs**, `old_mode` sert à retrouver le comportement d'**anciennes versions** de MariaDB ou MySQL. C'est un outil de transition, utile lorsqu'une mise à jour modifie un comportement par défaut et qu'on souhaite, provisoirement, conserver l'ancien le temps d'adapter l'application. On la mentionne ici pour ne pas la confondre avec `sql_mode` ; son usage relève des scénarios de migration (chapitre 19).

---

## Portée, persistance et migration

Quelques principes pratiques pour finir. Pour une configuration cohérente de l'instance, on définit `sql_mode` **globalement** dans un fichier (cf. 11.1), valeur que les nouvelles connexions hériteront. La portée de **session** reste précieuse pour un traitement particulier — par exemple un script d'import ou de migration qui a besoin temporairement du mode Oracle ou d'un mode allégé, sans toucher au réglage global. Il faut aussi savoir que de nombreux connecteurs et ORM imposent leur propre `sql_mode` à la connexion : la valeur vue par l'application n'est donc pas toujours celle du serveur.

Le point de vigilance majeur concerne la **migration**. Un changement de `sql_mode` par défaut — typiquement le passage en mode strict lors d'une montée de version — peut faire échouer des `INSERT`/`UPDATE` qui, auparavant, étaient acceptés au prix d'un ajustement silencieux. La tentation est alors de désactiver le mode strict pour « débloquer » l'application ; c'est rarement le bon choix, car cela rétablit le risque de corruption discrète. La démarche recommandée est d'auditer et de corriger les requêtes ou les données fautives, en conservant l'intégrité offerte par le mode strict. Ces aspects sont approfondis dans la partie consacrée à la migration (chapitre 19).

---

## En résumé

`sql_mode` est l'interrupteur qui gouverne à la fois la rigueur de validation des données et la compatibilité de MariaDB avec d'autres SGBD. Sa pièce maîtresse est le **mode strict** (`STRICT_TRANS_TABLES`/`STRICT_ALL_TABLES`), actif par défaut, qui transforme une valeur invalide en erreur plutôt qu'en ajustement silencieux — un gage d'intégrité qu'il vaut mieux préserver. De nombreux drapeaux et des valeurs combinées (`TRADITIONAL`, `ANSI`) affinent ce comportement, tandis que le mode **`ORACLE`** ouvre la voie aux fonctionnalités de compatibilité disséminées dans la formation. Variable globale et de session, dynamique, `sql_mode` se pérennise par fichier et constitue un point d'attention central lors des migrations.

La section suivante quitte le réglage du comportement pour la production de traces exploitables au quotidien — **11.4, Gestion des logs**.

⏭️ [Gestion des logs](/11-administration-configuration/04-gestion-logs.md)
