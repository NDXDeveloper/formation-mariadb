🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.5 Compatibilité des applications

Réussir une migration vers MariaDB 12.3 ne se résume pas à transférer les données et à démarrer le serveur. Une fois la base en ligne, c'est l'**application** — son code, ses pilotes, son ORM, ses requêtes — qui doit continuer à fonctionner *correctement*. Or une application repose sur des dizaines de contrats implicites avec le moteur : le protocole réseau, le dialecte SQL accepté, la sémantique d'exécution des requêtes, le mécanisme d'authentification, les jeux de caractères. Chacun de ces contrats peut se rompre lors d'une migration, parfois silencieusement.

La distinction essentielle à garder en tête est celle entre **incompatibilité bloquante** (la requête échoue avec une erreur — facile à détecter) et **incompatibilité silencieuse** (la requête s'exécute sans erreur mais produit un résultat ou un comportement différent — beaucoup plus dangereuse). Cette section recense les couches concernées, puis détaille les changements de comportement propres à la 12.3 qui peuvent affecter une application migrée depuis la 11.8 ou depuis MySQL.

> 🔗 La validation concrète de cette compatibilité (jeux de tests, environnements, capture/rejeu) est traitée dans la section suivante, §19.6 *Tests de compatibilité*. La présente section décrit *quoi* vérifier ; la §19.6 décrit *comment* le vérifier.

---

## 19.5.1 Les couches de la compatibilité applicative

La compatibilité d'une application avec MariaDB se joue sur plusieurs couches superposées. Une migration peut être parfaitement transparente sur certaines et problématique sur d'autres :

| Couche | Question posée | Risque typique |
|--------|----------------|----------------|
| **Protocole réseau** | Le client sait-il dialoguer avec le serveur ? | Très faible (protocole MySQL compatible) |
| **Pilote / connecteur** | La version du connecteur gère-t-elle les fonctionnalités du serveur ? | Modéré (versions anciennes, plugins d'auth) |
| **Dialecte SQL** | Les requêtes sont-elles syntaxiquement acceptées ? | Modéré (mots réservés, fonctions, `sql_mode`) |
| **Sémantique d'exécution** | Les requêtes produisent-elles le *même résultat* ? | Élevé et silencieux (isolation, collations, NULL) |
| **Authentification** | Le client parvient-il à se connecter ? | Bloquant (plugins d'auth incompatibles) |
| **Jeu de caractères / collation** | Les comparaisons et tris sont-ils identiques ? | Élevé et silencieux (utf8mb4, UCA 14.0.0) |

L'ordre n'est pas anodin : les couches du haut produisent des erreurs franches et immédiates, faciles à corriger ; les couches du bas produisent des écarts de comportement qui ne se révèlent qu'en production, sur des cas particuliers.

---

## 19.5.2 Compatibilité du protocole et des pilotes

MariaDB implémente le **protocole client/serveur MySQL**, ce qui garantit qu'au niveau le plus bas, la quasi-totalité des clients MySQL peuvent se connecter à MariaDB. C'est cette compatibilité historique qui rend les migrations « techniquement » indolores dans la plupart des cas.

Toutefois, les deux projets ont divergé depuis plusieurs années, et MariaDB maintient désormais ses **propres connecteurs officiels**, distincts de ceux de MySQL :

| Langage / techno | Connecteur recommandé (MariaDB) | Connecteur MySQL utilisable ? |
|------------------|----------------------------------|-------------------------------|
| Java | MariaDB Connector/J (`jdbc:mariadb://`) | Oui, avec réserves |
| Python | MariaDB Connector/Python, ou PyMySQL | Oui (mysql-connector-python) |
| Node.js | `mariadb` (Connector/Node.js) | Oui (`mysql2`) |
| C / C++ | MariaDB Connector/C | Oui (libmysqlclient) |
| ODBC | MariaDB Connector/ODBC | Oui |
| .NET | MySqlConnector (recommandé) | MySql.Data possible |

> 🔗 Le détail des connecteurs par langage est couvert au chapitre 17, *Intégration et Développement* (§17.1).

Trois points d'attention pratiques au niveau pilote :

**La chaîne de connexion.** Le passage à Connector/J impose le schéma d'URL `jdbc:mariadb://` en lieu et place de `jdbc:mysql://`. Une application qui code en dur l'URL MySQL devra être ajustée, ou conserver le pilote MySQL (qui sait se connecter à MariaDB).

**La détection de version (*version sniffing*).** Certaines bibliothèques et certains ORM inspectent la chaîne de version retournée par le serveur (`SELECT VERSION()`) pour adapter leur comportement. MariaDB retourne une chaîne contenant `MariaDB` (par ex. `12.3.2-MariaDB`), parfois précédée d'un préfixe de compatibilité hérité. Un code qui suppose un format de version MySQL strict peut prendre de mauvaises décisions. Il faut s'assurer que la couche d'abstraction reconnaît bien MariaDB comme un moteur distinct.

**La gestion des plugins d'authentification**, traitée plus bas (§19.5.5) : un connecteur trop ancien peut ne pas savoir négocier un plugin moderne.

La recommandation générale est de **migrer le pilote en même temps que le serveur** et d'épingler une version testée, plutôt que de s'appuyer sur la rétrocompatibilité d'un connecteur ancien.

---

## 19.5.3 Compatibilité du dialecte SQL

Au-delà du protocole, l'application envoie des requêtes SQL qui doivent être acceptées par le moteur. Plusieurs facteurs influencent cette acceptation.

### Le rôle de `sql_mode`

La variable `sql_mode` est le levier central de compatibilité du dialecte. Elle contrôle la rigueur de l'interprétation SQL et active des comportements de compatibilité avec d'autres SGBD. Un même corpus de requêtes peut passer ou échouer selon le `sql_mode` actif.

```sql
-- Vérifier le mode actif
SELECT @@sql_mode;

-- Activer le mode de compatibilité Oracle pour une session
SET SESSION sql_mode = 'ORACLE';
```

> 🔗 Le détail des modes SQL est traité en §11.3. Le mode `ORACLE` débloque de nombreuses fonctionnalités de compatibilité (voir ci-dessous).

Un piège fréquent lors d'une migration : le `sql_mode` de l'ancienne instance et celui de la nouvelle diffèrent. Une requête tolérée sous un mode permissif (par exemple une insertion avec valeur tronquée) peut échouer sous un mode strict. La première chose à comparer est donc le `sql_mode` source vs cible.

### Mots réservés et fonctions

MariaDB 12.3 introduit de nouveaux mots-clés (notamment liés aux Optimizer Hints, §15.15, et aux nouvelles fonctionnalités). Un identifiant de colonne ou de table qui coïncide avec un nouveau mot réservé devra être protégé par des accents graves (`` `colonne` ``). C'est un cas rare mais réel lors d'une montée de version.

Côté fonctions, la 12.3 *ajoute* sans retirer dans la plupart des cas, ce qui limite le risque. Les fonctions de compatibilité Oracle (`TO_DATE`, `TO_NUMBER`, `TRUNC`, `TO_CHAR` avec format `FM`, §3.7.1) facilitent au contraire la migration des applications conçues pour Oracle.

### Fonctionnalités de compatibilité héritées d'autres SGBD

Pour les applications provenant d'Oracle, la 12.3 consolide un ensemble de fonctionnalités qui réduisent fortement la réécriture nécessaire : syntaxe `( + )` pour les jointures externes (§3.3.5), tableaux associatifs (§8.1.4), `SYS_REFCURSOR` (§8.5.2), type `XMLTYPE` basique (§2.2.6). Pour MySQL 8, l'ajout de `caching_sha2_password` (§10.5.5) et la prise en charge de certaines fonctions GIS récentes (§19.1.1) lissent la transition.

> 🔗 La cartographie complète des fonctionnalités de compatibilité est détaillée en §19.2.1 (depuis Oracle) et §19.1 (depuis MySQL).

---

## 19.5.4 Compatibilité sémantique : les pièges silencieux

C'est la couche la plus délicate. Les requêtes s'exécutent sans erreur, mais leur **résultat ou leur comportement transactionnel diffère**. Ces écarts ne sont détectés ni par un linter, ni par un test de connexion : ils n'apparaissent qu'à l'usage, souvent sur des cas limites.

### Isolation transactionnelle : `innodb_snapshot_isolation`

L'**isolation par instantané** (*snapshot isolation*) est l'un des points sémantiques les plus structurants. La variable `innodb_snapshot_isolation` est **activée par défaut** — non pas depuis la 12.3, mais **depuis la 11.6.2**, elle l'est donc déjà en 11.8 (§6.9). Sous le niveau `REPEATABLE READ` (le défaut d'InnoDB), une transaction qui tente de modifier une ligne ayant été modifiée *et validée* par une autre transaction après l'établissement de son instantané reçoit une **erreur** explicite :

```
ERROR 1020 (HY000): Record has changed since last read in table 't1'; try restarting transaction
```

Le code d'erreur est `1020` (`ER_CHECKREAD`), et le message invite explicitement à **rejouer la transaction** (*try restarting transaction*).

⚠️ **Périmètre de migration.** Comme ce paramètre est **déjà à `ON` en 11.8**, une migration `11.8 → 12.3` n'introduit **aucun changement** de ce côté — les deux versions se comportent à l'identique. Le sujet concerne en revanche les migrations depuis une version **antérieure à 11.6.2** (MariaDB 10.6 / 10.11 / 11.4, où le défaut était `OFF`) ou depuis **MySQL** : là, le code passe d'une modification silencieuse contre la dernière version validée à une **erreur** explicite. Ce passage à un véritable *snapshot isolation* prévient certaines anomalies (mises à jour perdues, *write skew*), mais il **introduit une classe d'erreurs** absente auparavant.

Conséquence pratique : une application transactionnelle venant d'un tel socle doit prévoir une logique de **capture de l'erreur et de rejeu de la transaction**. C'est l'archétype de l'incompatibilité silencieuse-devenue-bloquante : le code « fonctionnait » avant, il lève maintenant une exception qui, si elle n'est pas gérée, remonte jusqu'à l'utilisateur. Les applications peuvent, si nécessaire, restaurer l'ancien comportement en positionnant explicitement `innodb_snapshot_isolation = OFF`, le temps d'adapter le code.

### Collations et comparaisons de chaînes

Depuis la 11.8, le jeu de caractères par défaut est `utf8mb4` avec des collations basées sur **UCA 14.0.0** (§11.11). Les règles de comparaison et de tri d'une collation UCA peuvent différer de celles d'une collation plus ancienne (`utf8mb4_general_ci`, par exemple). Cela a trois effets potentiellement silencieux :

- Une clause `WHERE col = 'valeur'` peut désormais correspondre (ou ne plus correspondre) à des lignes selon la sensibilité à la casse et aux accents de la nouvelle collation.
- Un `ORDER BY` peut produire un ordre différent — visible dans une pagination ou un export.
- Une contrainte `UNIQUE` peut considérer comme identiques (ou distinctes) deux valeurs qui ne l'étaient pas auparavant, provoquant un échec d'insertion ou, à l'inverse, l'acceptation de doublons fonctionnels.

La migration depuis `utf8` (3 octets) vers `utf8mb4` mérite aussi attention : longueur maximale des index, occupation disque, et émojis/caractères sur 4 octets qui étaient auparavant rejetés ou tronqués.

### Logique ternaire et valeurs NULL

Le traitement des `NULL` suit la logique ternaire SQL standard (§4.6). Une application migrée depuis un SGBD au comportement différent sur les `NULL` (comparaisons, concaténations, agrégats) peut observer des écarts. Ce point relève surtout des migrations *inter-SGBD* (Oracle, SQL Server) plus que d'une montée de version MariaDB.

### Conversions implicites et `sql_mode` strict

Sous un `sql_mode` strict, les conversions implicites tolérées ailleurs (chaîne vide vers `0`, date invalide vers `'0000-00-00'`) sont rejetées. Une application qui s'appuyait sur cette tolérance verra ses `INSERT`/`UPDATE` échouer. Là encore, la comparaison du `sql_mode` source/cible est le premier réflexe.

---

## 19.5.5 Compatibilité de l'authentification

L'échec de connexion lié à un plugin d'authentification incompatible est l'un des problèmes de migration les plus courants — et les plus déroutants, car il bloque l'application *avant* toute requête.

| Plugin | Origine | Statut en MariaDB 12.3 | Cas d'usage migration |
|--------|---------|------------------------|------------------------|
| `mysql_native_password` | Historique MySQL/MariaDB | Disponible (déprécié côté MySQL 8) | Compatibilité maximale, anciens clients |
| `ed25519` | Natif MariaDB | Recommandé | Authentification moderne sécurisée |
| `caching_sha2_password` | MySQL 8 (défaut) | **Pris en charge (nouveauté 12.x)** | Migration d'apps configurées pour MySQL 8 |
| `PARSEC` | MariaDB | Disponible | Authentification renforcée |

L'ajout de **`caching_sha2_password`** (§10.5.5) est une avancée majeure pour la compatibilité : c'est le plugin par défaut de MySQL 8, et de nombreux connecteurs récents le négocient automatiquement. Avant cette prise en charge, une application configurée pour MySQL 8 pouvait échouer à se connecter à MariaDB. Désormais, la transition se fait sans modifier la méthode d'authentification de l'application.

Deux autres points liés à la connexion :

**Connecteur trop ancien.** Un plugin moderne (`ed25519`, `caching_sha2_password`) exige un connecteur capable de le négocier. Un pilote vétuste peut échouer même si le serveur le propose — argument supplémentaire pour mettre à jour les connecteurs (§19.5.2).

**TLS zéro-configuration** (depuis 11.8, §10.7.3). MariaDB peut désormais négocier le chiffrement TLS sans configuration explicite. Une application qui se connectait en clair peut désormais établir une connexion chiffrée, ce qui est souhaitable, mais il faut vérifier le comportement de validation de certificat côté client (mode `VERIFY_CA` / `VERIFY_IDENTITY`) pour éviter un échec inattendu.

---

## 19.5.6 Changements de comportement spécifiques à la 12.3 (vs 11.8)

Pour une application migrée *depuis la 11.8*, plusieurs changements de comportement de la série 12.x doivent être passés en revue. Ils sont récapitulés ici du point de vue applicatif :

- **`innodb_snapshot_isolation`** n'est en réalité **pas** un changement propre au passage `11.8 → 12.3` : le paramètre est **déjà activé par défaut en 11.8** (depuis la 11.6.2). Il ne représente une nouveauté que pour les migrations depuis une version antérieure (MariaDB ≤ 11.4) ou depuis MySQL — voir §19.5.4. À ce titre, il n'a pas sa place dans la liste des deltas spécifiques à la 12.3, mais reste mentionné ici car c'est une confusion fréquente.
- **Variables système retirées** : `big_tables`, `large_page_size`, `storage_engine` (§11.2.3). Tout script de démarrage, fichier `my.cnf` ou commande `SET` référençant ces variables provoquera désormais une erreur. À auditer dans la configuration *et* dans le code applicatif qui positionnerait des variables de session.
- **Noms de contraintes de clés étrangères uniques par table seulement** (§18.12), et non plus à l'échelle de la base. Ce changement est globalement *assouplissant*, mais un outil de migration de schéma ou un ORM qui supposait l'unicité à l'échelle de la base peut générer du DDL inadapté.
- **Nouveau binlog intégré à InnoDB** (§11.5.4), introduit en 12.3 mais **optionnel** : il s'active explicitement via `binlog_storage_engine=innodb` (en plus de `log_bin`) et reste **désactivé par défaut**. Tant qu'on ne l'active pas, rien ne change. Une fois activé, il est transparent pour l'application elle-même, mais impacte les outils externes de lecture du binlog (CDC, Debezium — §20.8) qui doivent prendre en charge le nouveau format.

> 🔗 La checklist exhaustive et ordonnée des changements de comportement 11.8 → 12.3 fait l'objet d'une section dédiée, §19.10. La présente liste en isole les éléments qui touchent la couche applicative.

---

## 19.5.7 Compatibilité des ORM et frameworks

Les ORM masquent en partie le SQL, mais ils ne l'éliminent pas : ils *génèrent* du SQL et du DDL spécifiques au moteur ciblé. Leur compatibilité dépend donc de deux choix : le **dialecte** sélectionné et la **version** de l'ORM/connecteur.

- **Hibernate (Java)** : utiliser le dialecte `MariaDBDialect` (et sa variante versionnée correspondante) plutôt qu'un dialecte MySQL générique, afin que le SQL généré exploite correctement les fonctionnalités du moteur (§17.3.1).
- **SQLAlchemy (Python)** : préférer le dialecte `mariadb+pymysql://` (ou `mariadb+mariadbconnector://`) qui reconnaît MariaDB comme moteur distinct (§17.3.2).
- **Entity Framework Core (.NET)** : le fournisseur Pomelo (`Pomelo.EntityFrameworkCore.MySql`) prend en charge MariaDB ; vérifier la version alignée sur la 12.3 (§17.3.5).
- **Prisma, Sequelize, Django ORM** : épingler les versions de l'ORM *et* du connecteur sous-jacent, et exécuter la suite de tests d'intégration après migration (§17.3).

Le réflexe de migration côté ORM est triple : **sélectionner le bon dialecte**, **épingler les versions** (ORM + connecteur), puis **regénérer et relire le SQL/DDL** produit, en particulier les migrations de schéma automatiques qui peuvent émettre du DDL légèrement différent selon la version cible.

---

## 19.5.8 Démarche d'évaluation de la compatibilité

Avant de basculer en production, une évaluation structurée de la compatibilité applicative s'organise autour de quatre temps :

1. **Inventaire** — recenser, pour chaque application : connecteurs et leurs versions, ORM et dialectes, méthode d'authentification, jeu de caractères/collation utilisés, et `sql_mode` attendu. Cet inventaire est la matière première de l'analyse.
2. **Analyse statique** — comparer le `sql_mode` source/cible, repérer les identifiants entrant en collision avec de nouveaux mots réservés, et identifier les requêtes dépendantes de comportements transactionnels (`REPEATABLE READ` avec écritures concurrentes) ou de collations (comparaisons, `ORDER BY`, contraintes `UNIQUE`).
3. **Validation dynamique** — exécuter l'application contre une instance 12.3 de pré-production. La fonctionnalité de **capture et rejeu de charge** de MaxScale (Workload Capture/Replay, §14.5) permet de rejouer un trafic de production réel contre la nouvelle version et de comparer les résultats — un outil précieux pour débusquer les incompatibilités silencieuses. La construction des jeux de tests proprement dite est détaillée en §19.6.
4. **Surveillance post-bascule** — après la mise en production, surveiller les journaux d'erreurs applicatifs et le slow query log pour détecter les erreurs nouvelles (notamment les erreurs de conflit d'instantané) et les éventuelles régressions de performance liées à des plans d'exécution différents.

---

## Points clés à retenir

- La compatibilité applicative est une préoccupation *distincte* du succès technique de la migration ; elle se joue sur plusieurs couches, du protocole jusqu'aux collations.
- Les incompatibilités **bloquantes** (authentification, dialecte sous `sql_mode` strict) sont gênantes mais visibles ; les incompatibilités **silencieuses** (isolation, collations, conversions implicites) sont les plus dangereuses car elles ne se révèlent qu'en production.
- Les vrais changements de comportement `11.8 → 12.3` à auditer côté applicatif sont les **variables système retirées** (`big_tables`, `large_page_size`, `storage_engine`), les **noms de contraintes de clés étrangères désormais uniques par table** et le **binlog InnoDB** (optionnel). Attention : `innodb_snapshot_isolation = ON` est **déjà le défaut en 11.8** (depuis la 11.6.2) — ce n'est donc *pas* un delta `11.8 → 12.3`, mais bien un changement pour qui migre depuis MariaDB ≤ 11.4 ou MySQL (erreur `1020`, logique de rejeu).
- L'ajout de `caching_sha2_password` simplifie nettement la migration des applications conçues pour MySQL 8.
- Penser à **migrer connecteurs et dialectes ORM en même temps que le serveur**, en épinglant des versions testées.
- Auditer la configuration et le code pour les **variables retirées** (`big_tables`, `large_page_size`, `storage_engine`).

> 🔗 **Pour aller plus loin** : §19.6 *Tests de compatibilité* (mise en œuvre concrète de la validation), §19.10 *Migration 11.8 → 12.3* (checklist exhaustive des changements de comportement), §6.9 *Snapshot Isolation*, §11.3 *Modes SQL*, et le chapitre 17 *Intégration et Développement* pour le détail des connecteurs et ORM.

⏭️ [Tests de compatibilité](/19-migration-compatibilite/06-tests-compatibilite.md)
