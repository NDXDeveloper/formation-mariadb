🔝 Retour au [Sommaire](/SOMMAIRE.md)

# A.1 - Termes MariaDB Essentiels

> **Niveau** : Tous niveaux (Référence)  
> **Type** : Glossaire technique - Définitions des concepts fondamentaux  
> **Mise à jour** : Décembre 2025 - MariaDB 11.8 LTS

---

## 📖 Introduction

Cette section définit les **termes techniques essentiels** utilisés dans l'écosystème MariaDB. Chaque terme est présenté avec sa définition, son contexte d'utilisation et des exemples pratiques lorsque pertinent.

### Légende des Niveaux
- 🟢 **Débutant** : Concept fondamental
- 🟡 **Intermédiaire** : Nécessite des bases MariaDB
- 🔴 **Avancé** : Concept technique spécialisé
- 🟣 **Expert** : Utilisation en production avancée

---

## A

### ACID 🟢
**Propriétés fondamentales garantissant la fiabilité des transactions dans une base de données.**

Les quatre propriétés ACID sont :
- **Atomicity** (Atomicité) : Une transaction est exécutée complètement ou pas du tout
- **Consistency** (Cohérence) : Les données respectent toujours les contraintes définies
- **Isolation** : Les transactions concurrentes ne s'affectent pas mutuellement
- **Durability** (Durabilité) : Une transaction validée reste permanente même en cas de panne

📌 **Contexte** : InnoDB garantit les propriétés ACID, contrairement à MyISAM  
🔗 **Voir aussi** : Transaction, InnoDB, Isolation Levels, COMMIT, ROLLBACK

---

### Adaptive Hash Index 🔴
**Structure d'index automatique créée par InnoDB en mémoire pour accélérer les recherches fréquentes sur des colonnes spécifiques.**

InnoDB surveille les patterns d'accès aux index B-Tree et construit automatiquement des index hash en mémoire pour les pages les plus consultées, offrant des performances O(1) pour les lookups exacts.

📌 **Contexte** : Activé par défaut, bénéfique pour les workloads OLTP avec accès répétitifs  
💡 **Configuration** : `innodb_adaptive_hash_index = ON/OFF`  
🔗 **Voir aussi** : InnoDB, Buffer Pool, B-Tree Index, Hash Index

---

### ALTER TABLE 🟡
**Commande SQL permettant de modifier la structure d'une table existante.**

Permet d'ajouter/supprimer des colonnes, modifier des types de données, créer/supprimer des index, changer le moteur de stockage, etc.

📌 **Contexte** : Opération potentiellement bloquante sur grandes tables  
💡 **Exemple** :
```sql
ALTER TABLE users ADD COLUMN age INT;
ALTER TABLE orders ENGINE=InnoDB;
```
🆕 **MariaDB 11.8** : `innodb_alter_copy_bulk` améliore les performances  
🔗 **Voir aussi** : Online DDL, pt-online-schema-change, gh-ost

---

### Application Time Period Tables 🔴 🆕
**Tables permettant de gérer des périodes de validité applicatives avec support SQL natif pour les opérations temporelles.**

Extension des System-Versioned Tables pour gérer des périodes métier (validité d'un contrat, durée d'un abonnement) directement dans le schéma de la table.

📌 **Contexte** : Nouveauté MariaDB 11.8 pour gérer des données temporelles métier  
💡 **Exemple** : Gérer l'historique des prix avec des périodes de validité  
🔗 **Voir aussi** : System-Versioned Tables, Temporal Tables, Bitemporal

---

### Aria 🟡
**Moteur de stockage MariaDB conçu comme successeur de MyISAM, offrant crash recovery et support des transactions.**

Aria améliore MyISAM avec journalisation des transactions (crash-safe), meilleure concurrence et compression. Par défaut pour les tables système internes de MariaDB.

📌 **Contexte** : Utilisé pour tables temporaires et système, alternative à MyISAM  
💡 **Caractéristiques** : Crash recovery, pas de support FK  
🔗 **Voir aussi** : MyISAM, InnoDB, Storage Engines

---

### Auto-Increment 🟢
**Propriété de colonne générant automatiquement des valeurs numériques séquentielles uniques.**

Utilisé principalement pour les clés primaires. La valeur est incrémentée à chaque INSERT.

📌 **Contexte** : Essentiel pour les identifiants uniques  
💡 **Exemple** :
```sql
CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100)
);
```
⚠️ **Attention** : Les valeurs ne sont pas réutilisées après DELETE  
🔗 **Voir aussi** : PRIMARY KEY, SEQUENCE, UUID

---

## B

### Backup Stage 🟡 🆕
**Mécanisme de verrous coordonnés permettant des sauvegardes cohérentes sans bloquer complètement la base.**

MariaDB 11.8 étend le support de BACKUP STAGE dans Mariabackup pour des sauvegardes plus efficaces et moins bloquantes.

📌 **Contexte** : Utilisé par Mariabackup pour sauvegardes à chaud  
💡 **Commandes** : `BACKUP STAGE START/BLOCK_DDL/BLOCK_COMMIT/END`  
🔗 **Voir aussi** : Mariabackup, Hot Backup, FLUSH TABLES WITH READ LOCK

---

### Binary Log (Binlog) 🟡
**Journal contenant toutes les modifications de données (INSERT, UPDATE, DELETE) sous forme d'événements.**

Le binlog sert à :
- La réplication (envoi des changements aux replicas)
- La restauration point-in-time (PITR)
- L'audit des modifications

📌 **Contexte** : Essentiel pour réplication et recovery  
💡 **Formats** : STATEMENT, ROW, MIXED  
💡 **Configuration** :
```ini
log_bin = /var/log/mysql/mariadb-bin
binlog_format = ROW
expire_logs_days = 7
```
🔗 **Voir aussi** : Réplication, GTID, PITR, mysqlbinlog

---

### B-Tree Index 🟡
**Structure d'index par défaut dans MariaDB, organisée en arbre équilibré permettant des recherches efficaces en O(log n).**

Les B-Tree (Balanced Tree) maintiennent les données triées et permettent :
- Recherches par égalité et par plage
- ORDER BY et GROUP BY efficaces
- Accès séquentiel rapide

📌 **Contexte** : Type d'index par défaut, polyvalent et performant  
💡 **Usage** : Colonnes dans WHERE, JOIN, ORDER BY, GROUP BY  
🔗 **Voir aussi** : Index, EXPLAIN, Covering Index, Composite Index

---

### Buffer Pool 🟡
**Zone mémoire InnoDB stockant en cache les données et index pour réduire les accès disque.**

C'est le paramètre de performance le plus important d'InnoDB. Idéalement configuré à 70-80% de la RAM disponible sur un serveur dédié.

📌 **Contexte** : Configuration critique pour les performances InnoDB  
💡 **Configuration** :
```ini
innodb_buffer_pool_size = 8G
innodb_buffer_pool_instances = 8
```
💡 **Monitoring** :
```sql
SHOW STATUS LIKE 'Innodb_buffer_pool%';
```
🔗 **Voir aussi** : InnoDB, innodb_buffer_pool_size, Cache Hit Ratio

---

## C

### Certification-Based Replication 🔴
**Mécanisme de réplication synchrone utilisé par Galera Cluster basé sur la certification des writesets.**

Chaque transaction est validée localement, puis son writeset est diffusé aux autres nœuds. La certification vérifie l'absence de conflits avant le commit final.

📌 **Contexte** : Base du fonctionnement de Galera Cluster  
💡 **Avantage** : Réplication synchrone multi-master sans conflits  
🔗 **Voir aussi** : Galera Cluster, Writeset, Quorum, Multi-Master

---

### Change Data Capture (CDC) 🟡
**Processus de capture et suivi des modifications de données pour propagation vers d'autres systèmes.**

Utilise le binary log pour détecter et diffuser les changements en quasi temps-réel vers des systèmes externes (Data Warehouse, Kafka, etc.).

📌 **Contexte** : Architecture Event-Driven, intégration de données  
💡 **Outils** : Debezium, Maxwell, Canal  
🔗 **Voir aussi** : Binary Log, Event-Driven Architecture, Debezium

---

### Charset (Character Set) 🟢
**Ensemble de caractères supporté pour stocker du texte dans une base de données.**

Détermine quels caractères peuvent être stockés (latin1, utf8, utf8mb4, etc.).

📌 **Contexte** : Crucial pour l'internationalisation  
💡 **Défaut MariaDB 11.8** : `utf8mb4` (support emoji, caractères asiatiques)  
💡 **Exemple** :
```sql
CREATE DATABASE mydb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```
🆕 **MariaDB 11.8** : utf8mb4 par défaut avec UCA 14.0.0  
🔗 **Voir aussi** : Collation, utf8mb4, Unicode

---

### Checkpoint 🔴
**Point de synchronisation où InnoDB écrit les pages modifiées du Buffer Pool vers le disque.**

Les checkpoints permettent de réduire le temps de recovery en cas de crash en s'assurant que les données sont périodiquement persistées.

📌 **Contexte** : Processus automatique InnoDB, crucial pour la durabilité  
💡 **Configuration** : Contrôlé par `innodb_io_capacity` et flush settings  
🔗 **Voir aussi** : InnoDB, Redo Log, Buffer Pool, Crash Recovery

---

### Collation 🟢
**Règles de comparaison et tri pour un charset donné.**

Détermine comment les chaînes sont comparées (sensibilité à la casse, accents, ordre de tri).

📌 **Contexte** : Impacte ORDER BY, comparaisons, index  
💡 **Exemples** :
- `utf8mb4_unicode_ci` : Insensible à la casse (CI = Case Insensitive)
- `utf8mb4_bin` : Sensible à la casse (binaire)
- `utf8mb4_general_ci` : Tri général, moins précis

💡 **Usage** :
```sql
SELECT * FROM users WHERE name = 'MARIE' COLLATE utf8mb4_bin;
```
🔗 **Voir aussi** : Charset, utf8mb4, UCA

---

### ColumnStore 🔴
**Moteur de stockage analytique orienté colonnes pour requêtes OLAP et data warehousing.**

Optimisé pour :
- Agrégations sur grandes volumétries
- Lecture de quelques colonnes sur des milliards de lignes
- Compression élevée (10-50x)
- Requêtes analytiques complexes

📌 **Contexte** : Alternative à InnoDB pour workloads analytiques  
💡 **Architecture** : Stockage par colonnes, compression, parallélisme massif  
🔗 **Voir aussi** : OLAP, Data Warehouse, InnoDB (OLTP)

---

### Commit 🟢
**Opération validant définitivement une transaction et rendant ses modifications permanentes.**

Une fois un COMMIT exécuté :
- Les modifications sont visibles par les autres transactions
- Les changements sont garantis durables (propriété Durability)
- La transaction est terminée

📌 **Contexte** : Commande fondamentale du contrôle transactionnel  
💡 **Exemple** :
```sql
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```
🔗 **Voir aussi** : ROLLBACK, Transaction, ACID, Autocommit

---

### Composite Index 🟡
**Index portant sur plusieurs colonnes combinées.**

L'ordre des colonnes est crucial : l'index est utilisable si la requête filtre sur les colonnes de gauche à droite.

📌 **Contexte** : Optimise les requêtes multi-colonnes  
💡 **Exemple** :
```sql
CREATE INDEX idx_name_city ON users(last_name, first_name, city);
-- Utilisable pour : WHERE last_name = 'Dupont'
-- Utilisable pour : WHERE last_name = 'Dupont' AND first_name = 'Jean'
-- NON utilisable pour : WHERE city = 'Paris' (colonne pas en début d'index)
```
🔗 **Voir aussi** : Index, B-Tree, Covering Index, Index Prefix

---

### Concurrency 🟡
**Capacité à gérer plusieurs transactions simultanées sans corruption de données.**

MariaDB utilise :
- **MVCC** pour lectures concurrentes sans blocage
- **Verrous** pour coordonner les écritures
- **Niveaux d'isolation** pour équilibrer performance/cohérence

📌 **Contexte** : Fondamental pour les systèmes multi-utilisateurs  
🔗 **Voir aussi** : MVCC, Locking, Isolation Levels, Deadlock

---

### Constraint 🟢
**Règle d'intégrité imposée sur les données d'une table.**

Types de contraintes :
- **PRIMARY KEY** : Identifiant unique
- **FOREIGN KEY** : Référence vers autre table
- **UNIQUE** : Valeur unique dans la colonne
- **NOT NULL** : Valeur obligatoire
- **CHECK** : Condition personnalisée
- **DEFAULT** : Valeur par défaut

📌 **Contexte** : Garantit la qualité et cohérence des données  
💡 **Exemple** :
```sql
CREATE TABLE orders (
  id INT PRIMARY KEY,
  user_id INT NOT NULL,
  amount DECIMAL(10,2) CHECK (amount > 0),
  FOREIGN KEY (user_id) REFERENCES users(id)
);
```
🔗 **Voir aussi** : PRIMARY KEY, FOREIGN KEY, Referential Integrity

---

### Covering Index 🟡
**Index contenant toutes les colonnes nécessaires à une requête, évitant l'accès à la table.**

Quand un index "couvre" toutes les colonnes du SELECT et du WHERE, MariaDB peut répondre sans lire les lignes de la table (index-only scan).

📌 **Contexte** : Optimisation majeure pour performances lecture  
💡 **Exemple** :
```sql
CREATE INDEX idx_cover ON users(last_name, first_name, email);
-- Cette requête utilise uniquement l'index :
SELECT email FROM users WHERE last_name = 'Dupont' AND first_name = 'Marie';
```
💡 **EXPLAIN** : Indiqué par "Using index" dans Extra  
🔗 **Voir aussi** : Index, B-Tree, EXPLAIN, Composite Index

---

### Crash Recovery 🟡
**Processus automatique de restauration de la cohérence après un arrêt brutal.**

InnoDB utilise :
- **Redo Log** : Rejoue les transactions committées non écrites sur disque
- **Undo Log** : Annule les transactions non committées

📌 **Contexte** : Garantit ACID même après crash système  
💡 **Automatique** : Au redémarrage de MariaDB  
🔗 **Voir aussi** : InnoDB, Redo Log, Undo Log, ACID

---

## D

### Deadlock 🟡
**Situation où deux transactions ou plus s'attendent mutuellement, créant un blocage circulaire.**

Exemple :
- Transaction A verrouille ligne 1, attend ligne 2
- Transaction B verrouille ligne 2, attend ligne 1
→ Deadlock !

📌 **Contexte** : InnoDB détecte et résout automatiquement en annulant une transaction  
💡 **Prévention** :
- Ordre cohérent d'accès aux ressources
- Transactions courtes
- Index appropriés pour réduire les verrous

💡 **Détection** :
```sql
SHOW ENGINE INNODB STATUS; -- Section LATEST DETECTED DEADLOCK
```
🔗 **Voir aussi** : Locking, Transaction, MVCC, Isolation Levels

---

### Dynamic Columns 🔴
**Fonctionnalité MariaDB permettant de stocker des colonnes flexibles dans un BLOB.**

Alternative au JSON pour gérer des attributs variables. Moins utilisé depuis l'amélioration du support JSON.

📌 **Contexte** : Legacy, préférer JSON pour nouveaux projets  
🔗 **Voir aussi** : JSON, BLOB, Schema Flexibility

---

## E

### Encryption at Rest 🟡
**Chiffrement des données sur disque pour protéger contre l'accès physique non autorisé.**

MariaDB supporte :
- Chiffrement transparent des tablespaces InnoDB
- Chiffrement des binary logs
- Chiffrement des tables temporaires

📌 **Contexte** : Sécurité, conformité RGPD/HIPAA  
💡 **Configuration** :
```ini
plugin_load_add = file_key_management
file_key_management_filename = /etc/mysql/encryption/keyfile
innodb_encrypt_tables = ON
innodb_encrypt_log = ON
```
🔗 **Voir aussi** : SSL/TLS, Key Management, Security

---

### Event 🟡
**Tâche SQL planifiée exécutée automatiquement par le serveur.**

Équivalent des CRON jobs mais directement dans MariaDB.

📌 **Contexte** : Maintenance automatisée, traitements périodiques  
💡 **Exemple** :
```sql
CREATE EVENT cleanup_old_logs
ON SCHEDULE EVERY 1 DAY
DO DELETE FROM logs WHERE created_at < NOW() - INTERVAL 90 DAY;
```
💡 **Activation** :
```sql
SET GLOBAL event_scheduler = ON;
```
🔗 **Voir aussi** : Stored Procedures, Triggers, Event Scheduler

---

### EXPLAIN 🟡
**Commande affichant le plan d'exécution d'une requête pour analyse des performances.**

Montre :
- Les tables accédées et dans quel ordre
- Les index utilisés
- Le nombre estimé de lignes examinées
- Les optimisations appliquées

📌 **Contexte** : Outil essentiel d'optimisation de requêtes  
💡 **Exemple** :
```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 123;
```
💡 **EXPLAIN ANALYZE** : Version étendue avec temps réels d'exécution  
🔗 **Voir aussi** : Query Optimization, Index, ANALYZE TABLE

---

## F

### Foreign Key (FK) 🟢
**Contrainte créant une relation référentielle entre deux tables.**

Assure l'intégrité référentielle : une valeur FK doit exister comme PK dans la table référencée.

📌 **Contexte** : Modélisation relationnelle, intégrité des données  
💡 **Exemple** :
```sql
CREATE TABLE orders (
  id INT PRIMARY KEY,
  user_id INT,
  FOREIGN KEY (user_id) REFERENCES users(id)
    ON DELETE CASCADE
    ON UPDATE RESTRICT
);
```
💡 **Actions** :
- CASCADE : Propagation automatique
- RESTRICT : Blocage si référencé
- SET NULL : Mise à NULL

⚠️ **Attention** : Supporte uniquement par InnoDB, pas MyISAM/Aria  
🔗 **Voir aussi** : PRIMARY KEY, Referential Integrity, InnoDB

---

### Full-Text Index 🟡
**Index spécialisé pour recherche textuelle en langage naturel.**

Permet des recherches de type "Google" dans le texte avec :
- Pertinence des résultats (score)
- Support des stop words
- Recherche en mode naturel ou booléen

📌 **Contexte** : Recherche dans articles, descriptions, commentaires  
💡 **Exemple** :
```sql
CREATE FULLTEXT INDEX ft_description ON products(description);
SELECT * FROM products 
WHERE MATCH(description) AGAINST('smartphone rapide' IN NATURAL LANGUAGE MODE);
```
💡 **Modes** : NATURAL LANGUAGE, BOOLEAN, WITH QUERY EXPANSION  
🔗 **Voir aussi** : Index, Search, MyISAM, InnoDB FT

---

## G

### Galera Cluster 🟣
**Solution de clustering multi-master synchrone pour MariaDB.**

Caractéristiques :
- **Multi-master** : Écriture sur tous les nœuds
- **Synchrone** : Réplication en temps réel avec certification
- **Haute disponibilité** : Pas de point unique de défaillance
- **Consistent** : Cohérence forte entre nœuds

📌 **Contexte** : HA enterprise, tolérance aux pannes  
💡 **Architecture** : Minimum 3 nœuds pour quorum  
💡 **Technologies** : Certification-based replication, Virtual Synchrony  
🔗 **Voir aussi** : Replication, High Availability, Quorum, SST, IST

---

### Generated Column 🟡
**Colonne dont la valeur est calculée automatiquement à partir d'une expression.**

Deux types :
- **VIRTUAL** : Calculée à la lecture (pas stockée)
- **STORED** : Calculée à l'écriture et stockée

📌 **Contexte** : Dénormalisation contrôlée, optimisation requêtes  
💡 **Exemple** :
```sql
CREATE TABLE products (
  price DECIMAL(10,2),
  tax DECIMAL(10,2),
  total_price DECIMAL(10,2) AS (price + tax) STORED,
  price_category VARCHAR(10) AS (
    CASE 
      WHEN price < 100 THEN 'Low'
      WHEN price < 1000 THEN 'Medium'
      ELSE 'High'
    END
  ) VIRTUAL
);
```
💡 **Indexation possible** sur colonnes STORED et VIRTUAL  
🔗 **Voir aussi** : Virtual Column, Computed Column, Indexing

---

### GTID (Global Transaction Identifier) 🟡
**Identifiant unique global pour chaque transaction dans un environnement de réplication.**

Format : `domain_id-server_id-sequence_number`  
Exemple : `0-100-5432`

📌 **Contexte** : Simplifie la configuration réplication et failover  
💡 **Avantages** :
- Pas besoin de connaître la position binlog exacte
- Failover automatique facilité
- Identification unique des transactions

💡 **Configuration** :
```ini
gtid_strict_mode = ON
log_slave_updates = ON
```
🔗 **Voir aussi** : Replication, Binary Log, Failover, Position-based Replication

---

## H

### High Availability (HA) 🟣
**Capacité d'un système à rester opérationnel avec un minimum de downtime.**

Stratégies MariaDB :
- **Galera Cluster** : Multi-master synchrone
- **Master-Slave Replication** + failover automatique
- **MaxScale** : Load balancing et routing intelligent
- **Geo-distribution** : Réplication entre datacenters

📌 **Contexte** : Production critique, SLA élevés  
💡 **Métriques** : Availability = Uptime / (Uptime + Downtime)  
🔗 **Voir aussi** : Galera, Replication, MaxScale, Failover

---

### HNSW (Hierarchical Navigable Small Worlds) 🔴 🆕
**Algorithme d'index pour recherche vectorielle par plus proches voisins (k-NN).**

Utilisé par MariaDB Vector pour indexer et rechercher efficacement dans des vecteurs de haute dimension (embeddings IA).

📌 **Contexte** : Nouveauté MariaDB 11.8 pour applications IA/ML  
💡 **Performance** : Recherche approximative ultra-rapide en haute dimension  
💡 **Exemple** :
```sql
CREATE TABLE embeddings (
  id INT PRIMARY KEY,
  vector VECTOR(1536) NOT NULL,
  VECTOR INDEX (vector) HNSW(M=16, ef_construction=64)
);
```
🔗 **Voir aussi** : MariaDB Vector, Vector Search, RAG, Semantic Search

---

## I

### InnoDB 🟢
**Moteur de stockage transactionnel par défaut de MariaDB.**

Caractéristiques principales :
- **ACID compliant** : Transactions fiables
- **MVCC** : Lectures non bloquantes
- **Foreign Keys** : Intégrité référentielle
- **Crash Recovery** : Récupération automatique
- **Row-level Locking** : Concurrence optimale

📌 **Contexte** : Moteur recommandé pour 99% des cas d'usage  
💡 **Architecture** : Buffer Pool, Redo/Undo Logs, Tablespaces  
🔗 **Voir aussi** : Storage Engine, ACID, MVCC, Buffer Pool

---

### Index 🟢
**Structure de données accélérant la recherche et le tri dans les tables.**

Types principaux :
- **B-Tree** : Recherche généraliste (défaut)
- **Hash** : Égalité exacte seulement
- **Full-Text** : Recherche textuelle
- **Spatial** : Données géographiques
- **VECTOR (HNSW)** : Recherche vectorielle IA 🆕

📌 **Contexte** : Crucial pour les performances  
💡 **Trade-off** : Accélère SELECT, ralentit INSERT/UPDATE/DELETE  
⚠️ **Attention** : Trop d'index = contre-productif  
🔗 **Voir aussi** : B-Tree, EXPLAIN, Covering Index, Composite Index

---

### Isolation Level 🟡
**Degré de séparation entre transactions concurrentes, équilibrant performance et cohérence.**

Quatre niveaux SQL standard (du moins au plus strict) :
1. **READ UNCOMMITTED** : Dirty reads possibles
2. **READ COMMITTED** : Lectures validées uniquement
3. **REPEATABLE READ** : Lectures reproductibles (défaut InnoDB)
4. **SERIALIZABLE** : Isolation totale

📌 **Contexte** : Configuration globale ou par session  
💡 **Configuration** :
```sql
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```
💡 **Trade-off** : Plus strict = plus sûr mais moins performant  
🔗 **Voir aussi** : Transaction, MVCC, Concurrency, Locking

---

### IST (Incremental State Transfer) 🔴
**Méthode de synchronisation incrémentale entre nœuds Galera Cluster.**

Quand un nœud est légèrement désynchronisé, IST lui envoie uniquement les writesets manquants depuis son dernier état connu (via cache gcache).

📌 **Contexte** : Plus rapide que SST, utilisé si le gap est petit  
💡 **Condition** : Les writesets manquants doivent être dans gcache  
🔗 **Voir aussi** : SST, Galera Cluster, Writeset, State Transfer

---

## J

### JOIN 🟢
**Opération combinant des lignes de plusieurs tables basée sur une condition de relation.**

Types principaux :
- **INNER JOIN** : Intersection (lignes correspondant dans les deux tables)
- **LEFT JOIN** : Toutes lignes table gauche + correspondances table droite
- **RIGHT JOIN** : Toutes lignes table droite + correspondances table gauche
- **CROSS JOIN** : Produit cartésien (toutes combinaisons)

📌 **Contexte** : Fondamental pour requêtes relationnelles  
💡 **Exemple** :
```sql
SELECT u.name, o.total
FROM users u
INNER JOIN orders o ON u.id = o.user_id;
```
🔗 **Voir aussi** : INNER JOIN, LEFT JOIN, Foreign Key, Index

---

### JSON 🟡
**Type de données et fonctions pour stocker et manipuler des documents JSON.**

MariaDB stocke JSON comme texte (longtext) mais fournit :
- Validation syntaxique à l'insertion
- Fonctions d'extraction et manipulation
- Indexation via colonnes virtuelles

📌 **Contexte** : Données semi-structurées, attributs flexibles  
💡 **Exemple** :
```sql
CREATE TABLE products (
  id INT PRIMARY KEY,
  attributes JSON
);
INSERT INTO products VALUES (1, '{"color":"red","size":"L"}');
SELECT JSON_EXTRACT(attributes, '$.color') FROM products;
-- ou raccourci :
SELECT attributes->>'$.color' FROM products;
```
🆕 **MariaDB 11.8** : Amélioration JSON Path Expressions et validation  
🔗 **Voir aussi** : JSON Functions, Virtual Columns, NoSQL

---

## L

### Lock 🟡
**Mécanisme de verrouillage empêchant l'accès concurrent à une ressource.**

Types de verrous InnoDB :
- **Shared Lock (S)** : Lecture partagée, bloque écritures
- **Exclusive Lock (X)** : Écriture exclusive, bloque tout
- **Intention Locks** : Signale intention de verrouiller
- **Row-level Locks** : Verrouillage au niveau ligne
- **Table-level Locks** : Verrouillage table entière

📌 **Contexte** : Gestion automatique par InnoDB, explicite si besoin  
💡 **Explicite** :
```sql
SELECT * FROM orders WHERE id = 123 FOR UPDATE; -- X lock
SELECT * FROM orders WHERE id = 123 FOR SHARE;  -- S lock
```
🔗 **Voir aussi** : MVCC, Deadlock, Isolation Level, Concurrency

---

### LTS (Long Term Support) 🟢
**Version de MariaDB avec support étendu de 3 ans.**

Politique depuis MariaDB 11.4 :
- **LTS** : Version stable, support 3 ans (ex: 11.4, 11.8, 12.3 prévu)
- **Rolling** : Versions trimestrielles, support court

📌 **Contexte** : Production, prévisibilité, stabilité  
💡 **Versions LTS actuelles** :
- 10.6 LTS (jusqu'à 2026)
- 10.11 LTS (jusqu'à 2028)
- 11.4 LTS (jusqu'à 2027)
- **11.8 LTS** (jusqu'à 2028) 🆕

🔗 **Voir aussi** : Rolling Release, Version Policy, Support Cycle

---

## M

### Mariabackup 🟡
**Outil de sauvegarde physique à chaud pour InnoDB et XtraDB.**

Fonctionnalités :
- Sauvegarde complète (full backup)
- Sauvegarde incrémentale
- Sauvegarde sans arrêt du serveur
- Support compression et chiffrement

📌 **Contexte** : Sauvegarde production, alternative à mysqldump  
💡 **Exemple** :
```bash
mariabackup --backup --target-dir=/backup/full --user=root
mariabackup --prepare --target-dir=/backup/full
mariabackup --copy-back --target-dir=/backup/full
```
🆕 **MariaDB 11.8** : Support BACKUP STAGE amélioré  
🔗 **Voir aussi** : Backup, PITR, mysqldump, Hot Backup

---

### MaxScale 🟣
**Proxy intelligent de base de données développé par MariaDB Corporation.**

Fonctionnalités :
- **Load Balancing** : Distribution de charge
- **Read/Write Split** : Routage intelligent
- **Query Routing** : Routage basé sur règles
- **Database Firewall** : Sécurité requêtes
- **Connection Pooling** : Optimisation connexions

📌 **Contexte** : HA, performance, sécurité en production  
💡 **Architecture** : Proxy transparent entre application et MariaDB  
🆕 **Version 25.01** : Workload Capture/Replay, Diff Router  
🔗 **Voir aussi** : High Availability, Load Balancing, ProxySQL

---

### MVCC (Multi-Version Concurrency Control) 🟡
**Mécanisme InnoDB permettant des lectures concurrentes sans verrous.**

Principe :
- Chaque transaction voit une "version" cohérente des données
- Les lectures ne bloquent pas les écritures
- Les écritures ne bloquent pas les lectures
- Implémenté via Undo Log

📌 **Contexte** : Cœur de la concurrence InnoDB  
💡 **Avantage** : Haute performance en lecture/écriture concurrente  
💡 **Fonctionnement** : Undo log conserve les anciennes versions  
🔗 **Voir aussi** : InnoDB, Isolation Level, Undo Log, Concurrency

---

### MyISAM 🟡 🔄
**Ancien moteur de stockage MariaDB, remplacé par InnoDB et Aria.**

Caractéristiques :
- **Pas de transactions** (pas ACID)
- **Table-level locking** (concurrence limitée)
- **Pas de Foreign Keys**
- **Crash non-safe**

📌 **Contexte** : Legacy, éviter pour nouveaux projets  
⚠️ **Déprécié** : Utiliser InnoDB ou Aria à la place  
🔗 **Voir aussi** : InnoDB, Aria, Storage Engine

---

## N

### Normalization 🟢
**Processus de conception de schéma réduisant la redondance et améliorant l'intégrité.**

Formes normales principales :
- **1NF** : Valeurs atomiques (pas de listes)
- **2NF** : Pas de dépendance partielle à la clé
- **3NF** : Pas de dépendance transitive
- **BCNF** : Forme de Boyce-Codd (3NF stricte)

📌 **Contexte** : Conception de schéma optimal  
💡 **Trade-off** : Normalisation vs dénormalisation pour performance  
🔗 **Voir aussi** : Schema Design, Foreign Key, Denormalization

---

## O

### OLAP (Online Analytical Processing) 🟡
**Traitement analytique de grandes volumétries pour reporting et BI.**

Caractéristiques :
- Requêtes complexes avec agrégations
- Lecture de millions/milliards de lignes
- Peu d'écritures concurrentes
- Orienté lecture

📌 **Contexte** : Data Warehouse, Business Intelligence  
💡 **Moteur recommandé** : ColumnStore  
🔗 **Voir aussi** : OLTP, ColumnStore, Data Warehouse, Aggregation

---

### OLTP (Online Transaction Processing) 🟡
**Traitement transactionnel de nombreuses opérations courtes et concurrentes.**

Caractéristiques :
- Transactions courtes et fréquentes
- Lecture/écriture mixte
- Forte concurrence
- Latence faible requise

📌 **Contexte** : Applications web, e-commerce, SaaS  
💡 **Moteur recommandé** : InnoDB  
🔗 **Voir aussi** : OLAP, InnoDB, Transaction, Index

---

### Online DDL 🟡
**Capacité d'exécuter des modifications de schéma sans bloquer la table.**

MariaDB supporte de nombreuses opérations ALTER TABLE online :
- Ajout/suppression d'index
- Ajout de colonnes (certaines conditions)
- Changement de type de colonne (limité)

📌 **Contexte** : Maintenance en production sans downtime  
💡 **Algorithmes** :
- `ALGORITHM=INPLACE` : Pas de copie table (rapide)
- `ALGORITHM=COPY` : Copie complète (lent, bloquant)
- `ALGORITHM=INSTANT` : Instantané (très limité)

💡 **Exemple** :
```sql
ALTER TABLE users ADD COLUMN age INT, ALGORITHM=INPLACE, LOCK=NONE;
```
🆕 **MariaDB 11.8** : Optimistic ALTER TABLE pour réplication  
🔗 **Voir aussi** : ALTER TABLE, gh-ost, pt-online-schema-change

---

## P

### Partitioning 🟡
**Division d'une grande table en sous-ensembles physiques pour améliorer la performance.**

Types de partitionnement :
- **RANGE** : Par intervalles (dates, IDs)
- **LIST** : Par valeurs explicites
- **HASH** : Par fonction de hachage
- **KEY** : Comme HASH mais géré par MariaDB

📌 **Contexte** : Tables volumineuses (millions/milliards lignes)  
💡 **Avantages** :
- Partition pruning (scan uniquement partitions pertinentes)
- Maintenance par partition
- Archivage facilité

💡 **Exemple** :
```sql
CREATE TABLE orders (
  id INT,
  order_date DATE,
  ...
) PARTITION BY RANGE (YEAR(order_date)) (
  PARTITION p2023 VALUES LESS THAN (2024),
  PARTITION p2024 VALUES LESS THAN (2025),
  PARTITION p2025 VALUES LESS THAN (2026)
);
```
🔗 **Voir aussi** : Partition Pruning, Sharding, Table Maintenance

---

### PITR (Point-In-Time Recovery) 🟡
**Restauration de la base à un instant précis grâce aux binary logs.**

Processus :
1. Restaurer dernière sauvegarde complète
2. Rejouer les binary logs jusqu'à l'instant T
3. Base restaurée à l'état exact de l'instant T

📌 **Contexte** : Recovery après erreur humaine, corruption  
💡 **Prérequis** : Binary logs activés et archivés  
💡 **Commande** :
```bash
mysqlbinlog --start-datetime="2025-12-01 14:30:00" \
            --stop-datetime="2025-12-01 14:35:00" \
            binlog.000123 | mysql -u root -p
```
🔗 **Voir aussi** : Binary Log, Backup, Mariabackup, Recovery

---

### Primary Key (PK) 🟢
**Colonne(s) identifiant de manière unique chaque ligne d'une table.**

Propriétés :
- **Unique** : Aucune valeur dupliquée
- **Not NULL** : Valeur obligatoire
- **Indexé automatiquement** : Recherche rapide

📌 **Contexte** : Fondamental en modélisation relationnelle  
💡 **Exemple** :
```sql
CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(255) UNIQUE NOT NULL
);
-- ou clé composite :
CREATE TABLE order_items (
  order_id INT,
  product_id INT,
  PRIMARY KEY (order_id, product_id)
);
```
💡 **InnoDB** : PK = Clustered Index (données triées par PK)  
🔗 **Voir aussi** : Foreign Key, Auto-Increment, Clustered Index

---

### Prepared Statement 🟡
**Requête SQL pré-compilée avec paramètres, exécutable plusieurs fois.**

Avantages :
- **Sécurité** : Protection contre injections SQL
- **Performance** : Requête analysée une seule fois
- **Réutilisabilité** : Exécution avec différents paramètres

📌 **Contexte** : Développement application, sécurité  
💡 **Exemple SQL** :
```sql
PREPARE stmt FROM 'SELECT * FROM users WHERE id = ?';
SET @id = 123;
EXECUTE stmt USING @id;
DEALLOCATE PREPARE stmt;
```
💡 **En PHP (PDO)** :
```php
$stmt = $pdo->prepare('SELECT * FROM users WHERE id = ?');
$stmt->execute([123]);
```
🔗 **Voir aussi** : SQL Injection, Security, Query Cache

---

## Q

### Query Cache 🟡 🔄
**Cache de résultats de requêtes SELECT (déprécié et désactivé).**

⚠️ **Déprécié depuis MySQL 8.0 / MariaDB 10.x** : Causait plus de problèmes que d'avantages (contention, invalidation excessive).

📌 **Alternative** : Caching applicatif (Redis, Memcached)  
🔗 **Voir aussi** : Buffer Pool, Application Cache, Performance

---

### Quorum 🔴
**Majorité de nœuds requis dans Galera Cluster pour maintenir la cohérence.**

Calcul : Quorum = (nombre de nœuds / 2) + 1

📌 **Contexte** : Prévention split-brain dans Galera  
💡 **Exemple** : Cluster 3 nœuds → quorum = 2 nœuds minimum  
💡 **Importance** : Si quorum perdu → cluster en read-only  
🔗 **Voir aussi** : Galera Cluster, Split-Brain, High Availability

---

## R

### RAG (Retrieval-Augmented Generation) 🔴 🆕
**Architecture IA combinant recherche vectorielle et génération de texte par LLM.**

Processus RAG :
1. **Vectorisation** : Convertir données en embeddings
2. **Stockage** : Sauvegarder dans MariaDB Vector
3. **Recherche** : Trouver documents similaires (k-NN)
4. **Génération** : LLM génère réponse basée sur documents

📌 **Contexte** : Applications IA conversationnelles, chatbots intelligents  
💡 **MariaDB 11.8** : Support natif avec Vector type et HNSW index  
🔗 **Voir aussi** : MariaDB Vector, HNSW, Semantic Search, LLM

---

### Redo Log 🟡
**Journal InnoDB enregistrant toutes les modifications avant écriture sur disque.**

Rôle :
- **Durabilité** : Garantit que transactions committées survivent aux crashes
- **Crash Recovery** : Rejoue modifications après crash
- **Write-Ahead Logging** : Log écrit avant données

📌 **Contexte** : Mécanisme central de durabilité InnoDB  
💡 **Configuration** :
```ini
innodb_log_file_size = 512M
innodb_log_files_in_group = 2
```
💡 **Fonctionnement** : Écriture circulaire (rotation automatique)  
🔗 **Voir aussi** : InnoDB, Undo Log, ACID, Crash Recovery

---

### Referential Integrity 🟢
**Garantie que les relations entre tables restent cohérentes.**

Assuré par les Foreign Keys :
- Valeur FK doit exister dans table référencée
- Actions ON DELETE / ON UPDATE contrôlent propagation

📌 **Contexte** : Qualité des données, cohérence  
💡 **Exemple** :
```sql
-- Impossible d'insérer order avec user_id inexistant
-- Impossible de supprimer user ayant des orders (selon action FK)
```
🔗 **Voir aussi** : Foreign Key, Constraint, Data Integrity

---

### Replication 🟡
**Processus de copie automatique des données d'un serveur (source) vers un ou plusieurs autres (replicas).**

Types MariaDB :
- **Asynchrone** : Source n'attend pas confirmation replica (standard)
- **Semi-synchrone** : Source attend 1+ replicas
- **Synchrone** : Galera Cluster (certification-based)

📌 **Contexte** : Scalabilité lecture, backup, haute disponibilité  
💡 **Architectures** :
- Master-Slave (1 source, N replicas)
- Multi-source (1 replica, N sources)
- Master-Master (bidirectionnel)
- Galera (multi-master synchrone)

🔗 **Voir aussi** : Binary Log, GTID, Galera, MaxScale

---

### Rollback 🟢
**Opération annulant toutes les modifications d'une transaction non committée.**

Restaure la base à l'état avant START TRANSACTION.

📌 **Contexte** : Gestion d'erreurs, annulation de changements  
💡 **Exemple** :
```sql
START TRANSACTION;
UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
-- Oups, erreur détectée !
ROLLBACK; -- Annulation
```
💡 **Automatique** : En cas d'erreur ou déconnexion  
🔗 **Voir aussi** : COMMIT, Transaction, Savepoint, ACID

---

### Row-Level Locking 🟡
**Verrouillage au niveau des lignes individuelles plutôt que de la table entière.**

Avantage majeur d'InnoDB vs MyISAM :
- Concurrence maximale
- Pas de blocage de toute la table
- Performances élevées en OLTP

📌 **Contexte** : InnoDB standard, MyISAM = table-level uniquement  
💡 **Types** : Shared locks (S), Exclusive locks (X)  
🔗 **Voir aussi** : InnoDB, Lock, MVCC, Concurrency

---

## S

### Savepoint 🟡
**Point de sauvegarde intermédiaire dans une transaction permettant un rollback partiel.**

Permet d'annuler une partie de transaction sans tout annuler.

📌 **Contexte** : Gestion fine des transactions complexes  
💡 **Exemple** :
```sql
START TRANSACTION;
INSERT INTO orders VALUES (1, 100);
SAVEPOINT sp1;
INSERT INTO orders VALUES (2, 200);
-- Oups, erreur sur deuxième insertion
ROLLBACK TO SAVEPOINT sp1; -- Annule seulement après sp1
COMMIT; -- Valide première insertion
```
🔗 **Voir aussi** : Transaction, ROLLBACK, COMMIT

---

### Schema 🟢
**Structure logique définissant l'organisation des données (tables, colonnes, types, contraintes).**

Dans MariaDB, "schema" est synonyme de "database".

📌 **Contexte** : Conception de base de données  
💡 **Commandes** :
```sql
CREATE SCHEMA mydb; -- Identique à CREATE DATABASE mydb
SHOW SCHEMAS; -- Liste les bases
```
🔗 **Voir aussi** : Database, Table, DDL, Normalization

---

### Sequence 🟡
**Objet générant des nombres séquentiels, alternative à AUTO_INCREMENT.**

Avantages vs AUTO_INCREMENT :
- Partageable entre tables
- Contrôle fin (incréments, min/max, cycle)
- Valeur obtenue avant INSERT

📌 **Contexte** : Génération d'identifiants complexes  
💡 **Exemple** :
```sql
CREATE SEQUENCE order_seq START WITH 1000 INCREMENT BY 1;
SELECT NEXT VALUE FOR order_seq; -- Obtient 1000
SELECT NEXT VALUE FOR order_seq; -- Obtient 1001
```
🔗 **Voir aussi** : AUTO_INCREMENT, Identity, Primary Key

---

### Sharding 🔴
**Partitionnement horizontal distribué : division des données sur plusieurs serveurs.**

Stratégies :
- **Range-based** : Par plage (IDs 1-1000 → serveur1, 1001-2000 → serveur2)
- **Hash-based** : Par hash de clé
- **Geo-based** : Par localisation géographique

📌 **Contexte** : Scalabilité extrême, distribution géographique  
💡 **Implémentation MariaDB** : Spider storage engine, ProxySQL  
⚠️ **Complexité** : Joins cross-shard difficiles, gestion complexe  
🔗 **Voir aussi** : Partitioning, Spider, Scalability, Distribution

---

### Slow Query Log 🟡
**Journal enregistrant les requêtes dépassant un seuil de temps d'exécution.**

Outil essentiel d'optimisation : identifie les requêtes problématiques.

📌 **Contexte** : Diagnostic performance, optimisation  
💡 **Configuration** :
```ini
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2  # Secondes
log_queries_not_using_indexes = ON
```
💡 **Analyse** :
```bash
pt-query-digest /var/log/mysql/slow.log
```
🔗 **Voir aussi** : Performance, EXPLAIN, pt-query-digest

---

### Spider 🔴
**Moteur de stockage permettant le sharding et accès transparent à tables distantes.**

Fonctionnalités :
- Partitionnement sur plusieurs serveurs MariaDB
- Transparence : table unique visible par application
- Support transactions distribuées

📌 **Contexte** : Sharding natif MariaDB  
💡 **Use case** : Scale-out horizontal, distribution géographique  
🔗 **Voir aussi** : Sharding, Storage Engine, Scale-out

---

### Spatial Index 🟡
**Index spécialisé pour données géographiques (points, lignes, polygones).**

Utilise R-Tree pour recherches spatiales efficaces :
- Points dans rayon
- Intersection de polygones
- Proximité géographique

📌 **Contexte** : Applications géolocalisées, cartographie  
💡 **Types** : POINT, LINESTRING, POLYGON, GEOMETRY  
💡 **Exemple** :
```sql
CREATE TABLE locations (
  id INT PRIMARY KEY,
  coordinates POINT NOT NULL,
  SPATIAL INDEX (coordinates)
);
SELECT * FROM locations 
WHERE ST_Distance_Sphere(coordinates, POINT(2.3522, 48.8566)) < 5000; -- 5km
```
🔗 **Voir aussi** : Index, R-Tree, GIS, Geometry

---

### SQL Injection 🟢
**Attaque exploitant une faille de sécurité en injectant du code SQL malveillant.**

Prévention :
- **Prepared Statements** : Séparation code/données
- **Validation des entrées** : Vérifier types et formats
- **Échappement** : Traiter caractères spéciaux
- **Principe du moindre privilège** : Limiter droits utilisateurs DB

📌 **Contexte** : Sécurité critique des applications  
💡 **Exemple vulnérable** :
```php
// DANGEREUX !
$query = "SELECT * FROM users WHERE id = " . $_GET['id'];
```
💡 **Exemple sécurisé** :
```php
// SÉCURISÉ
$stmt = $pdo->prepare("SELECT * FROM users WHERE id = ?");
$stmt->execute([$_GET['id']]);
```
🔗 **Voir aussi** : Prepared Statement, Security, Validation

---

### SST (State Snapshot Transfer) 🔴
**Méthode de synchronisation complète d'un nœud Galera Cluster.**

Utilisé quand :
- Nouveau nœud rejoint le cluster
- Nœud trop désynchronisé pour IST
- Corruption détectée

📌 **Contexte** : Mécanisme de sync initial/complet Galera  
💡 **Méthodes** :
- **rsync** : Rapide mais bloque le donneur
- **mysqldump** : Lent, compatible
- **mariabackup** : Recommandé (rapide, non-bloquant)
- **xtrabackup-v2** : Alternative Percona

💡 **Durée** : Dépend taille base (minutes à heures)  
🔗 **Voir aussi** : IST, Galera Cluster, State Transfer

---

### Storage Engine 🟢
**Composant gérant le stockage physique et les opérations sur les données.**

Architecture pluggable : chaque table peut utiliser un moteur différent.

📌 **Contexte** : Choix selon use case  
💡 **Moteurs principaux** :
- **InnoDB** : Transactionnel, OLTP (défaut)
- **Aria** : Crash-safe, tables système
- **ColumnStore** : Analytique, OLAP
- **MyISAM** : Legacy (éviter)
- **Memory** : Données en RAM
- **Spider** : Sharding distribué

💡 **Commande** :
```sql
SHOW ENGINES;
SELECT ENGINE FROM information_schema.TABLES WHERE TABLE_NAME = 'users';
```
🔗 **Voir aussi** : InnoDB, Aria, ColumnStore, Engine Selection

---

### Stored Procedure 🟡
**Programme SQL stocké côté serveur, exécutable par simple appel.**

Avantages :
- Encapsulation logique métier
- Réutilisabilité
- Performance (pré-compilé)
- Sécurité (contrôle d'accès)

📌 **Contexte** : Logique complexe, traitement batch  
💡 **Exemple** :
```sql
DELIMITER //
CREATE PROCEDURE transfer_money(
  IN from_account INT,
  IN to_account INT,
  IN amount DECIMAL(10,2)
)
BEGIN
  START TRANSACTION;
  UPDATE accounts SET balance = balance - amount WHERE id = from_account;
  UPDATE accounts SET balance = balance + amount WHERE id = to_account;
  COMMIT;
END //
DELIMITER ;

CALL transfer_money(1, 2, 100.00);
```
🔗 **Voir aussi** : Function, Trigger, Event, PL/SQL

---

### System-Versioned Tables 🟡
**Tables temporelles conservant automatiquement l'historique complet des modifications.**

MariaDB ajoute automatiquement :
- Colonnes temporelles (row_start, row_end)
- Table historique séparée
- Versioning transparent

📌 **Contexte** : Audit, historisation, requêtes temporelles  
💡 **Exemple** :
```sql
CREATE TABLE products (
  id INT PRIMARY KEY,
  name VARCHAR(100),
  price DECIMAL(10,2)
) WITH SYSTEM VERSIONING;

-- Requête temporelle
SELECT * FROM products FOR SYSTEM_TIME AS OF '2025-01-01 12:00:00';
```
🆕 **MariaDB 11.8** : Extension TIMESTAMP 2106, Application Time Period  
🔗 **Voir aussi** : Temporal Tables, Audit, History, Application Time Period

---

## T

### Table 🟢
**Structure relationnelle organisée en lignes et colonnes stockant les données.**

Composants :
- **Colonnes** : Définition des attributs (nom, type, contraintes)
- **Lignes** : Enregistrements de données
- **Index** : Structures d'accélération
- **Contraintes** : Règles d'intégrité

📌 **Contexte** : Élément fondamental des SGBD relationnels  
💡 **Création** :
```sql
CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(255) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;
```
🔗 **Voir aussi** : Schema, Column, Row, Storage Engine

---

### Tablespace 🟡
**Fichier physique contenant les données et index d'une ou plusieurs tables InnoDB.**

Types :
- **System Tablespace** : ibdata1 (tables système, undo logs)
- **File-per-table** : Fichier .ibd par table (défaut recommandé)
- **General Tablespace** : Tablespace partagé personnalisé

📌 **Contexte** : Organisation physique InnoDB  
💡 **Configuration** :
```ini
innodb_file_per_table = ON
```
🔗 **Voir aussi** : InnoDB, Storage, File System

---

### Thread Pool 🟡
**Mécanisme de gestion efficace des connexions via un pool de threads réutilisables.**

Avantages :
- Réduit overhead création/destruction threads
- Meilleure scalabilité (milliers de connexions)
- Consommation mémoire optimisée

📌 **Contexte** : Production haute concurrence  
💡 **Configuration** :
```ini
thread_handling = pool-of-threads
thread_pool_size = 16  # Généralement = nombre de CPUs
```
🔗 **Voir aussi** : Connection, Concurrency, Performance

---

### Transaction 🟢
**Unité logique de travail regroupant plusieurs opérations SQL exécutées atomiquement.**

Garanties ACID :
- Soit toutes les opérations réussissent (COMMIT)
- Soit aucune n'est appliquée (ROLLBACK)

📌 **Contexte** : Fondamental pour intégrité des données  
💡 **Exemple** :
```sql
START TRANSACTION;
-- ou BEGIN;
INSERT INTO orders (user_id, total) VALUES (1, 100);
INSERT INTO order_items (order_id, product_id) VALUES (LAST_INSERT_ID(), 5);
COMMIT;
```
💡 **Contrôle** :
- `START TRANSACTION` / `BEGIN` : Démarre
- `COMMIT` : Valide
- `ROLLBACK` : Annule

🔗 **Voir aussi** : ACID, COMMIT, ROLLBACK, Isolation Level

---

### Trigger 🟡
**Procédure automatiquement exécutée en réponse à un événement (INSERT, UPDATE, DELETE).**

Moments d'exécution :
- **BEFORE** : Avant modification (validation, transformation)
- **AFTER** : Après modification (audit, cascade)

📌 **Contexte** : Automatisation, audit, règles métier  
💡 **Exemple** :
```sql
CREATE TRIGGER audit_user_update
AFTER UPDATE ON users
FOR EACH ROW
  INSERT INTO audit_log (table_name, action, user_id, changed_at)
  VALUES ('users', 'UPDATE', NEW.id, NOW());
```
💡 **Variables** : `OLD` (valeurs avant), `NEW` (valeurs après)  
🔗 **Voir aussi** : Stored Procedure, Event, Automation

---

## U

### UCA (Unicode Collation Algorithm) 🟡 🆕
**Algorithme standard Unicode pour le tri et la comparaison de chaînes multilingues.**

MariaDB 11.8 adopte UCA 14.0.0 par défaut pour utf8mb4, améliorant :
- Support langues complexes
- Tri multilingue correct
- Compatibilité Unicode moderne

📌 **Contexte** : Internationalisation, tri correct  
💡 **Collations** : `utf8mb4_unicode_ci` utilise UCA  
🔗 **Voir aussi** : Collation, Charset, utf8mb4, Unicode

---

### Undo Log 🟡
**Journal InnoDB stockant les anciennes versions de lignes pour MVCC et rollback.**

Rôles :
- **MVCC** : Fournit versions antérieures pour lectures concurrentes
- **Rollback** : Permet annulation de transactions
- **Crash Recovery** : Annule transactions non committées

📌 **Contexte** : Mécanisme central concurrence InnoDB  
💡 **Stockage** : Undo tablespace (ibdata ou fichiers séparés)  
💡 **Purge** : Nettoyage automatique des anciennes versions  
🔗 **Voir aussi** : MVCC, Redo Log, InnoDB, Transaction

---

### Unique Index 🟢
**Index garantissant l'unicité des valeurs dans une ou plusieurs colonnes.**

Empêche l'insertion de doublons.

📌 **Contexte** : Contrainte d'unicité (email, username, etc.)  
💡 **Exemple** :
```sql
CREATE UNIQUE INDEX idx_email ON users(email);
-- ou lors de CREATE TABLE :
CREATE TABLE users (
  id INT PRIMARY KEY,
  email VARCHAR(255) UNIQUE
);
```
💡 **NULL** : Plusieurs NULL autorisés (NULL ≠ NULL)  
🔗 **Voir aussi** : Index, Constraint, Primary Key

---

### utf8mb4 🟢
**Charset UTF-8 complet (4 bytes) supportant tous les caractères Unicode incluant emojis.**

Différence avec utf8 (MariaDB) :
- **utf8** : Pseudo-UTF-8, max 3 bytes (incomplet)
- **utf8mb4** : Vrai UTF-8, max 4 bytes (complet)

📌 **Contexte** : Standard moderne, support emoji/caractères asiatiques  
💡 **Recommandation** : Toujours utiliser utf8mb4, pas utf8  
🆕 **MariaDB 11.8** : utf8mb4 par défaut avec UCA 14.0.0  
💡 **Configuration** :
```ini
character_set_server = utf8mb4
collation_server = utf8mb4_unicode_ci
```
🔗 **Voir aussi** : Charset, Collation, Unicode, UCA

---

### UUID (Universally Unique Identifier) 🟡
**Identifiant unique de 128 bits généré aléatoirement.**

Format : `550e8400-e29b-41d4-a716-446655440000`

📌 **Contexte** : Identifiants distribués, éviter conflits auto-increment  
💡 **Génération** :
```sql
SELECT UUID(); -- 550e8400-e29b-41d4-a716-446655440000
SELECT UUID_SHORT(); -- 92395783895228416 (plus compact)
```
💡 **Stockage** :
```sql
CREATE TABLE items (
  id BINARY(16) PRIMARY KEY DEFAULT (UNHEX(REPLACE(UUID(),'-','')))
);
-- ou comme VARCHAR(36) si lisibilité importante
```
⚠️ **Performance** : Plus lent que INT AUTO_INCREMENT (non séquentiel)  
🔗 **Voir aussi** : Primary Key, AUTO_INCREMENT, GUID

---

## V

### Vector (Type de données) 🔴 🆕
**Type de données MariaDB 11.8 pour stocker des vecteurs numériques de dimension fixe.**

Utilisé pour :
- Embeddings de modèles IA (OpenAI, Claude, etc.)
- Recherche sémantique
- Recommandations
- Détection d'anomalies

📌 **Contexte** : Nouveauté majeure MariaDB 11.8 pour IA/ML  
💡 **Syntaxe** :
```sql
CREATE TABLE documents (
  id INT PRIMARY KEY,
  content TEXT,
  embedding VECTOR(1536) NOT NULL,  -- OpenAI ada-002
  VECTOR INDEX (embedding) HNSW(M=16, ef_construction=64)
);
```
💡 **Recherche k-NN** :
```sql
SELECT id, VEC_DISTANCE_COSINE(embedding, ?) AS distance
FROM documents
ORDER BY distance LIMIT 5;
```
🔗 **Voir aussi** : HNSW, RAG, Semantic Search, AI/ML

---

### View 🟢
**Requête SQL stockée présentée comme une table virtuelle.**

Types :
- **Simple views** : Updatable si conditions respectées
- **Complex views** : Read-only (agrégations, joins multiples)
- **Materialized** : Non supporté nativement (workaround possible)

📌 **Contexte** : Abstraction, sécurité, simplification  
💡 **Exemple** :
```sql
CREATE VIEW active_users AS
SELECT id, name, email
FROM users
WHERE status = 'active';

SELECT * FROM active_users; -- Utilisation comme une table
```
💡 **Avantages** :
- Masquage colonnes sensibles
- Simplification requêtes complexes
- Logique métier centralisée

🔗 **Voir aussi** : Materialized View, Security, Abstraction

---

### Virtual Column 🟡
**Colonne calculée dont la valeur n'est pas stockée physiquement mais calculée à la lecture.**

Complément : **STORED** Column = valeur calculée et stockée.

📌 **Contexte** : Dénormalisation sans redondance stockage  
💡 **Exemple** :
```sql
CREATE TABLE products (
  price DECIMAL(10,2),
  tax_rate DECIMAL(4,2),
  price_with_tax DECIMAL(10,2) AS (price * (1 + tax_rate)) VIRTUAL
);
```
💡 **VIRTUAL vs STORED** :
- VIRTUAL : Pas de stockage, calcul à la lecture
- STORED : Stocké, calcul à l'écriture, indexable

🔗 **Voir aussi** : Generated Column, Computed Column, Index

---

## W

### Window Function 🟡
**Fonction SQL effectuant des calculs sur un ensemble de lignes liées à la ligne courante.**

Catégories :
- **Ranking** : ROW_NUMBER, RANK, DENSE_RANK
- **Analytical** : SUM, AVG, COUNT avec fenêtre
- **Value** : LAG, LEAD, FIRST_VALUE, LAST_VALUE

📌 **Contexte** : Analyses complexes sans GROUP BY  
💡 **Exemple** :
```sql
SELECT 
  employee_id,
  salary,
  RANK() OVER (ORDER BY salary DESC) AS salary_rank,
  AVG(salary) OVER (PARTITION BY department_id) AS dept_avg
FROM employees;
```
💡 **Syntaxe** :
```sql
<fonction>() OVER (
  [PARTITION BY ...]
  [ORDER BY ...]
  [ROWS|RANGE ...]
)
```
🔗 **Voir aussi** : Analytical Functions, PARTITION BY, ROW_NUMBER

---

### Writeset 🔴
**Ensemble des modifications (lignes changées) d'une transaction dans Galera Cluster.**

Le writeset contient :
- Clés des lignes modifiées
- Nouvelles valeurs
- Métadonnées de certification

📌 **Contexte** : Unité de réplication Galera  
💡 **Processus** :
1. Transaction commit local → génère writeset
2. Writeset diffusé à tous les nœuds
3. Certification : détection conflits
4. Application si pas de conflit

🔗 **Voir aussi** : Galera Cluster, Certification, Replication

---

## X

### XA Transaction 🔴
**Transaction distribuée coordonnée sur plusieurs ressources (bases, queues, etc.).**

Protocole 2-Phase Commit (2PC) :
1. **Prepare** : Toutes ressources préparent transaction
2. **Commit/Rollback** : Décision globale appliquée partout

📌 **Contexte** : Transactions cross-database, systèmes hétérogènes  
💡 **Support MariaDB** : Limité, préférer alternatives modernes  
⚠️ **Complexité** : Lent, erreurs possibles, éviter si possible  
🔗 **Voir aussi** : Distributed Transaction, 2PC, Saga Pattern

---

## Y

### YEAR 🟢
**Type de données stockant une année sur 4 chiffres.**

Format : YYYY (ex: 2025)  
Plage : 1901-2155

📌 **Contexte** : Stockage année uniquement  
💡 **Usage** :
```sql
CREATE TABLE events (
  id INT PRIMARY KEY,
  event_year YEAR
);
INSERT INTO events VALUES (1, 2025);
```
🔗 **Voir aussi** : DATE, DATETIME, TIMESTAMP

---

## Z

### Zero-Downtime Migration 🟣
**Stratégie de migration de base sans arrêt de service.**

Techniques :
- **Blue/Green Deployment** : Deux environnements parallèles
- **Réplication avec bascule** : Réplication → failover → ancien = replica
- **Online DDL** : Modifications schéma sans blocage
- **pt-online-schema-change / gh-ost** : Outils pour ALTER TABLE online

📌 **Contexte** : Production critique, SLA élevés  
💡 **Étapes typiques** :
1. Réplication ancien → nouveau
2. Sync et validation
3. Bascule applicative (DNS, proxy)
4. Ancien devient standby

🔗 **Voir aussi** : High Availability, Online DDL, Blue-Green, Replication

---

## ✅ Points Clés à Retenir

### Concepts Fondamentaux
- **ACID** : Propriétés garantissant fiabilité transactionnelle
- **InnoDB** : Moteur par défaut, transactionnel, MVCC
- **Index** : Structure accélérant recherches (B-Tree par défaut)
- **Transaction** : Unité atomique de travail (COMMIT/ROLLBACK)

### Réplication et Haute Disponibilité
- **Binary Log** : Journal des modifications pour réplication
- **GTID** : Identifiant global facilitant failover
- **Galera Cluster** : Multi-master synchrone
- **MaxScale** : Proxy intelligent pour load balancing

### Performance et Optimisation
- **Buffer Pool** : Cache InnoDB crucial (70-80% RAM)
- **EXPLAIN** : Analyse plan d'exécution requêtes
- **Partitioning** : Division grandes tables
- **MVCC** : Lectures concurrentes sans blocage

### Nouveautés MariaDB 11.8
- **MariaDB Vector** : Type VECTOR + HNSW pour IA
- **utf8mb4 + UCA 14.0.0** : Défaut moderne
- **TIMESTAMP 2106** : Extension résolution Y2038
- **PARSEC** : Nouveau plugin authentification
- **MaxScale 25.01** : Workload Capture/Replay

---

## 🔗 Références

### Documentation Officielle
- [MariaDB Knowledge Base - Glossary](https://mariadb.com/kb/en/mariadb-glossary/)
- [MariaDB Server Documentation](https://mariadb.com/kb/en/documentation/)
- [InnoDB Storage Engine](https://mariadb.com/kb/en/innodb/)
- [Galera Cluster Documentation](https://mariadb.com/kb/en/galera-cluster/)

### Standards
- [SQL:2023 Standard](https://www.iso.org/standard/76583.html)
- [Unicode Standard](https://unicode.org/standard/standard.html)

---

## ➡️ Section Suivante

**[A.2 Acronymes Courants →](./02-acronymes-courants.md)**  
Découvrez la signification des abréviations techniques : FK, PK, CTE, GTID, SST, IST, et plus encore.

---

**MariaDB** : 11.8 LTS  


⏭️ [Acronymes courants (FK, PK, CTE, GTID, SST, IST, etc.)](/annexes/glossaire/02-acronymes-courants.md)
