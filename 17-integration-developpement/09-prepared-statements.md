🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.9 Prepared statements et parameterized queries

Les *prepared statements* (requêtes préparées) sont le **mécanisme** derrière la parade aux injections vue au §17.8, et un **levier de performance** pour les requêtes répétées. Cette section en détaille le fonctionnement, au niveau applicatif comme au niveau du protocole, en consolidant les marqueurs par langage du §17.1, puis situe la nouveauté 12.3 mentionnée dans le titre — les **curseurs sur prepared statements** —, dont les détails côté routines stockées relèvent du §8.5.

---

## Le principe

Une requête préparée dissocie deux phases :

1. **Préparation** — la requête, écrite avec des **marqueurs** à la place des valeurs (`… WHERE id = ?`), est analysée et planifiée **une fois** ;
2. **Exécution** — on **lie** les valeurs aux marqueurs et on exécute, éventuellement **plusieurs fois** avec des valeurs différentes.

La structure de la requête est ainsi **figée**, et les valeurs voyagent **séparément**. C'est cette séparation qui fonde les deux bénéfices ci-dessous.

---

## Les deux bénéfices

- **Sécurité** — les valeurs liées ne sont **jamais interprétées comme du SQL** : c'est la défense de référence contre les injections (§17.8), quel que soit le contenu de la valeur.
- **Performance** — pour une requête exécutée de nombreuses fois (insertion en boucle, requête répétée), on prépare **une fois** et l'on réutilise le plan, évitant de ré-analyser le SQL à chaque appel ; le protocole binaire d'exécution est en outre plus compact.

---

## Côté client ou côté serveur ?

Distinction essentielle, déjà croisée dans le chapitre : une requête peut être préparée **côté serveur** (réelle) ou **côté client** (émulée).

- **Côté serveur (réelle)** — la requête est envoyée au serveur, qui l'analyse et renvoie un **identifiant de statement** ; à l'exécution, seules les **valeurs** transitent (protocole binaire). Le statement est **réutilisable** et peut être mis en cache. Coût : un aller-retour pour la préparation.
- **Côté client (émulée)** — le pilote **insère lui-même** les valeurs (échappées) dans le texte SQL, puis envoie une requête ordinaire (protocole texte). Un seul aller-retour par exécution, pas de cache serveur ; la protection contre les injections est néanmoins assurée si le pilote fait correctement son travail.

Les pilotes diffèrent par leur **défaut** : PDO **émule** par défaut (`ATTR_EMULATE_PREPARES`, §17.1.1) ; mysql2 distingue `query` (texte) et `execute` (réelle, §17.1.4) ; le pilote Go n'utilise pas la préparation serveur si `interpolateParams=true` (§17.1.5) ; Connector/J l'active via `useServerPrepStmts` (§17.1.3). En règle générale : **réelle** pour les requêtes répétées (cache), **émulée** acceptable pour les requêtes ponctuelles.

---

## Mécanique au niveau du protocole

Côté serveur, la préparation s'appuie sur des commandes dédiées du protocole : `COM_STMT_PREPARE` (préparer, obtenir un identifiant), `COM_STMT_EXECUTE` (exécuter avec les paramètres), `COM_STMT_CLOSE` (libérer). Une optimisation existe : MariaDB (depuis la 10.2) permet d'envoyer `COM_STMT_PREPARE` et `COM_STMT_EXECUTE` **d'un seul tenant** (*pipelining*), ce qui accélère la première exécution, et un identifiant de statement valant **-1** indique de réutiliser la requête précédemment préparée. Les bons connecteurs exploitent ces mécanismes pour limiter les allers-retours.

---

## Les marqueurs selon le pilote (rappel)

Le §17.1 l'a montré, le **marqueur** varie d'un écosystème à l'autre — mais le principe est invariant : **passer les valeurs en paramètres, jamais par concaténation**.

| Marqueur | Pilotes |
|----------|---------|
| `?` (positionnel) | JDBC/Connector-J, Go, mysql2 / connecteur `mariadb`, PDO, MariaDB Connector/Python |
| `%s` / `%(nom)s` | PyMySQL, mysql-connector-python |
| `:nom` (nommé) | PDO, SQLAlchemy (`text()`) |
| `@nom` (nommé) | ADO.NET / MySqlConnector |

---

## PREPARE / EXECUTE en SQL

MariaDB expose aussi les requêtes préparées **directement en SQL**, utiles dans les routines stockées et le SQL dynamique :

```sql
PREPARE stmt FROM 'SELECT id, nom FROM produits WHERE prix < ?';
SET @max = 50.00;
EXECUTE stmt USING @max;
DEALLOCATE PREPARE stmt;
```

C'est l'équivalent SQL de ce que les connecteurs font côté serveur : préparer, lier, exécuter, libérer.

---

## Curseurs sur prepared statements (🆕 12.3)

Voici la nouveauté mise en avant dans le titre. Jusqu'alors, un curseur de routine stockée devait porter sur un **`SELECT` statique** : sa requête ne pouvait pas être composée dynamiquement (le contournement consistait à créer dynamiquement une vue, via un *prepared statement*, puis à ouvrir le curseur sur cette vue). 

Depuis **MariaDB 12.3**, un curseur peut être déclaré **sur un *prepared statement***, ce qui ouvre l'usage du **SQL dynamique au sein des routines stockées**. Le curseur est lié à un **nom de *prepared statement***, qui doit avoir été défini par `PREPARE` **avant l'ouverture** du curseur. La clause `FOR` du `DECLARE CURSOR` accepte donc désormais, au choix, un `SELECT` ou un nom de *prepared statement* :

```sql
-- Forme conceptuelle (exemple complet au §8.5.1)
DECLARE cur CURSOR FOR mon_statement_prepare;   -- nom d'un PREPARE
```

Cette fonctionnalité (introduite en 12.3.1) lève une limitation forte : jusque-là, parcourir **ligne à ligne** le résultat d'une requête **dynamique** dans une routine stockée n'était pas possible directement — un curseur exigeait un `SELECT` statique, et `EXECUTE … INTO` ne ramenait qu'une seule ligne. C'était un frein pour les routines stockées et la **migration depuis Oracle**, qui dispose de curseurs sur SQL dynamique. La mise en œuvre détaillée côté routines stockées — ordonnancement `DECLARE`/`PREPARE`/`OPEN`/`FETCH`/`CLOSE`, et la variante **`SYS_REFCURSOR`** avec `max_open_cursors` — est traitée au §8.5.1 et §8.5.2.

---

## Bonnes pratiques

- **Toujours paramétrer** — c'est la règle de sécurité du §17.8, et le réflexe par défaut quel que soit le pilote.
- **Préférer les requêtes préparées réelles** pour une exécution répétée (bénéfice du cache de plan), tout en gardant à l'esprit le coût de l'aller-retour de préparation.
- **Mettre en cache et réutiliser** les *prepared statements* — via `cachePrepStmts`/`useServerPrepStmts` (Connector/J) ou le cache du pilote — plutôt que de les re-préparer à chaque fois.
- **Ne pas préparer inutilement dans une boucle** : préparer une fois, exécuter en boucle.
- **Libérer** les statements (ou s'en remettre au cache/pool) ; en SQL, `DEALLOCATE PREPARE`.
- **Connaître le défaut de son pilote** (émulé vs réel) et l'ajuster selon le profil de charge (§17.1.x).

---

## Ce qu'il faut retenir

- Une requête préparée **dissocie préparation (structure figée) et exécution (valeurs liées)** : d'où la **sécurité** (anti-injection, §17.8) et la **performance** sur les requêtes répétées.
- Distinguer **côté serveur** (réelle, réutilisable, protocole binaire) et **côté client** (émulée, un aller-retour, sans cache) ; les pilotes diffèrent par leur défaut (§17.1.x).
- Protocole : `COM_STMT_PREPARE`/`EXECUTE`/`CLOSE`, avec *pipelining* possible (depuis 10.2) ; en SQL : `PREPARE`/`EXECUTE`/`DEALLOCATE`.
- **Marqueurs** variables selon le pilote (`?`, `%s`, `:nom`, `@nom`) — mais paramétrer **toujours**.
- 🆕 **12.3** : un **curseur peut porter sur un *prepared statement***, permettant le **SQL dynamique** dans les routines stockées (utile pour la migration Oracle) ; détails au §8.5.1/§8.5.2.
- En pratique : préparer une fois, **réutiliser/mettre en cache**, ne pas préparer en boucle, libérer ensuite.

⏭️ [Fonctionnalités Avancées](/18-fonctionnalites-avancees/README.md)
