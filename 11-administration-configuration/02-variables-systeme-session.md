🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.2 Variables système et de session

> **Niveau** : Avancé  
> **Durée estimée** : 1.5-2 heures  
> **Prérequis** :
> - Section 11.1 (Fichiers de configuration)
> - Compréhension des concepts client/serveur
> - Expérience SQL de base

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Distinguer** les variables système globales des variables de session
- **Consulter** les valeurs des variables avec SHOW VARIABLES et SELECT @@
- **Modifier** les variables de manière temporaire (SET) et permanente (my.cnf)
- **Comprendre** la différence entre variables dynamiques et statiques
- **Anticiper** l'impact des modifications sur les performances et la stabilité
- **Appliquer** les bonnes pratiques de gestion des variables en production
- **Exploiter** les nouvelles variables MariaDB 11.8

---

## Introduction

Les **variables système** constituent le cœur de la configuration dynamique de MariaDB. Contrairement aux fichiers de configuration qui nécessitent un redémarrage, les variables permettent d'ajuster le comportement du serveur **à chaud**, offrant une flexibilité essentielle en production.

### Pourquoi les variables sont cruciales

```
Configuration statique (my.cnf)
        ↓
Variables système au démarrage
        ↓
Modifications dynamiques (SET)
        ↓
Comportement en temps réel
```

Les variables contrôlent **tous les aspects** de MariaDB :
- 💾 **Mémoire** : Buffer pools, caches, limites
- 🔒 **Sécurité** : Authentification, chiffrement, privilèges
- ⚡ **Performance** : Concurrence, I/O, optimiseur
- 🔄 **Réplication** : Binlogs, GTID, formats
- 📊 **Logging** : Niveaux, destinations, verbosité

---

## Architecture des variables

### Trois niveaux de scope

MariaDB utilise un système de **scope hiérarchique** pour les variables :

```
┌─────────────────────────────────────┐
│   VARIABLES COMPILÉES               │
│   (Valeurs par défaut hardcodées)   │
└─────────────────────────────────────┘
            ↓
┌─────────────────────────────────────┐
│   VARIABLES GLOBALES                │
│   (Affectent tout le serveur)       │
│   @@global.variable_name            │
└─────────────────────────────────────┘
            ↓
┌─────────────────────────────────────┐
│   VARIABLES DE SESSION              │
│   (Affectent une connexion)         │
│   @@session.variable_name           │
│   @@variable_name (défaut)          │
└─────────────────────────────────────┘
```

### Variables globales

Les variables **globales** affectent le comportement général du serveur et s'appliquent à **toutes les nouvelles connexions**.

**Caractéristiques** :
- ✅ Modifiables uniquement par les utilisateurs avec privilège `SUPER` ou `SYSTEM_VARIABLES_ADMIN`
- ✅ Affectent toutes les **futures** connexions
- ❌ N'affectent **pas** les connexions existantes (sauf si la variable a aussi un scope session)
- ⚠️ Perdues au redémarrage si non sauvegardées dans `my.cnf`

**Exemples typiques** :
- `max_connections` : Nombre maximal de connexions simultanées
- `innodb_buffer_pool_size` : Taille du buffer pool InnoDB
- `binlog_format` : Format des binary logs
- `thread_pool_size` : Taille du thread pool

```sql
-- Consulter une variable globale
SELECT @@global.max_connections;

-- Ou
SHOW GLOBAL VARIABLES LIKE 'max_connections';
```

### Variables de session

Les variables **de session** affectent uniquement la **connexion courante**.

**Caractéristiques** :
- ✅ Modifiables par n'importe quel utilisateur pour sa propre session
- ✅ Permettent des comportements personnalisés par connexion
- ✅ Héritent de la valeur globale à l'ouverture de la session
- ❌ Perdues à la fermeture de la connexion

**Exemples typiques** :
- `sql_mode` : Mode SQL de la session
- `autocommit` : Activation du commit automatique
- `foreign_key_checks` : Vérification des clés étrangères
- `unique_checks` : Vérification des contraintes UNIQUE

```sql
-- Consulter une variable de session
SELECT @@session.sql_mode;

-- Forme abrégée (session par défaut)
SELECT @@sql_mode;

-- Ou
SHOW SESSION VARIABLES LIKE 'sql_mode';
```

### Variables à double scope

Certaines variables possèdent **à la fois** un scope global et session :

```sql
-- Exemple avec sql_mode
-- Valeur globale (pour nouvelles connexions)
SHOW GLOBAL VARIABLES LIKE 'sql_mode';

-- Valeur de session (connexion courante)
SHOW SESSION VARIABLES LIKE 'sql_mode';
```

**Comportement** :
1. La valeur **globale** définit la valeur par défaut pour les nouvelles sessions
2. Chaque session **hérite** de cette valeur globale à sa création
3. Modifier la valeur **session** n'affecte que la connexion courante
4. Modifier la valeur **globale** n'affecte que les **futures** connexions

**Illustration** :

```sql
-- Session A se connecte
-- Elle hérite: @@session.sql_mode = @@global.sql_mode

-- Admin modifie la globale
SET GLOBAL sql_mode = 'STRICT_ALL_TABLES';

-- Session A n'est PAS affectée (garde son ancienne valeur)
-- Session B (nouvelle connexion) hérite de la nouvelle valeur
```

---

## Classification des variables

### Variables dynamiques vs statiques

#### Variables dynamiques

Les variables **dynamiques** peuvent être modifiées **sans redémarrer** le serveur via `SET GLOBAL` ou `SET SESSION`.

```sql
-- Exemple : max_connections (dynamique)
SET GLOBAL max_connections = 500;
-- ✅ Effet immédiat, pas de redémarrage nécessaire
```

**Avantages** :
- ⚡ Effet immédiat
- 🔧 Ajustement à chaud en production
- 🧪 Tests rapides de configurations

**Attention** :
- ⚠️ Changements **non persistants** (perdus au redémarrage)
- ⚠️ Nécessite privilèges élevés
- ⚠️ Peut déstabiliser un système en production

#### Variables statiques

Les variables **statiques** nécessitent un **redémarrage** complet du serveur pour être modifiées.

```sql
-- Exemple : innodb_log_file_size (statique)
SET GLOBAL innodb_log_file_size = 2147483648;
-- ❌ ERREUR: Variable is a read only variable
```

**Pour modifier** :

```ini
# 1. Modifier my.cnf
[mysqld]
innodb_log_file_size = 2G

# 2. Redémarrer MariaDB
sudo systemctl restart mariadb
```

**Variables statiques courantes** :
- `datadir` : Répertoire des données
- `innodb_log_file_size` : Taille des fichiers redo log
- `innodb_page_size` : Taille des pages InnoDB
- `lower_case_table_names` : Sensibilité à la casse des noms de tables
- `bind_address` : Adresse d'écoute réseau

### Identifier le type de variable

```sql
-- Vérifier si une variable est dynamique
SELECT
    VARIABLE_NAME,
    VARIABLE_SCOPE,
    VARIABLE_TYPE,
    VARIABLE_COMMENT
FROM information_schema.SYSTEM_VARIABLES
WHERE VARIABLE_NAME = 'max_connections';
```

**Méthode empirique** :

```sql
-- Tenter de modifier
SET GLOBAL ma_variable = 'nouvelle_valeur';

-- Si succès → dynamique
-- Si erreur "read only variable" → statique
```

---

## Consultation des variables

### Syntaxe SELECT @@

La syntaxe `SELECT @@variable_name` permet une consultation rapide et directe :

```sql
-- Variable de session (défaut)
SELECT @@sql_mode;

-- Variable globale (explicite)
SELECT @@global.max_connections;

-- Variable de session (explicite)
SELECT @@session.autocommit;

-- Multiple variables
SELECT
    @@global.max_connections AS max_conn,
    @@global.innodb_buffer_pool_size AS buffer_pool,
    @@session.sql_mode AS current_sql_mode;
```

**Avantages** :
- ✅ Rapide et concis
- ✅ Intégrable dans des requêtes complexes
- ✅ Compatible avec WHERE, JOIN, etc.

**Limitations** :
- ❌ Nécessite de connaître le nom exact
- ❌ Pas de filtrage par motif (LIKE)

### SHOW VARIABLES

La commande `SHOW VARIABLES` permet une consultation plus flexible :

```sql
-- Toutes les variables (attention : liste très longue)
SHOW VARIABLES;

-- Filtrage par motif
SHOW VARIABLES LIKE 'innodb%';
SHOW VARIABLES LIKE '%timeout%';

-- Scope explicite
SHOW GLOBAL VARIABLES LIKE 'max_connections';
SHOW SESSION VARIABLES LIKE 'sql_mode';
```

**Avantages** :
- ✅ Filtrage avec motifs (% et _)
- ✅ Vue d'ensemble rapide
- ✅ Lisible dans le client CLI

### INFORMATION_SCHEMA.GLOBAL_VARIABLES

Pour des requêtes SQL avancées :

```sql
-- Recherche flexible
SELECT
    VARIABLE_NAME,
    VARIABLE_VALUE
FROM information_schema.GLOBAL_VARIABLES
WHERE VARIABLE_NAME LIKE '%buffer%'
ORDER BY VARIABLE_NAME;

-- Analyse de configuration
SELECT
    VARIABLE_NAME,
    VARIABLE_VALUE,
    CAST(VARIABLE_VALUE AS UNSIGNED) / 1024 / 1024 AS value_mb
FROM information_schema.GLOBAL_VARIABLES
WHERE VARIABLE_NAME IN (
    'innodb_buffer_pool_size',
    'key_buffer_size',
    'tmp_table_size'
);
```

**Vues disponibles** :
- `GLOBAL_VARIABLES` : Variables globales
- `SESSION_VARIABLES` : Variables de la session courante
- `SYSTEM_VARIABLES` : Méta-informations (scope, type, etc.)

---

## Modification des variables

### SET SESSION : Modification locale

Affecte **uniquement** la connexion courante.

```sql
-- Syntaxe complète
SET SESSION sql_mode = 'STRICT_TRANS_TABLES';

-- Syntaxe abrégée (équivalente)
SET sql_mode = 'STRICT_TRANS_TABLES';

-- Multiples variables
SET
    SESSION sql_mode = 'TRADITIONAL',
    SESSION autocommit = 0,
    SESSION foreign_key_checks = 0;
```

**Cas d'usage** :
- 🔧 Import/export avec désactivation temporaire des checks
- 🧪 Tests de comportement SQL
- 🎯 Optimisations spécifiques à une tâche

**Exemple pratique - Import rapide** :

```sql
-- Optimisation pour import massif
SET SESSION foreign_key_checks = 0;
SET SESSION unique_checks = 0;
SET SESSION sql_log_bin = 0;  -- Désactive binlog pour cette session

-- Import
SOURCE /tmp/large_dump.sql;

-- Réactiver les checks
SET SESSION foreign_key_checks = 1;
SET SESSION unique_checks = 1;
SET SESSION sql_log_bin = 1;
```

### SET GLOBAL : Modification serveur

Affecte toutes les **futures** connexions.

```sql
-- Syntaxe
SET GLOBAL max_connections = 500;

-- Ou avec @@
SET @@global.max_connections = 500;
```

⚠️ **ATTENTION** : Nécessite le privilège `SUPER` ou `SYSTEM_VARIABLES_ADMIN`.

**Impact** :

```sql
-- Avant modification
-- Connexion A: @@session.max_allowed_packet = 16M

SET GLOBAL max_allowed_packet = 67108864;  -- 64M

-- Après modification
-- Connexion A: @@session.max_allowed_packet = 16M (inchangé)
-- Connexion B (nouvelle): @@session.max_allowed_packet = 64M (nouvelle valeur)
```

### SET PERSIST (MariaDB 10.5+)

`SET PERSIST` modifie la variable **et la sauvegarde** automatiquement.

```sql
-- Modification persistante
SET PERSIST max_connections = 500;

-- Équivalent à :
-- 1. SET GLOBAL max_connections = 500;
-- 2. Ajouter dans un fichier de config auto-généré
```

**Fichier de persistance** : `/var/lib/mysql/mysqld-auto.cnf` (format JSON)

```json
{
  "Version": 1,
  "mysql_dynamic_variables": {
    "max_connections": {
      "Value": "500",
      "Metadata": {
        "Timestamp": 1702468800,
        "User": "root",
        "Host": "localhost"
      }
    }
  }
}
```

**Avantages** :
- ✅ Persistance automatique sans éditer `my.cnf`
- ✅ Historique des modifications (timestamp, user)
- ✅ Facilite l'automatisation

**Limitations** :
- ⚠️ Fichier JSON peut devenir volumineux
- ⚠️ Moins lisible qu'un `my.cnf` commenté
- ⚠️ Peut entrer en conflit avec `my.cnf`

💡 **Bonne pratique** : En production, préférez `my.cnf` avec versionning Git pour la traçabilité et la documentation.

### SET PERSIST_ONLY

Sauvegarde sans appliquer immédiatement (utile pour variables statiques).

```sql
-- Pour une variable statique (nécessite redémarrage)
SET PERSIST_ONLY innodb_log_file_size = 2147483648;

-- La valeur sera appliquée au prochain redémarrage
```

---

## Portée et héritage des variables

### Héritage global → session

```sql
-- État initial
SHOW GLOBAL VARIABLES LIKE 'sql_mode';
-- STRICT_TRANS_TABLES

-- Nouvelle connexion hérite de la globale
SELECT @@session.sql_mode;
-- STRICT_TRANS_TABLES

-- Modification session (locale)
SET sql_mode = 'TRADITIONAL';

-- Vérification
SELECT @@session.sql_mode;  -- TRADITIONAL
SELECT @@global.sql_mode;   -- STRICT_TRANS_TABLES (inchangé)
```

### Impact des modifications globales

```sql
-- Session A et B connectées
-- @@session.max_allowed_packet = 16M (pour les deux)

-- Admin modifie la globale
SET GLOBAL max_allowed_packet = 67108864;  -- 64M

-- Résultat :
-- Session A : @@session.max_allowed_packet = 16M (inchangé)
-- Session B : @@session.max_allowed_packet = 16M (inchangé)
-- Session C (nouvelle connexion) : @@session.max_allowed_packet = 64M
```

💡 **Implication production** : Pour appliquer une modification globale à toutes les connexions actives, il faut :
1. Modifier la variable globale
2. Attendre que les connexions existantes se ferment naturellement, OU
3. Redémarrer le serveur (si critique), OU
4. Forcer la déconnexion des sessions (KILL)

---

## Variables critiques pour la production

### Mémoire et performance

```sql
-- Buffer Pool InnoDB (variable la plus importante)
SELECT @@global.innodb_buffer_pool_size / 1024 / 1024 / 1024 AS buffer_pool_gb;
-- Recommandation : 70-80% de la RAM sur serveur dédié

-- Connexions
SELECT @@global.max_connections;
-- Dimensionner selon votre charge (100-500 typique, 1000+ haute charge)

-- 🆕 MariaDB 11.8 : Espace temporaire
SELECT
    @@global.max_tmp_space_usage / 1024 / 1024 / 1024 AS max_tmp_per_session_gb,
    @@global.max_total_tmp_space_usage / 1024 / 1024 / 1024 AS max_tmp_total_gb;
```

### Sécurité

```sql
-- 🆕 MariaDB 11.8 : TLS par défaut
SELECT @@global.require_secure_transport;  -- ON = connexions TLS obligatoires

-- Authentification
SELECT @@global.plugin_dir;
SELECT @@global.authentication_policy;

-- Timeout connexion
SELECT @@global.connect_timeout;
SELECT @@global.interactive_timeout;
SELECT @@global.wait_timeout;
```

### Charset et collation

```sql
-- 🆕 MariaDB 11.8 : utf8mb4 par défaut
SELECT
    @@global.character_set_server,    -- utf8mb4
    @@global.collation_server;        -- utf8mb4_uca1400_ai_ci

-- Par session (peut différer)
SELECT
    @@session.character_set_client,
    @@session.character_set_connection,
    @@session.character_set_results;
```

### Réplication

```sql
-- Binary logs
SELECT
    @@global.log_bin,
    @@global.binlog_format,
    @@global.expire_logs_days;

-- GTID
SELECT
    @@global.gtid_strict_mode,
    @@global.gtid_current_pos;

-- Semi-synchrone
SELECT
    @@global.rpl_semi_sync_master_enabled,
    @@global.rpl_semi_sync_slave_enabled;
```

---

## Bonnes pratiques de gestion des variables

### 1. Tester avant d'appliquer

```sql
-- ❌ MAUVAIS : Modifier directement en production
SET GLOBAL innodb_buffer_pool_size = 34359738368;  -- 32 GB

-- ✅ BON : Tester sur environnement de staging
-- 1. Staging
SET GLOBAL innodb_buffer_pool_size = 34359738368;
-- 2. Observer les métriques pendant 24-48h
-- 3. Si OK, appliquer en production pendant maintenance
```

### 2. Documenter les changements

```sql
-- ✅ BON : Commenter les changements
SET GLOBAL max_connections = 500;
-- Raison: Pic de trafic anticipé Black Friday
-- Date: 2025-12-13
-- Admin: dba@example.com
-- Rollback: SET GLOBAL max_connections = 200;
```

### 3. Sauvegarder avant modification critique

```bash
# Sauvegarder la configuration actuelle
mariadb -e "SHOW GLOBAL VARIABLES" > /backup/variables-$(date +%Y%m%d).txt

# Modifier
mariadb -e "SET GLOBAL max_connections = 500"

# En cas de problème, consulter la sauvegarde
less /backup/variables-20251213.txt | grep max_connections
```

### 4. Utiliser des variables de session pour les tests

```sql
-- ✅ BON : Tester sur une session
SET SESSION sql_mode = 'STRICT_ALL_TABLES';
-- Tester les requêtes...
-- Si KO, déconnexion = rollback automatique

-- ❌ RISQUÉ : Modifier la globale directement
SET GLOBAL sql_mode = 'STRICT_ALL_TABLES';
-- Affecte toutes les nouvelles connexions !
```

### 5. Monitorer après modification

```sql
-- Après SET GLOBAL max_connections = 500
-- Surveiller :
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Max_used_connections';
SHOW STATUS LIKE 'Connection_errors%';

-- Si Max_used_connections proche de max_connections → augmenter encore
-- Si Connection_errors_max_connections > 0 → connexions refusées !
```

---

## Variables dynamiques importantes

### Variables pouvant être modifiées à chaud

| Variable | Scope | Impact | Recommandation |
|----------|-------|--------|----------------|
| `max_connections` | Global | Nb connexions max | Adapter à la charge |
| `max_allowed_packet` | Global + Session | Taille max packet | 16M-1G selon besoin |
| `sql_mode` | Global + Session | Comportement SQL | STRICT en prod |
| `slow_query_log` | Global | Activation slow log | ON pour diagnostic |
| `long_query_time` | Global + Session | Seuil slow query | 1-5s selon usage |
| `max_tmp_space_usage` 🆕 | Global | Limite tmp/session | 10G typique |
| `max_total_tmp_space_usage` 🆕 | Global | Limite tmp globale | 100G typique |

### Exemples de modifications courantes

```sql
-- Augmenter temporairement les connexions (pic de trafic)
SET GLOBAL max_connections = 1000;

-- Activer le slow query log (diagnostic)
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 2;

-- Ajuster max_allowed_packet pour un import
SET GLOBAL max_allowed_packet = 1073741824;  -- 1 GB

-- 🆕 Limiter l'espace temporaire (protéger le disque)
SET GLOBAL max_tmp_space_usage = 10737418240;  -- 10 GB
```

---

## Variables statiques critiques

### Variables nécessitant un redémarrage

| Variable | Impact | Modification |
|----------|--------|--------------|
| `datadir` | Emplacement données | Arrêt + move + restart |
| `innodb_log_file_size` | Taille redo log | my.cnf + restart |
| `innodb_page_size` | Taille pages | REBUILD complet |
| `lower_case_table_names` | Casse tables | my.cnf + restart |
| `bind_address` | Adresse écoute | my.cnf + restart |

### Procédure de modification variable statique

```bash
# 1. Vérifier la valeur actuelle
mariadb -e "SELECT @@global.innodb_log_file_size"

# 2. Éditer my.cnf
sudo vim /etc/mysql/mariadb.conf.d/20-innodb.cnf
# innodb_log_file_size = 2G

# 3. Valider la syntaxe
mysqld --verbose --help > /dev/null

# 4. Planifier maintenance
# 5. Redémarrer
sudo systemctl restart mariadb

# 6. Vérifier
mariadb -e "SELECT @@global.innodb_log_file_size / 1024 / 1024 / 1024"
```

---

## Nouveautés MariaDB 11.8

### Variables liées à l'espace temporaire

```sql
-- 🆕 Limite par session (10 GB par défaut)
SELECT @@global.max_tmp_space_usage;
SET GLOBAL max_tmp_space_usage = 10737418240;  -- 10 GB

-- 🆕 Limite totale serveur (100 GB par défaut)
SELECT @@global.max_total_tmp_space_usage;
SET GLOBAL max_total_tmp_space_usage = 107374182400;  -- 100 GB
```

**Cas d'usage** : Éviter la saturation de `/tmp` avec des requêtes créant d'énormes tables temporaires.

### Variables charset par défaut

```sql
-- 🆕 utf8mb4 par défaut
SELECT @@global.character_set_server;
-- utf8mb4

SELECT @@global.collation_server;
-- utf8mb4_uca1400_ai_ci (UCA 14.0.0)
```

### Variables TIMESTAMP étendu

```sql
-- 🆕 Support TIMESTAMP jusqu'en 2106 (résolution Y2038)
-- Pas de variable spécifique, comportement automatique
-- Les nouveaux champs TIMESTAMP utilisent 7 octets au lieu de 4
```

### Variables InnoDB optimisées

```sql
-- 🆕 Construction index efficace
SELECT @@global.innodb_alter_copy_bulk;
SET GLOBAL innodb_alter_copy_bulk = ON;  -- Défaut ON depuis 11.8
```

---

## ✅ Points clés à retenir

- **Deux scopes** : Global (serveur entier) vs Session (connexion)
- **Héritage** : Session hérite de Global à la création de la connexion
- **Dynamiques vs statiques** : Certaines modifiables à chaud, d'autres nécessitent restart
- **Consultation** : `SELECT @@variable`, `SHOW VARIABLES`, `INFORMATION_SCHEMA`
- **Modification** : `SET GLOBAL` (futures connexions), `SET SESSION` (connexion courante)
- **Persistance** : `SET PERSIST` auto-sauvegarde, mais préférer `my.cnf` + Git en prod
- **Privilèges** : `SET GLOBAL` nécessite `SUPER` ou `SYSTEM_VARIABLES_ADMIN`
- **Impact** : `SET GLOBAL` n'affecte **pas** les connexions existantes
- **Tests** : Toujours tester sur staging avant prod
- **Documentation** : Commenter **pourquoi**, pas seulement **quoi**
- 🆕 **MariaDB 11.8** : `max_tmp_space_usage`, `max_total_tmp_space_usage`, utf8mb4 défaut
- **Monitoring** : Surveiller `Max_used_connections`, `Connection_errors`, `Threads_connected`

---

## 🔗 Ressources et références

- [📖 Documentation officielle - Server System Variables](https://mariadb.com/kb/en/server-system-variables/)
- [📖 Documentation officielle - SET Statement](https://mariadb.com/kb/en/set/)
- [📖 Documentation officielle - SHOW VARIABLES](https://mariadb.com/kb/en/show-variables/)
- [📖 Documentation officielle - Dynamic System Variables](https://mariadb.com/kb/en/dynamic-system-variables/)
- [📖 MariaDB 11.8 Release Notes](https://mariadb.com/kb/en/mariadb-1180-release-notes/)
- [🔧 MySQL Tuner](https://github.com/major/MySQLTuner-perl) - Analyse et recommandations

---

## ➡️ Section suivante

**[11.2.1 SHOW VARIABLES et SET](./02.1-show-variables-set.md)** : Exploration détaillée des commandes de consultation et modification des variables, avec exemples pratiques et cas d'usage production.

---

**💡 Conseil final** : Les variables système sont un outil puissant mais potentiellement dangereux. Comme le dit le principe de Spider-Man : "Un grand pouvoir implique de grandes responsabilités". Testez, documentez, surveillez ! 🕷️🚀

⏭️ [SHOW VARIABLES et SET](/11-administration-configuration/02.1-show-variables-set.md)
