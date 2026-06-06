🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.3 — Système de privilèges

Le système de privilèges constitue la **couche d'autorisation** du modèle de sécurité : une fois un client authentifié (phase 1, vue en 10.1), ce sont ses privilèges qui déterminent, pour chaque instruction, ce qu'il a le droit de faire (phase 2). Cette section présente le système dans son ensemble — la notion de privilège, leur catalogue, le mécanisme de vérification et les outils d'inspection. L'attribution proprement dite (`GRANT`), la révocation (`REVOKE`) et le détail des différents niveaux font l'objet des sous-sections 10.3.1, 10.3.2 et 10.3.3.

---

## Qu'est-ce qu'un privilège ?

Un privilège est une autorisation d'effectuer un type d'opération précis sur un objet ou sur le serveur : lire une table, créer une base, arrêter le serveur, etc. Un compte ne peut accomplir une opération que s'il détient le privilège correspondant, au bon niveau de portée.

Deux propriétés du modèle MariaDB sont à garder à l'esprit :

- **Les privilèges sont positifs et cumulatifs.** On accorde des droits ; on n'en « interdit » pas. Il n'existe pas de mécanisme de *deny* qui viendrait soustraire, sur un objet particulier, un droit accordé plus largement. Pour restreindre un compte, on accorde au niveau le plus étroit nécessaire (principe du moindre privilège) ; pour lui retirer un droit, on le **révoque** (voir 10.3.2).
- **Les privilèges s'additionnent entre les niveaux.** Les droits effectifs d'un compte pour une opération donnée sont l'union de tous les privilèges applicables, du niveau global au niveau colonne.

---

## Le catalogue des privilèges

MariaDB reconnaît un large ensemble de privilèges, que l'on peut classer en quatre familles. Les tableaux ci-dessous recensent les plus courants ; la liste **complète et faisant autorité** pour une version donnée s'obtient avec `SHOW PRIVILEGES`.

### Privilèges sur les données (DML)

| Privilège | Autorise |
|---|---|
| `SELECT` | Lire les données |
| `INSERT` | Insérer des lignes |
| `UPDATE` | Modifier des lignes |
| `DELETE` | Supprimer des lignes |

### Privilèges de structure (DDL)

| Privilège | Autorise |
|---|---|
| `CREATE` | Créer des bases et des tables |
| `ALTER` | Modifier la structure d'une table |
| `DROP` | Supprimer bases, tables et vues |
| `INDEX` | Créer et supprimer des index |
| `CREATE VIEW` | Créer des vues |
| `SHOW VIEW` | Consulter la définition d'une vue (`SHOW CREATE VIEW`) |
| `CREATE ROUTINE` | Créer des procédures et fonctions stockées |
| `ALTER ROUTINE` | Modifier ou supprimer des routines |
| `EXECUTE` | Exécuter des routines stockées |
| `CREATE TEMPORARY TABLES` | Créer des tables temporaires |
| `TRIGGER` | Créer et supprimer des triggers |
| `EVENT` | Gérer les événements planifiés |
| `REFERENCES` | Créer des clés étrangères |

### Privilèges d'administration

| Privilège | Autorise |
|---|---|
| `CREATE USER` | Créer, supprimer, renommer des comptes |
| `GRANT OPTION` | Transmettre ses propres privilèges à d'autres comptes |
| `PROCESS` | Voir l'ensemble des threads (`SHOW PROCESSLIST`) |
| `RELOAD` | Opérations `FLUSH` et `RESET` |
| `SHUTDOWN` | Arrêter le serveur |
| `FILE` | Lire/écrire des fichiers côté serveur (`LOAD DATA`, `SELECT … INTO OUTFILE`) |
| `LOCK TABLES` | Verrouiller explicitement des tables (avec `SELECT`) |
| `SHOW DATABASES` | Lister toutes les bases |
| `REPLICATION SLAVE` | Permettre à un réplica de lire le binlog |
| `REPLICATION CLIENT` | Interroger l'état de la réplication |
| `SUPER` | Opérations d'administration sensibles |

Historiquement, `SUPER` agrégeait de nombreuses opérations d'administration. MariaDB tend à le **décomposer en privilèges granulaires** plus ciblés (`CONNECTION ADMIN`, `BINLOG ADMIN`, `READ_ONLY ADMIN`, `SET USER`, etc.), qui permettent d'accorder précisément une capacité sans en concéder d'autres. Ces privilèges granulaires sont traités à la section 10.11.

### Privilèges spéciaux

Trois entrées ne désignent pas une opération particulière mais jouent un rôle à part : `ALL PRIVILEGES`, `USAGE` et `GRANT OPTION`. Elles sont détaillées ci-dessous.

---

## Trois privilèges à part

**`ALL PRIVILEGES`** (ou simplement `ALL`) est un raccourci désignant *tous* les privilèges disponibles au niveau considéré — mais **à l'exception de `GRANT OPTION`**, qui doit toujours être accordé séparément. `GRANT ALL PRIVILEGES ON base.* TO …` octroie donc l'ensemble des droits sur une base sans pour autant autoriser le compte à les redistribuer. À manier avec prudence : c'est l'antithèse du moindre privilège.

**`USAGE`** signifie « aucun privilège ». Il sert de marqueur neutre : il permet de faire référence à un compte (par exemple pour en modifier les options de ressources ou de TLS) sans lui accorder de droit. Un `SHOW GRANTS` renvoie d'ailleurs toujours au minimum une ligne `GRANT USAGE ON *.* TO …`, qui matérialise l'existence du compte même dépourvu de tout privilège.

**`GRANT OPTION`** est l'autorisation de **transmettre** à d'autres comptes les privilèges que l'on détient soi-même. Il est distinct des privilèges qu'il accompagne et se gère séparément (clause `WITH GRANT OPTION` lors de l'attribution). Un compte ne peut déléguer que des droits qu'il possède effectivement.

À côté de ces trois entrées, le privilège `PROXY` autorise un compte à agir comme mandataire d'un autre ; il intervient notamment avec les authentifications PAM/LDAP (voir 10.5).

---

## Comment les privilèges sont vérifiés

Les privilèges se déclinent selon une hiérarchie de portées, de la plus large à la plus précise :

- **Global** (`*.*`) — s'applique à toutes les bases ;
- **Base de données** (`base.*`) — s'applique à tous les objets d'une base ;
- **Table** (`base.table`) — s'applique à une table précise ;
- **Colonne** — s'applique à des colonnes nommées ;
- **Routine** — s'applique aux procédures et fonctions stockées.

Lorsqu'un compte tente une opération, le serveur évalue ses droits en **accumulant** les privilèges applicables à chacun de ces niveaux. Un droit accordé globalement vaut partout ; un droit accordé sur une base vaut pour toutes ses tables ; et ainsi de suite. Si le niveau global suffit à autoriser l'opération, les niveaux inférieurs n'ont pas besoin d'être consultés ; sinon, la décision résulte de la somme des droits trouvés aux niveaux plus spécifiques. Le fonctionnement détaillé de chaque niveau — et les tables système qui les stockent — est présenté en 10.3.3.

---

## Où sont stockés les privilèges

Conformément à ce qui a été vu en 10.1, chaque niveau de portée correspond à une table de droits dans la base `mysql` : `global_priv` (et la vue `user`) pour le niveau global, `db` pour les bases, `tables_priv` pour les tables, `columns_priv` pour les colonnes et `procs_priv` pour les routines. Ces tables sont chargées en mémoire au démarrage ; les instructions `GRANT` et `REVOKE` les mettent à jour automatiquement, sans `FLUSH PRIVILEGES`.

---

## Inspecter les privilèges

Deux commandes principales permettent d'explorer le système de privilèges.

`SHOW PRIVILEGES` énumère l'ensemble des privilèges reconnus par le serveur, avec leur contexte d'application et une description — c'est la référence à jour pour la version installée :

```sql
SHOW PRIVILEGES;
```

`SHOW GRANTS` affiche, sous forme d'instructions `GRANT`, les privilèges effectivement détenus par un compte :

```sql
SHOW GRANTS FOR 'alice'@'localhost';
SHOW GRANTS;                      -- pour le compte courant
SHOW GRANTS FOR CURRENT_USER();
```

Pour une exploration programmatique, les vues de `information_schema` exposent les privilèges par niveau :

```sql
SELECT * FROM information_schema.USER_PRIVILEGES;   -- niveau global
SELECT * FROM information_schema.SCHEMA_PRIVILEGES;  -- niveau base
SELECT * FROM information_schema.TABLE_PRIVILEGES;   -- niveau table
SELECT * FROM information_schema.COLUMN_PRIVILEGES;  -- niveau colonne
```

---

## Privilèges et rôles

Attribuer les privilèges compte par compte devient vite ingérable à mesure que les utilisateurs se multiplient. Les **rôles** (section 10.4) permettent de regrouper un ensemble de privilèges sous un nom unique, puis d'attribuer ce rôle aux comptes concernés — une approche bien plus maintenable, sur laquelle s'appuient la plupart des politiques d'accès en production.

---

## À retenir

Le système de privilèges régit l'autorisation dans MariaDB. Les privilèges sont **positifs** (on accorde, on ne « refuse » pas) et **cumulatifs** entre les niveaux. Ils se répartissent en privilèges de données (DML), de structure (DDL), d'administration et spéciaux. Trois entrées méritent une attention particulière : `ALL PRIVILEGES` (tout sauf `GRANT OPTION`), `USAGE` (aucun privilège) et `GRANT OPTION` (droit de redistribuer). La portée s'échelonne du **global** à la **colonne**, et les droits effectifs résultent de leur accumulation. `SHOW PRIVILEGES` donne la liste de référence des privilèges, `SHOW GRANTS` ceux d'un compte. Les sections suivantes détaillent `GRANT` (10.3.1), `REVOKE` (10.3.2) et les niveaux de portée (10.3.3).

---

> 🔔 **Note de version.** Le modèle de privilèges décrit ici est stable dans **MariaDB 12.3 LTS**. La principale évolution récente est la décomposition progressive de `SUPER` en privilèges granulaires (section 10.11). Pour la liste exacte des privilèges d'une instance donnée, `SHOW PRIVILEGES` fait toujours foi.

⏭️ [GRANT : Attribution de privilèges](/10-securite-gestion-utilisateurs/03.1-grant.md)
