🔝 Retour au [Sommaire](/SOMMAIRE.md)

# A.2 - Acronymes Courants

> **Niveau** : Tous niveaux (Référence)  
> **Type** : Glossaire technique - Définitions des acronymes  
> **Mise à jour** : Décembre 2025 - MariaDB 11.8 LTS

---

## 📖 Introduction

Cette section définit les **acronymes et abréviations** couramment utilisés dans l'écosystème MariaDB et les technologies associées. Chaque acronyme est accompagné de sa signification complète, son contexte d'utilisation et des exemples pratiques.

### Légende des Niveaux
- 🟢 **Débutant** : Acronyme fondamental
- 🟡 **Intermédiaire** : Nécessite des bases MariaDB
- 🔴 **Avancé** : Contexte technique spécialisé
- 🟣 **Expert** : Utilisation en production avancée

---

## A

### ACL 🟡
**Access Control List** - Liste de contrôle d'accès

Liste définissant les permissions d'accès pour utilisateurs ou groupes.

📌 **Contexte** : Sécurité, gestion des droits  
💡 **MariaDB** : Système de privilèges basé sur tables grant  
🔗 **Voir aussi** : GRANT, REVOKE, Privileges, Security

---

### ACID 🟢
**Atomicity, Consistency, Isolation, Durability** - Atomicité, Cohérence, Isolation, Durabilité

Quatre propriétés garantissant la fiabilité des transactions.

📌 **Contexte** : Fondamental pour bases transactionnelles  
💡 **Garantie** : InnoDB est ACID-compliant  
🔗 **Voir aussi** : Transaction, InnoDB, Commit, Rollback

---

### AES 🟡
**Advanced Encryption Standard** - Standard de chiffrement avancé

Algorithme de chiffrement symétrique utilisé pour l'encryption at rest.

📌 **Contexte** : Sécurité des données, chiffrement  
💡 **MariaDB** : `AES_ENCRYPT()`, `AES_DECRYPT()`, encryption tablespaces  
💡 **Exemple** :
```sql
SELECT AES_ENCRYPT('sensitive data', 'encryption_key');
```
🔗 **Voir aussi** : Encryption at Rest, TLS/SSL, Security

---

### API 🟢
**Application Programming Interface** - Interface de programmation d'application

Interface permettant aux applications d'interagir avec MariaDB.

📌 **Contexte** : Développement applicatif  
💡 **Exemples** : JDBC API, PDO API, Connector APIs  
🔗 **Voir aussi** : Connector, Driver, Integration

---

### ARO 🔴
**Achievable Recovery Objective** - Objectif de récupération réalisable

Temps de récupération réellement atteignable avec les moyens actuels.

📌 **Contexte** : Disaster Recovery, planification  
💡 **Comparaison** : RTO = objectif, ARO = réalité mesurée  
🔗 **Voir aussi** : RTO, RPO, DR, Backup

---

### AS 🟢
**Alias Syntax** - Syntaxe d'alias

Mot-clé optionnel pour créer des alias de tables ou colonnes.

📌 **Contexte** : Requêtes SQL, lisibilité  
💡 **Exemple** :
```sql
SELECT u.name AS user_name FROM users AS u;
-- ou sans AS :
SELECT u.name user_name FROM users u;
```
🔗 **Voir aussi** : SELECT, Alias, Query

---

### ASCII 🟢
**American Standard Code for Information Interchange** - Code américain normalisé pour l'échange d'information

Encodage de caractères 7-bit (128 caractères).

📌 **Contexte** : Encodage historique, limité  
💡 **MariaDB** : Charset `ascii`, fonction `ASCII()`  
⚠️ **Limitation** : Pas de caractères accentués, préférer UTF-8  
🔗 **Voir aussi** : Charset, utf8mb4, Unicode

---

### ATO 🔴
**Achievable Time Objective** - Objectif de temps réalisable

Métrique de disponibilité réellement atteignable.

📌 **Contexte** : SLA, disponibilité  
🔗 **Voir aussi** : RTO, ARO, High Availability

---

## B

### BCNF 🟡
**Boyce-Codd Normal Form** - Forme normale de Boyce-Codd

Forme normale stricte (3NF améliorée) éliminant certaines anomalies.

📌 **Contexte** : Conception de schéma, normalisation  
💡 **Définition** : Toute dépendance fonctionnelle implique une clé candidate  
🔗 **Voir aussi** : Normalization, 3NF, Schema Design

---

### BI 🟡
**Business Intelligence** - Intelligence d'affaires

Analyses et reporting pour aide à la décision.

📌 **Contexte** : Analytique, reporting  
💡 **MariaDB** : ColumnStore pour BI/OLAP workloads  
🔗 **Voir aussi** : OLAP, Data Warehouse, ColumnStore

---

### BLOB 🟢
**Binary Large Object** - Grand objet binaire

Type de données pour stocker de grandes données binaires (images, fichiers).

📌 **Contexte** : Stockage fichiers, médias  
💡 **Types** : TINYBLOB, BLOB, MEDIUMBLOB, LONGBLOB  
💡 **Tailles** :
- TINYBLOB: 255 bytes
- BLOB: 64 KB
- MEDIUMBLOB: 16 MB
- LONGBLOB: 4 GB

🔗 **Voir aussi** : Binary Types, LOB, Storage

---

## C

### CA 🟡
**Certificate Authority** - Autorité de certification

Entité émettant des certificats numériques pour SSL/TLS.

📌 **Contexte** : Sécurité, chiffrement connexions  
💡 **MariaDB** : Configuration SSL avec certificats CA  
💡 **Fichiers** : `ca-cert.pem`, `ssl-ca`  
🔗 **Voir aussi** : SSL/TLS, Certificate, Security

---

### CAP 🔴
**Consistency, Availability, Partition Tolerance** - Cohérence, Disponibilité, Tolérance au partitionnement

Théorème stipulant qu'un système distribué ne peut garantir que 2 des 3 propriétés.

📌 **Contexte** : Systèmes distribués, architecture  
💡 **MariaDB** : Galera privilégie C et A (sacrifice P partiel)  
🔗 **Voir aussi** : Distributed Systems, Galera, Consistency

---

### CDC 🟡
**Change Data Capture** - Capture des changements de données

Processus de détection et capture des modifications pour propagation.

📌 **Contexte** : Event-Driven Architecture, intégration  
💡 **Outils** : Debezium, Maxwell, binlog streaming  
💡 **Source** : Binary log MariaDB  
🔗 **Voir aussi** : Binary Log, Event-Driven, Debezium

---

### CI/CD 🟡
**Continuous Integration / Continuous Deployment** - Intégration continue / Déploiement continu

Pratiques DevOps d'automatisation build, test et déploiement.

📌 **Contexte** : DevOps, automatisation  
💡 **Outils** : GitLab CI, Jenkins, GitHub Actions  
💡 **Usage DB** : Migrations automatiques, tests DB  
🔗 **Voir aussi** : DevOps, Automation, GitOps

---

### CLI 🟢
**Command Line Interface** - Interface en ligne de commande

Interface textuelle pour interaction avec MariaDB.

📌 **Contexte** : Administration, scripts  
💡 **Commande** : `mariadb`, `mysql` (client)  
💡 **Exemple** :
```bash
mariadb -u root -p -h localhost mydb
```
🔗 **Voir aussi** : mariadb client, mysql client, Shell

---

### CLOB 🟡
**Character Large Object** - Grand objet de caractères

Type de données pour grandes chaînes de texte.

📌 **Contexte** : Stockage texte volumineux  
💡 **MariaDB** : TEXT, MEDIUMTEXT, LONGTEXT (équivalents CLOB)  
🔗 **Voir aussi** : TEXT, BLOB, LOB

---

### CPU 🟢
**Central Processing Unit** - Unité centrale de traitement

Processeur exécutant les opérations du serveur.

📌 **Contexte** : Performance, ressources matérielles  
💡 **Optimisation** : Thread pool, parallélisme queries  
💡 **Monitoring** : `SHOW PROCESSLIST`, CPU usage  
🔗 **Voir aussi** : Thread Pool, Performance, Hardware

---

### CQRS 🔴
**Command Query Responsibility Segregation** - Séparation responsabilité commandes/requêtes

Pattern séparant opérations lecture/écriture.

📌 **Contexte** : Architecture microservices, scalabilité  
💡 **MariaDB** : Réplication read replicas pour queries  
🔗 **Voir aussi** : Replication, Read/Write Split, Microservices

---

### CRD 🟡
**Custom Resource Definition** - Définition de ressource personnalisée

Extension Kubernetes pour ressources custom.

📌 **Contexte** : Kubernetes, orchestration  
💡 **MariaDB** : mariadb-operator utilise CRDs  
🔗 **Voir aussi** : Kubernetes, Operator, K8s

---

### CRUD 🟢
**Create, Read, Update, Delete** - Créer, Lire, Mettre à jour, Supprimer

Quatre opérations de base sur les données.

📌 **Contexte** : Opérations fondamentales  
💡 **SQL** :
- Create: `INSERT`
- Read: `SELECT`
- Update: `UPDATE`
- Delete: `DELETE`

🔗 **Voir aussi** : DML, SQL Operations, DBMS

---

### CSV 🟢
**Comma-Separated Values** - Valeurs séparées par virgules

Format de fichier texte pour données tabulaires.

📌 **Contexte** : Import/export données  
💡 **MariaDB** :
```sql
LOAD DATA INFILE 'data.csv' INTO TABLE users FIELDS TERMINATED BY ',';
SELECT * INTO OUTFILE 'export.csv' FIELDS TERMINATED BY ',' FROM users;
```
🔗 **Voir aussi** : LOAD DATA, Export, Import

---

### CTE 🟡
**Common Table Expression** - Expression de table commune

Résultat nommé temporaire d'une requête, utilisable dans la requête principale.

📌 **Contexte** : Requêtes complexes, lisibilité  
💡 **Syntaxe** : `WITH`  
💡 **Exemple** :
```sql
WITH top_customers AS (
  SELECT customer_id, SUM(amount) as total
  FROM orders
  GROUP BY customer_id
  HAVING total > 10000
)
SELECT c.name, tc.total
FROM top_customers tc
JOIN customers c ON tc.customer_id = c.id;
```
💡 **Recursive CTE** : `WITH RECURSIVE` pour hiérarchies  
🔗 **Voir aussi** : WITH, Subquery, Recursive Query

---

## D

### DB 🟢
**Database** - Base de données

Collection organisée de données structurées.

📌 **Contexte** : Terme générique  
💡 **MariaDB** : Synonyme de SCHEMA  
🔗 **Voir aussi** : Schema, DBMS, Table

---

### DBA 🟢
**Database Administrator** - Administrateur de base de données

Professionnel gérant installation, configuration, maintenance, sécurité DB.

📌 **Contexte** : Rôle, administration  
💡 **Responsabilités** : Backup, tuning, sécurité, monitoring  
🔗 **Voir aussi** : Administration, DevOps, SRE

---

### DBMS 🟢
**Database Management System** - Système de gestion de base de données

Logiciel gérant les bases de données.

📌 **Contexte** : Catégorie logicielle  
💡 **Exemples** : MariaDB, PostgreSQL, Oracle, SQL Server  
💡 **Types** : RDBMS (relationnel), NoSQL, NewSQL  
🔗 **Voir aussi** : RDBMS, SQL, Database

---

### DCL 🟡
**Data Control Language** - Langage de contrôle des données

Sous-ensemble SQL pour gestion des permissions.

📌 **Contexte** : Sécurité, contrôle d'accès  
💡 **Commandes** :
- `GRANT` : Attribution de privilèges
- `REVOKE` : Révocation de privilèges

💡 **Exemple** :
```sql
GRANT SELECT, INSERT ON mydb.* TO 'user'@'localhost';
REVOKE DELETE ON mydb.* FROM 'user'@'localhost';
```
🔗 **Voir aussi** : GRANT, REVOKE, DDL, DML

---

### DDL 🟢
**Data Definition Language** - Langage de définition des données

Sous-ensemble SQL pour définir la structure des données.

📌 **Contexte** : Création et modification schéma  
💡 **Commandes** :
- `CREATE` : Création objets (table, database, index)
- `ALTER` : Modification structure
- `DROP` : Suppression objets
- `TRUNCATE` : Vidage table
- `RENAME` : Renommage objets

💡 **Exemple** :
```sql
CREATE TABLE users (id INT PRIMARY KEY, name VARCHAR(100));
ALTER TABLE users ADD COLUMN email VARCHAR(255);
DROP TABLE old_table;
```
🔗 **Voir aussi** : DML, DCL, TCL, Schema

---

### DML 🟢
**Data Manipulation Language** - Langage de manipulation des données

Sous-ensemble SQL pour manipulation des données.

📌 **Contexte** : Opérations CRUD  
💡 **Commandes** :
- `SELECT` : Lecture
- `INSERT` : Insertion
- `UPDATE` : Mise à jour
- `DELETE` : Suppression

💡 **Exemple** :
```sql
INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com');
SELECT * FROM users WHERE name = 'Alice';
UPDATE users SET email = 'newemail@example.com' WHERE id = 1;
DELETE FROM users WHERE id = 1;
```
🔗 **Voir aussi** : DDL, DCL, CRUD, Query

---

### DNS 🟢
**Domain Name System** - Système de noms de domaine

Système traduisant noms d'hôtes en adresses IP.

📌 **Contexte** : Réseau, connexion  
💡 **MariaDB** : Résolution hostname dans connexions  
💡 **Configuration** : `skip-name-resolve` pour désactiver DNS lookup  
🔗 **Voir aussi** : Network, Connection, hostname

---

### DR 🟣
**Disaster Recovery** - Récupération après sinistre

Processus et stratégies pour restauration après incident majeur.

📌 **Contexte** : Continuité d'activité, résilience  
💡 **Composants** :
- Backups réguliers
- Site de secours (DR site)
- Procédures de restauration
- Tests DR

💡 **Métriques** : RTO, RPO  
🔗 **Voir aussi** : Backup, RTO, RPO, High Availability

---

### DRI 🟡
**Declarative Referential Integrity** - Intégrité référentielle déclarative

Mécanisme automatique assurant cohérence des références entre tables.

📌 **Contexte** : Intégrité des données  
💡 **Implémentation** : Foreign Keys  
🔗 **Voir aussi** : Foreign Key, Referential Integrity, Constraint

---

## E

### ELT 🟡
**Extract, Load, Transform** - Extraire, Charger, Transformer

Processus de chargement données puis transformation dans DB cible.

📌 **Contexte** : Data warehousing, alternative à ETL  
💡 **Différence ETL** : Transformation après chargement (dans DB)  
🔗 **Voir aussi** : ETL, Data Warehouse, Integration

---

### EOF 🟢
**End Of File** - Fin de fichier

Marqueur indiquant la fin d'un fichier.

📌 **Contexte** : Import/export fichiers  
🔗 **Voir aussi** : LOAD DATA, Import, File

---

### ERM 🟡
**Entity-Relationship Model** - Modèle entité-relation

Modèle conceptuel pour conception de bases de données.

📌 **Contexte** : Conception schéma, modélisation  
💡 **Composants** : Entités, relations, attributs  
🔗 **Voir aussi** : Schema Design, Normalization, ERD

---

### ETL 🟡
**Extract, Transform, Load** - Extraire, Transformer, Charger

Processus d'extraction, transformation puis chargement de données.

📌 **Contexte** : Data warehousing, migration  
💡 **Outils** : Talend, Apache NiFi, Pentaho  
🔗 **Voir aussi** : ELT, Data Warehouse, Integration

---

## F

### FK 🟢
**Foreign Key** - Clé étrangère

Colonne(s) référençant la clé primaire d'une autre table.

📌 **Contexte** : Relations entre tables, intégrité référentielle  
💡 **Exemple** :
```sql
CREATE TABLE orders (
  id INT PRIMARY KEY,
  user_id INT,
  FOREIGN KEY (user_id) REFERENCES users(id)
);
```
💡 **Actions** : CASCADE, RESTRICT, SET NULL, NO ACTION  
⚠️ **InnoDB uniquement** : MyISAM/Aria ne supportent pas FKs  
🔗 **Voir aussi** : Primary Key, Referential Integrity, Constraint

---

### FQN 🟡
**Fully Qualified Name** - Nom pleinement qualifié

Nom complet incluant tous les niveaux (database.table.column).

📌 **Contexte** : Référencement précis, éviter ambiguïté  
💡 **Exemple** :
```sql
SELECT mydb.users.email FROM mydb.users;
-- ou :
SELECT u.email FROM mydb.users AS u;
```
🔗 **Voir aussi** : Namespace, Schema, Identifier

---

### FTS 🟡
**Full-Text Search** - Recherche en texte intégral

Recherche dans contenu textuel avec pertinence.

📌 **Contexte** : Recherche documentaire, contenu  
💡 **MariaDB** : Index FULLTEXT  
💡 **Exemple** :
```sql
CREATE FULLTEXT INDEX ft_content ON articles(content);
SELECT * FROM articles WHERE MATCH(content) AGAINST('MariaDB performance');
```
🔗 **Voir aussi** : FULLTEXT Index, MATCH AGAINST, Search

---

## G

### GB 🟢
**Gigabyte** - Gigaoctet

Unité de mesure (1 GB = 1024 MB = 1,073,741,824 bytes).

📌 **Contexte** : Taille données, mémoire, stockage  
💡 **Configuration** : `innodb_buffer_pool_size = 8G`  
🔗 **Voir aussi** : MB, TB, Storage, Memory

---

### GIS 🟡
**Geographic Information System** - Système d'information géographique

Système gérant données spatiales/géographiques.

📌 **Contexte** : Données géolocalisées  
💡 **MariaDB** : Support types GEOMETRY, POINT, POLYGON  
💡 **Fonctions** : ST_Distance, ST_Contains, ST_Within  
🔗 **Voir aussi** : Spatial Index, Geometry, Location

---

### GRANT 🟢
**GRANT** - Accorder

Commande SQL pour attribution de privilèges.

📌 **Contexte** : Sécurité, contrôle d'accès (partie de DCL)  
💡 **Exemple** :
```sql
GRANT ALL PRIVILEGES ON mydb.* TO 'user'@'localhost';
GRANT SELECT ON mydb.users TO 'readonly'@'%';
```
🔗 **Voir aussi** : REVOKE, DCL, Privileges, Security

---

### GTID 🟡
**Global Transaction Identifier** - Identifiant global de transaction

Identifiant unique pour chaque transaction répliquée.

📌 **Contexte** : Réplication, failover  
💡 **Format** : `domain_id-server_id-sequence_number` (ex: `0-100-5432`)  
💡 **Avantage** : Simplifie failover, pas besoin position binlog  
💡 **Configuration** :
```ini
gtid_strict_mode = ON
log_slave_updates = ON
```
🔗 **Voir aussi** : Replication, Binary Log, Failover

---

### GUI 🟢
**Graphical User Interface** - Interface graphique utilisateur

Interface visuelle pour interaction (vs CLI).

📌 **Contexte** : Administration, développement  
💡 **Outils MariaDB** : HeidiSQL, DBeaver, phpMyAdmin, MySQL Workbench  
🔗 **Voir aussi** : CLI, HeidiSQL, DBeaver, Administration

---

## H

### HA 🟣
**High Availability** - Haute disponibilité

Capacité d'un système à rester opérationnel avec minimum downtime.

📌 **Contexte** : Production critique, SLA  
💡 **Solutions MariaDB** :
- Galera Cluster (multi-master)
- Réplication + failover
- MaxScale load balancing

💡 **Métrique** : Uptime (ex: 99.99% = ~52 min downtime/an)  
🔗 **Voir aussi** : Galera, Replication, MaxScale, DR

---

### HDD 🟢
**Hard Disk Drive** - Disque dur

Stockage magnétique mécanique (vs SSD).

📌 **Contexte** : Stockage, performance  
💡 **Performance** : Plus lent que SSD (IOPS limités)  
💡 **Usage** : Archivage, données froides  
🔗 **Voir aussi** : SSD, IOPS, Storage, I/O

---

### HNSW 🔴 🆕
**Hierarchical Navigable Small Worlds** - Petits mondes navigables hiérarchiques

Algorithme d'index pour recherche vectorielle k-NN.

📌 **Contexte** : MariaDB 11.8, recherche vectorielle IA  
💡 **Usage** : Indexation vecteurs embeddings pour RAG  
💡 **Exemple** :
```sql
CREATE TABLE embeddings (
  id INT PRIMARY KEY,
  vector VECTOR(1536),
  VECTOR INDEX (vector) HNSW(M=16, ef_construction=64)
);
```
💡 **Paramètres** :
- `M` : Nombre de connexions (trade-off vitesse/précision)
- `ef_construction` : Qualité construction index

🔗 **Voir aussi** : Vector, k-NN, RAG, Semantic Search

---

### HTTP 🟢
**Hypertext Transfer Protocol** - Protocole de transfert hypertexte

Protocole de communication web.

📌 **Contexte** : APIs REST, webhooks  
💡 **MariaDB** : Accès via REST APIs (via middleware)  
🔗 **Voir aussi** : REST API, HTTPS, Web

---

### HTTPS 🟢
**HTTP Secure** - HTTP sécurisé

HTTP avec chiffrement SSL/TLS.

📌 **Contexte** : Sécurité communications web  
🔗 **Voir aussi** : SSL/TLS, HTTP, Security

---

## I

### IaaS 🟡
**Infrastructure as a Service** - Infrastructure en tant que service

Modèle cloud fournissant infrastructure virtualisée.

📌 **Contexte** : Cloud computing  
💡 **Exemples** : AWS EC2, Azure VMs, Google Compute Engine  
💡 **Usage MariaDB** : Déploiement sur VMs cloud  
🔗 **Voir aussi** : PaaS, SaaS, Cloud, AWS, Azure

---

### IaC 🟡
**Infrastructure as Code** - Infrastructure en tant que code

Gestion infrastructure via fichiers de configuration.

📌 **Contexte** : DevOps, automatisation  
💡 **Outils** : Terraform, Ansible, CloudFormation  
💡 **Exemple** : Déploiement automatisé MariaDB cluster  
🔗 **Voir aussi** : Terraform, Ansible, DevOps, Automation

---

### IDE 🟢
**Integrated Development Environment** - Environnement de développement intégré

Logiciel facilitant développement (éditeur + outils).

📌 **Contexte** : Développement  
💡 **Exemples** : VS Code, IntelliJ, Eclipse, DataGrip  
🔗 **Voir aussi** : Development, Tools, SQL Editor

---

### IOPS 🟡
**Input/Output Operations Per Second** - Opérations d'entrée/sortie par seconde

Métrique de performance stockage.

📌 **Contexte** : Performance disque, I/O  
💡 **Typique** :
- HDD: 100-200 IOPS
- SSD: 10,000-100,000+ IOPS
- NVMe: 200,000-1,000,000+ IOPS

💡 **Configuration MariaDB** :
```ini
innodb_io_capacity = 2000  # Adapté à SSD
innodb_io_capacity_max = 4000
```
🔗 **Voir aussi** : I/O, SSD, Performance, innodb_io_capacity

---

### IP 🟢
**Internet Protocol** - Protocole Internet

Protocole réseau pour adressage et routage.

📌 **Contexte** : Connexions réseau  
💡 **MariaDB** : Connexions TCP/IP, bind-address  
💡 **Exemple** :
```ini
bind-address = 0.0.0.0  # Écoute toutes interfaces
```
🔗 **Voir aussi** : TCP, Network, Connection, bind-address

---

### IST 🔴
**Incremental State Transfer** - Transfert d'état incrémental

Synchronisation incrémentale nœud Galera via gcache.

📌 **Contexte** : Galera Cluster, synchronisation  
💡 **Condition** : Writesets manquants dans gcache du donneur  
💡 **Avantage** : Plus rapide que SST (pas de snapshot complet)  
🔗 **Voir aussi** : SST, Galera Cluster, State Transfer, gcache

---

## J

### JDBC 🟡
**Java Database Connectivity** - Connectivité de base de données Java

API Java standard pour connexion aux bases de données.

📌 **Contexte** : Développement Java  
💡 **Driver MariaDB** : `org.mariadb.jdbc.Driver`  
💡 **Exemple** :
```java
Connection conn = DriverManager.getConnection(
  "jdbc:mariadb://localhost:3306/mydb", "user", "pass"
);
```
🔗 **Voir aussi** : Connector/J, Java, Driver, API

---

### JSON 🟡
**JavaScript Object Notation** - Notation objet JavaScript

Format de données texte léger et structuré.

📌 **Contexte** : Données semi-structurées  
💡 **MariaDB** : Type JSON, fonctions JSON_*  
💡 **Exemple** :
```sql
CREATE TABLE products (
  id INT PRIMARY KEY,
  attributes JSON
);
INSERT INTO products VALUES (1, '{"color":"red","size":"L"}');
SELECT JSON_EXTRACT(attributes, '$.color') FROM products;
```
🔗 **Voir aussi** : JSON Functions, NoSQL, Semi-structured

---

## K

### K8s 🟡
**Kubernetes** - Kubernetes (K + 8 lettres + s)

Plateforme d'orchestration de conteneurs.

📌 **Contexte** : Orchestration, cloud-native  
💡 **MariaDB** : mariadb-operator pour K8s  
💡 **Composants** : Pods, StatefulSets, Services, PVCs  
🔗 **Voir aussi** : Kubernetes, Container, Docker, Operator

---

### KB 🟢
**Kilobyte** - Kilooctet

Unité de mesure (1 KB = 1024 bytes).

📌 **Contexte** : Taille données, mémoire  
🔗 **Voir aussi** : MB, GB, Storage

---

### k-NN 🔴
**k-Nearest Neighbors** - k plus proches voisins

Algorithme de recherche des k éléments les plus similaires.

📌 **Contexte** : Recherche vectorielle, IA/ML  
💡 **MariaDB 11.8** : Recherche k-NN via index HNSW  
💡 **Exemple** :
```sql
SELECT id, VEC_DISTANCE_COSINE(embedding, ?) AS distance
FROM documents
ORDER BY distance
LIMIT 5;  -- k=5 plus proches voisins
```
🔗 **Voir aussi** : HNSW, Vector, RAG, Similarity Search

---

## L

### LAN 🟢
**Local Area Network** - Réseau local

Réseau informatique limité géographiquement.

📌 **Contexte** : Réseau, connectivité  
💡 **MariaDB** : Connexions intra-datacenter  
🔗 **Voir aussi** : WAN, Network, Connection

---

### LDAP 🟡
**Lightweight Directory Access Protocol** - Protocole d'accès aux annuaires allégé

Protocole pour accès annuaires centralisés (Active Directory).

📌 **Contexte** : Authentification centralisée  
💡 **MariaDB** : Plugin auth_pam avec LDAP  
🔗 **Voir aussi** : Authentication, PAM, Active Directory, SSO

---

### LLM 🔴 🆕
**Large Language Model** - Grand modèle de langage

Modèle IA de traitement du langage naturel.

📌 **Contexte** : IA générative, RAG  
💡 **Exemples** : GPT-4, Claude, LLaMA, Mistral  
💡 **MariaDB** : Stockage embeddings LLM dans Vector type  
🔗 **Voir aussi** : RAG, Vector, Embeddings, AI/ML

---

### LOB 🟡
**Large Object** - Grand objet

Type de données pour stocker grandes valeurs (texte ou binaire).

📌 **Contexte** : Stockage données volumineuses  
💡 **Types** : BLOB (binaire), CLOB (texte)  
🔗 **Voir aussi** : BLOB, TEXT, Binary Types

---

### LTS 🟢
**Long Term Support** - Support long terme

Version avec support étendu (3 ans pour MariaDB depuis 11.4).

📌 **Contexte** : Production, stabilité  
💡 **Versions LTS** : 10.6, 10.11, 11.4, **11.8** 🆕  
💡 **Cycle** : Support 3 ans (correctifs, sécurité)  
🔗 **Voir aussi** : Version Policy, Rolling Release, Support

---

## M

### MB 🟢
**Megabyte** - Mégaoctet

Unité de mesure (1 MB = 1024 KB = 1,048,576 bytes).

📌 **Contexte** : Taille données, configuration  
💡 **Exemple** : `max_allowed_packet = 64M`  
🔗 **Voir aussi** : KB, GB, Storage

---

### MCP 🔴 🆕
**Model Context Protocol** - Protocole de contexte de modèle

Protocole standardisant interaction entre LLMs et sources de données.

📌 **Contexte** : Intégration IA, MariaDB 11.8  
💡 **MariaDB MCP Server** : Permet LLMs d'interroger MariaDB  
🔗 **Voir aussi** : LLM, AI Integration, RAG

---

### MVCC 🟡
**Multi-Version Concurrency Control** - Contrôle de concurrence multi-version

Mécanisme permettant lectures concurrentes sans verrous.

📌 **Contexte** : Concurrence InnoDB  
💡 **Principe** : Chaque transaction voit une version cohérente  
💡 **Implémentation** : Undo log conserve anciennes versions  
🔗 **Voir aussi** : InnoDB, Undo Log, Isolation, Concurrency

---

## N

### NAT 🟢
**Network Address Translation** - Traduction d'adresse réseau

Mécanisme de modification d'adresses IP dans paquets réseau.

📌 **Contexte** : Réseau, connectivité  
🔗 **Voir aussi** : Network, Firewall, IP

---

### NFS 🟡
**Network File System** - Système de fichiers réseau

Protocole de partage de fichiers sur réseau.

📌 **Contexte** : Stockage partagé  
💡 **Usage MariaDB** : Datadir sur NFS (avec précautions)  
⚠️ **Attention** : Risques performance et corruption  
🔗 **Voir aussi** : Storage, File System, Network

---

### NIC 🟢
**Network Interface Card** - Carte d'interface réseau

Composant matériel pour connexion réseau.

📌 **Contexte** : Hardware, réseau  
💡 **Performance** : 1 Gbps, 10 Gbps, 25 Gbps, 100 Gbps  
🔗 **Voir aussi** : Network, Hardware, Bandwidth

---

### NN 🟢
**Not Null** - Non nul

Contrainte interdisant valeurs NULL.

📌 **Contexte** : Intégrité des données  
💡 **Exemple** :
```sql
CREATE TABLE users (
  id INT PRIMARY KEY,
  email VARCHAR(255) NOT NULL
);
```
🔗 **Voir aussi** : Constraint, NULL, Data Integrity

---

### NUMA 🔴
**Non-Uniform Memory Access** - Accès mémoire non uniforme

Architecture où temps d'accès mémoire varie selon localisation.

📌 **Contexte** : Hardware, performance serveurs multi-CPU  
💡 **MariaDB** : Configuration NUMA peut affecter performances  
🔗 **Voir aussi** : Memory, Hardware, Performance

---

## O

### ODBC 🟡
**Open Database Connectivity** - Connectivité de base de données ouverte

Standard API multi-plateformes pour accès DB.

📌 **Contexte** : Connectivité Windows, intégration  
💡 **MariaDB** : Connector/ODBC disponible  
🔗 **Voir aussi** : JDBC, Driver, Connector, API

---

### OLAP 🟡
**Online Analytical Processing** - Traitement analytique en ligne

Workload orienté analyses complexes sur grandes volumétries.

📌 **Contexte** : Business Intelligence, Data Warehouse  
💡 **Caractéristiques** :
- Requêtes complexes, agrégations
- Lecture intensive
- Peu d'écritures concurrentes

💡 **Moteur MariaDB** : ColumnStore optimisé OLAP  
🔗 **Voir aussi** : OLTP, ColumnStore, Data Warehouse, BI

---

### OLTP 🟡
**Online Transaction Processing** - Traitement transactionnel en ligne

Workload orienté transactions courtes et fréquentes.

📌 **Contexte** : Applications web, e-commerce  
💡 **Caractéristiques** :
- Transactions courtes
- Lecture/écriture mixte
- Haute concurrence
- Latence faible

💡 **Moteur MariaDB** : InnoDB optimisé OLTP  
🔗 **Voir aussi** : OLAP, InnoDB, Transaction, Index

---

### ORM 🟡
**Object-Relational Mapping** - Cartographie objet-relationnel

Technique mappant objets application vers tables DB.

📌 **Contexte** : Développement applicatif  
💡 **Frameworks** :
- Java: Hibernate, JPA
- Python: SQLAlchemy, Django ORM
- JavaScript: Sequelize, TypeORM, Prisma
- .NET: Entity Framework

💡 **Avantages** : Abstraction SQL, portabilité  
⚠️ **Inconvénients** : Performance, requêtes complexes  
🔗 **Voir aussi** : Hibernate, SQLAlchemy, Application Development

---

### OS 🟢
**Operating System** - Système d'exploitation

Logiciel système gérant le matériel.

📌 **Contexte** : Infrastructure serveur  
💡 **MariaDB** : Supporte Linux, Windows, macOS, BSD  
💡 **Recommandé** : Linux (Ubuntu, CentOS, Debian)  
🔗 **Voir aussi** : Linux, Server, Infrastructure

---

## P

### PaaS 🟡
**Platform as a Service** - Plateforme en tant que service

Modèle cloud fournissant plateforme de développement/déploiement.

📌 **Contexte** : Cloud computing  
💡 **Exemples** : AWS RDS, Azure Database, Google Cloud SQL  
💡 **MariaDB** : Offres PaaS disponibles  
🔗 **Voir aussi** : IaaS, SaaS, Cloud, RDS

---

### PAM 🟡
**Pluggable Authentication Modules** - Modules d'authentification enfichables

Framework d'authentification Unix/Linux.

📌 **Contexte** : Authentification centralisée  
💡 **MariaDB** : Plugin auth_pam pour auth OS-level  
💡 **Exemple** :
```sql
CREATE USER 'user'@'localhost' IDENTIFIED VIA pam;
```
🔗 **Voir aussi** : Authentication, LDAP, Security, Unix

---

### PARSEC 🟡 🆕
**PARSEC** - Plugin d'authentification MariaDB 11.8

Nouveau plugin d'authentification sécurisé.

📌 **Contexte** : Nouveauté MariaDB 11.8, sécurité  
💡 **Avantage** : Authentification moderne et sécurisée  
🔗 **Voir aussi** : Authentication, ed25519, Security

---

### PDO 🟡
**PHP Data Objects** - Objets de données PHP

Extension PHP d'accès DB avec interface orientée objet.

📌 **Contexte** : Développement PHP  
💡 **Avantage** : Abstraction, prepared statements sécurisés  
💡 **Exemple** :
```php
$pdo = new PDO('mysql:host=localhost;dbname=mydb', 'user', 'pass');
$stmt = $pdo->prepare('SELECT * FROM users WHERE id = ?');
$stmt->execute([123]);
```
🔗 **Voir aussi** : PHP, mysqli, Prepared Statement, Driver

---

### PITR 🟡
**Point-In-Time Recovery** - Récupération à un instant donné

Restauration DB à un instant précis via binlogs.

📌 **Contexte** : Recovery, disaster recovery  
💡 **Processus** :
1. Restaurer dernière sauvegarde complète
2. Rejouer binary logs jusqu'à instant T
3. DB restaurée à l'état exact de T

💡 **Prérequis** : Binary logs activés et archivés  
🔗 **Voir aussi** : Binary Log, Backup, Recovery, DR

---

### PK 🟢
**Primary Key** - Clé primaire

Colonne(s) identifiant de manière unique chaque ligne.

📌 **Contexte** : Modélisation, intégrité des données  
💡 **Propriétés** : UNIQUE + NOT NULL + indexé automatiquement  
💡 **Exemple** :
```sql
CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(255) UNIQUE
);
-- ou clé composite :
CREATE TABLE order_items (
  order_id INT,
  product_id INT,
  PRIMARY KEY (order_id, product_id)
);
```
🔗 **Voir aussi** : Foreign Key, Unique, Auto-Increment, Index

---

### PRA 🟣
**Plan de Reprise d'Activité** - Disaster Recovery Plan (DRP en anglais)

Document définissant procédures de restauration après incident.

📌 **Contexte** : Continuité d'activité  
💡 **Contient** : Procédures, responsables, RTO/RPO, tests  
🔗 **Voir aussi** : DR, RTO, RPO, Backup

---

### PVC 🟡
**PersistentVolumeClaim** - Demande de volume persistant

Objet Kubernetes demandant stockage persistant.

📌 **Contexte** : Kubernetes, stockage  
💡 **Usage MariaDB** : Datadir persistant dans K8s  
💡 **Exemple** :
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 10Gi
```
🔗 **Voir aussi** : Kubernetes, PV, Storage, StatefulSet

---

## Q

### QPS 🟡
**Queries Per Second** - Requêtes par seconde

Métrique de throughput base de données.

📌 **Contexte** : Performance, monitoring  
💡 **Monitoring** :
```sql
SHOW GLOBAL STATUS LIKE 'Questions';
-- Calculer QPS sur intervalle
```
💡 **Typique** : Dizaines à dizaines de milliers selon workload  
🔗 **Voir aussi** : TPS, Performance, Throughput, Monitoring

---

## R

### RAID 🟡
**Redundant Array of Independent Disks** - Matrice redondante de disques indépendants

Technologie combinant plusieurs disques pour performance/redondance.

📌 **Contexte** : Stockage, performance, redondance  
💡 **Niveaux courants** :
- RAID 0: Performance (striping, pas de redondance)
- RAID 1: Mirroring (redondance totale)
- RAID 5: Striping + parité (balance performance/redondance)
- RAID 10: Mirror + Stripe (recommandé pour DB)

💡 **Recommandation MariaDB** : RAID 10 pour production  
🔗 **Voir aussi** : Storage, Disk, Performance, Redundancy

---

### RAM 🟢
**Random Access Memory** - Mémoire vive

Mémoire volatile rapide.

📌 **Contexte** : Performance, configuration  
💡 **MariaDB** : Buffer Pool utilise RAM (70-80% sur serveur dédié)  
💡 **Configuration** :
```ini
innodb_buffer_pool_size = 8G
```
🔗 **Voir aussi** : Buffer Pool, Memory, Performance

---

### RAG 🔴 🆕
**Retrieval-Augmented Generation** - Génération augmentée par récupération

Architecture IA combinant recherche vectorielle + LLM.

📌 **Contexte** : IA générative, chatbots intelligents  
💡 **Processus** :
1. Question utilisateur → embedding
2. Recherche k-NN dans MariaDB Vector
3. Documents pertinents → contexte LLM
4. LLM génère réponse basée sur contexte

💡 **MariaDB 11.8** : Support natif avec Vector type + HNSW  
🔗 **Voir aussi** : Vector, HNSW, LLM, Semantic Search

---

### RDBMS 🟢
**Relational Database Management System** - Système de gestion de base de données relationnelle

SGBD basé sur le modèle relationnel (tables, SQL).

📌 **Contexte** : Catégorie de SGBD  
💡 **Exemples** : MariaDB, PostgreSQL, MySQL, Oracle  
💡 **Caractéristiques** : Tables, relations, SQL, ACID  
🔗 **Voir aussi** : DBMS, SQL, Relational Model

---

### REGEX 🟡
**Regular Expression** - Expression régulière

Pattern de recherche dans chaînes de caractères.

📌 **Contexte** : Recherche, validation, extraction  
💡 **MariaDB** :
```sql
SELECT * FROM users WHERE email REGEXP '^[a-z]+@[a-z]+\\.com$';
SELECT REGEXP_REPLACE(text, '[0-9]+', 'X') FROM data;
```
🔗 **Voir aussi** : REGEXP, Pattern Matching, String Functions

---

### REST 🟡
**Representational State Transfer** - Transfert d'état représentationnel

Style architectural pour APIs web.

📌 **Contexte** : APIs, intégration  
💡 **Méthodes** : GET, POST, PUT, DELETE, PATCH  
💡 **MariaDB** : Accès via REST APIs (middleware requis)  
🔗 **Voir aussi** : API, HTTP, Integration

---

### REVOKE 🟢
**REVOKE** - Révoquer

Commande SQL pour retrait de privilèges.

📌 **Contexte** : Sécurité, contrôle d'accès (partie de DCL)  
💡 **Exemple** :
```sql
REVOKE INSERT, UPDATE ON mydb.* FROM 'user'@'localhost';
```
🔗 **Voir aussi** : GRANT, DCL, Privileges, Security

---

### RO 🟡
**Read-Only** - Lecture seule

Mode où modifications sont interdites.

📌 **Contexte** : Replicas, sécurité  
💡 **Configuration** :
```sql
SET GLOBAL read_only = ON;
SET GLOBAL super_read_only = ON;  -- Même pour SUPER users
```
🔗 **Voir aussi** : Replication, Replica, Read/Write Split

---

### RPO 🟣
**Recovery Point Objective** - Objectif de point de récupération

Quantité maximale de données acceptable à perdre (mesurée en temps).

📌 **Contexte** : Disaster Recovery, SLA  
💡 **Exemple** : RPO = 1 heure → max 1h de données perdues acceptable  
💡 **Impact** : Détermine fréquence backups  
💡 **Calcul** : Intervalle entre sauvegardes  
🔗 **Voir aussi** : RTO, Backup, DR, SLA

---

### RTO 🟣
**Recovery Time Objective** - Objectif de temps de récupération

Temps maximal acceptable pour restaurer service après incident.

📌 **Contexte** : Disaster Recovery, SLA  
💡 **Exemple** : RTO = 4 heures → service restauré sous 4h maximum  
💡 **Impact** : Détermine stratégie HA et DR  
🔗 **Voir aussi** : RPO, DR, High Availability, SLA

---

### RW 🟡
**Read-Write** - Lecture-écriture

Mode permettant lectures et écritures.

📌 **Contexte** : Configuration serveur, réplication  
💡 **Opposé** : Read-Only (RO)  
🔗 **Voir aussi** : Read-Only, Primary, Source

---

## S

### SaaS 🟡
**Software as a Service** - Logiciel en tant que service

Modèle cloud où logiciel est fourni via Internet.

📌 **Contexte** : Cloud computing  
💡 **Exemples** : Gmail, Salesforce, Office 365  
🔗 **Voir aussi** : PaaS, IaaS, Cloud

---

### SAN 🟡
**Storage Area Network** - Réseau de stockage

Réseau dédié au stockage bloc haute performance.

📌 **Contexte** : Stockage enterprise  
💡 **Protocoles** : Fibre Channel, iSCSI  
💡 **Usage MariaDB** : Stockage haute performance  
🔗 **Voir aussi** : NAS, Storage, iSCSI, Performance

---

### SIMD 🔴
**Single Instruction, Multiple Data** - Instruction unique, données multiples

Instructions CPU traitant plusieurs données simultanément.

📌 **Contexte** : Optimisations vectorielles  
💡 **MariaDB Vector** : Optimisations SIMD (AVX2, AVX512, ARM NEON)  
💡 **Performance** : Accélération calculs vectoriels (distances)  
🔗 **Voir aussi** : Vector, HNSW, Performance, CPU

---

### SLA 🟣
**Service Level Agreement** - Accord de niveau de service

Contrat définissant niveau de service attendu.

📌 **Contexte** : Production, garanties  
💡 **Métriques** : Uptime %, RTO, RPO, temps réponse  
💡 **Exemple** : 99.99% uptime = ~52 min downtime/an  
🔗 **Voir aussi** : High Availability, RTO, RPO, SLO

---

### SLO 🟣
**Service Level Objective** - Objectif de niveau de service

Objectifs mesurables composant le SLA.

📌 **Contexte** : Monitoring, performance  
💡 **Exemples** :
- Latence requêtes < 100ms pour 95% requêtes
- Disponibilité > 99.9%
- Temps réponse API < 500ms

🔗 **Voir aussi** : SLA, Monitoring, Performance

---

### SMTP 🟢
**Simple Mail Transfer Protocol** - Protocole simple de transfert de courrier

Protocole d'envoi d'emails.

📌 **Contexte** : Notifications, alerting  
💡 **Usage MariaDB** : Notifications backups, alertes monitoring  
🔗 **Voir aussi** : Email, Alerting, Notification

---

### SQL 🟢
**Structured Query Language** - Langage de requête structuré

Langage standard pour gestion bases relationnelles.

📌 **Contexte** : Fondamental RDBMS  
💡 **Sous-ensembles** : DDL, DML, DCL, TCL  
💡 **Standards** : SQL-92, SQL:1999, SQL:2003, SQL:2023  
🔗 **Voir aussi** : DDL, DML, RDBMS, Query

---

### SRE 🟡
**Site Reliability Engineering** - Ingénierie de fiabilité de site

Discipline appliquant principes software engineering aux opérations.

📌 **Contexte** : DevOps, production  
💡 **Responsabilités** : Automatisation, monitoring, incident response  
🔗 **Voir aussi** : DevOps, DBA, Operations, Automation

---

### SSD 🟢
**Solid-State Drive** - Disque à état solide

Stockage flash sans pièces mécaniques.

📌 **Contexte** : Stockage, performance  
💡 **Avantages** :
- IOPS élevés (10k-100k+)
- Latence faible (<1ms)
- Pas de fragmentation mécanique

💡 **Types** : SATA SSD, NVMe SSD  
💡 **Configuration MariaDB** :
```ini
innodb_io_capacity = 2000
innodb_flush_neighbors = 0  # Optimisation SSD
```
🆕 **MariaDB 11.8** : Cost optimizer amélioré pour SSD  
🔗 **Voir aussi** : HDD, NVMe, IOPS, Performance

---

### SSH 🟢
**Secure Shell** - Shell sécurisé

Protocole de connexion sécurisée à distance.

📌 **Contexte** : Administration serveur  
💡 **Usage** : Connexion aux serveurs MariaDB  
💡 **Port** : 22 (défaut)  
🔗 **Voir aussi** : Security, Remote Access, CLI

---

### SSL 🟡
**Secure Sockets Layer** - Couche de sockets sécurisée

Protocole de chiffrement (remplacé par TLS).

📌 **Contexte** : Sécurité connexions (terme historique)  
💡 **Moderne** : TLS a remplacé SSL  
💡 **Usage courant** : "SSL/TLS" désigne TLS  
🔗 **Voir aussi** : TLS, Encryption, Security

---

### SSO 🟡
**Single Sign-On** - Authentification unique

Système permettant une seule authentification pour multiples services.

📌 **Contexte** : Sécurité, authentification centralisée  
💡 **Protocoles** : SAML, OAuth, Kerberos  
💡 **MariaDB** : Via plugins PAM, LDAP, GSSAPI  
🔗 **Voir aussi** : PAM, LDAP, Authentication, Security

---

### SST 🔴
**State Snapshot Transfer** - Transfert d'instantané d'état

Synchronisation complète d'un nœud Galera Cluster.

📌 **Contexte** : Galera Cluster, synchronisation initiale  
💡 **Quand** :
- Nouveau nœud rejoint cluster
- Nœud trop désynchronisé (IST impossible)
- Corruption détectée

💡 **Méthodes** :
- `rsync` : Rapide mais bloque donneur
- `mysqldump` : Lent, compatible
- `mariabackup` : Recommandé (rapide, non-bloquant)
- `xtrabackup-v2` : Alternative Percona

🔗 **Voir aussi** : IST, Galera Cluster, State Transfer, mariabackup

---

## T

### TB 🟢
**Terabyte** - Téraoctet

Unité de mesure (1 TB = 1024 GB = 1,099,511,627,776 bytes).

📌 **Contexte** : Grandes volumétries  
🔗 **Voir aussi** : GB, MB, Storage

---

### TCP 🟢
**Transmission Control Protocol** - Protocole de contrôle de transmission

Protocole réseau fiable orienté connexion.

📌 **Contexte** : Réseau, connexions MariaDB  
💡 **Port MariaDB** : 3306 (défaut)  
💡 **Connexion** :
```bash
mariadb -h hostname -P 3306 -u user -p
```
🔗 **Voir aussi** : IP, Network, Port, Connection

---

### TCL 🟡
**Transaction Control Language** - Langage de contrôle de transaction

Sous-ensemble SQL pour gestion transactions.

📌 **Contexte** : Contrôle transactionnel  
💡 **Commandes** :
- `START TRANSACTION` / `BEGIN` : Démarre transaction
- `COMMIT` : Valide transaction
- `ROLLBACK` : Annule transaction
- `SAVEPOINT` : Point de sauvegarde
- `SET TRANSACTION` : Configure isolation

💡 **Exemple** :
```sql
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
SAVEPOINT sp1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```
🔗 **Voir aussi** : DDL, DML, Transaction, ACID

---

### TLS 🟡
**Transport Layer Security** - Sécurité de la couche de transport

Protocole de chiffrement des communications réseau.

📌 **Contexte** : Sécurité connexions  
💡 **Versions** : TLS 1.2, TLS 1.3 (actuel)  
🆕 **MariaDB 11.8** : TLS activé par défaut  
💡 **Configuration** :
```ini
ssl-cert = /etc/mysql/server-cert.pem
ssl-key = /etc/mysql/server-key.pem
ssl-ca = /etc/mysql/ca-cert.pem
require_secure_transport = ON
```
🔗 **Voir aussi** : SSL, Encryption, Security, Certificate

---

### TPS 🟡
**Transactions Per Second** - Transactions par seconde

Métrique de throughput transactionnel.

📌 **Contexte** : Performance OLTP, monitoring  
💡 **Monitoring** :
```sql
SHOW GLOBAL STATUS LIKE 'Com_commit';
SHOW GLOBAL STATUS LIKE 'Com_rollback';
```
🔗 **Voir aussi** : QPS, Performance, OLTP, Throughput

---

### TTL 🟡
**Time To Live** - Durée de vie

Durée de validité d'une donnée ou connexion.

📌 **Contexte** : Cache, DNS, sessions  
💡 **Exemples** :
- Cache expiration
- Session timeout
- DNS record validity

🔗 **Voir aussi** : Cache, Expiration, Timeout

---

## U

### UCA 🟡 🆕
**Unicode Collation Algorithm** - Algorithme de collation Unicode

Algorithme standard pour tri et comparaison Unicode.

📌 **Contexte** : Internationalisation, tri multilingue  
🆕 **MariaDB 11.8** : UCA 14.0.0 par défaut pour utf8mb4  
💡 **Collations** : `utf8mb4_unicode_ci`, `utf8mb4_unicode_520_ci`  
🔗 **Voir aussi** : Collation, Unicode, utf8mb4, Charset

---

### UDP 🟢
**User Datagram Protocol** - Protocole de datagramme utilisateur

Protocole réseau non-fiable sans connexion.

📌 **Contexte** : Réseau (MariaDB utilise principalement TCP)  
🔗 **Voir aussi** : TCP, Network, Protocol

---

### UI 🟢
**User Interface** - Interface utilisateur

Interface pour interaction utilisateur (GUI ou CLI).

📌 **Contexte** : Outils, administration  
🔗 **Voir aussi** : GUI, CLI, Tools

---

### UK 🟢
**Unique Key** - Clé unique

Contrainte garantissant unicité (synonyme UNIQUE).

📌 **Contexte** : Intégrité des données  
💡 **Exemple** :
```sql
CREATE TABLE users (
  id INT PRIMARY KEY,
  email VARCHAR(255) UNIQUE  -- UK
);
```
🔗 **Voir aussi** : Unique Index, Constraint, Primary Key

---

### URI 🟢
**Uniform Resource Identifier** - Identifiant uniforme de ressource

Chaîne identifiant une ressource.

📌 **Contexte** : Connexions, APIs  
💡 **Format connexion** : `mariadb://user:pass@host:port/database`  
🔗 **Voir aussi** : URL, Connection String, DSN

---

### URL 🟢
**Uniform Resource Locator** - Localisateur uniforme de ressource

URI spécifiant comment accéder à une ressource.

📌 **Contexte** : Web, connexions  
🔗 **Voir aussi** : URI, HTTP, Connection

---

### UTC 🟢
**Coordinated Universal Time** - Temps universel coordonné

Standard de temps international (fuseau horaire de référence).

📌 **Contexte** : Timestamps, internationalisation  
💡 **MariaDB** :
```sql
SELECT NOW();  -- Heure locale
SELECT UTC_TIMESTAMP();  -- Heure UTC
SET time_zone = '+00:00';  -- Configurer UTC
```
💡 **Best practice** : Stocker dates en UTC  
🔗 **Voir aussi** : TIMESTAMP, DATETIME, Timezone

---

### UTF-8 🟢
**Unicode Transformation Format - 8-bit** - Format de transformation Unicode 8 bits

Encodage de caractères Unicode variable (1-4 bytes).

📌 **Contexte** : Encodage texte international  
💡 **MariaDB** : Utiliser `utf8mb4` (vrai UTF-8)  
⚠️ **Attention** : `utf8` dans MariaDB = max 3 bytes (incomplet)  
🆕 **MariaDB 11.8** : utf8mb4 par défaut  
🔗 **Voir aussi** : utf8mb4, Charset, Unicode, UCA

---

### UUID 🟡
**Universally Unique Identifier** - Identifiant unique universel

Identifiant 128-bit pseudo-aléatoire globalement unique.

📌 **Contexte** : Identifiants distribués  
💡 **Format** : `550e8400-e29b-41d4-a716-446655440000`  
💡 **Génération** :
```sql
SELECT UUID();  -- Retourne UUID version 1
SELECT UUID_SHORT();  -- Retourne entier 64-bit plus compact
```
💡 **Stockage** :
```sql
-- Binaire optimisé :
id BINARY(16) DEFAULT (UNHEX(REPLACE(UUID(),'-','')))
-- Ou lisible :
id CHAR(36) DEFAULT (UUID())
```
🔗 **Voir aussi** : Primary Key, GUID, Identifier

---

## V

### VIP 🟡
**Virtual IP** - IP virtuelle

Adresse IP flottante pouvant être transférée entre serveurs.

📌 **Contexte** : High Availability, failover  
💡 **Outils** : keepalived, Pacemaker  
💡 **Usage** : Adresse stable pour clients lors de failover  
🔗 **Voir aussi** : High Availability, Failover, keepalived

---

### VM 🟢
**Virtual Machine** - Machine virtuelle

Système informatique émulé sur hardware physique.

📌 **Contexte** : Virtualisation, cloud  
💡 **Hyperviseurs** : VMware, KVM, Hyper-V, VirtualBox  
💡 **Cloud** : AWS EC2, Azure VMs, Google Compute Engine  
🔗 **Voir aussi** : Virtualization, Cloud, IaaS

---

### VPN 🟢
**Virtual Private Network** - Réseau privé virtuel

Réseau sécurisé sur infrastructure publique.

📌 **Contexte** : Sécurité, accès distant  
💡 **Usage MariaDB** : Connexions sécurisées inter-sites  
🔗 **Voir aussi** : Security, Network, Encryption

---

## W

### WAN 🟢
**Wide Area Network** - Réseau étendu

Réseau couvrant grande zone géographique.

📌 **Contexte** : Réseau, géo-distribution  
💡 **Usage MariaDB** : Réplication inter-datacenter  
🔗 **Voir aussi** : LAN, Network, Geo-Distribution

---

### WHO 🔴
**Write-Heavy Optimization** - Optimisation orientée écriture

Optimisations pour workloads à forte écriture.

📌 **Contexte** : Performance tuning  
💡 **Techniques** :
- Batch inserts
- Désactivation index temporairement
- Ajustement innodb_flush_log_at_trx_commit

🔗 **Voir aussi** : Performance, Write Workload, Tuning

---

## X

### XA 🔴
**eXtended Architecture** - Architecture étendue (transactions distribuées)

Standard pour transactions distribuées (2-Phase Commit).

📌 **Contexte** : Transactions cross-database  
💡 **Phases** :
1. PREPARE : Toutes ressources préparent
2. COMMIT/ROLLBACK : Décision globale

⚠️ **Limitations** : Complexe, lent, éviter si possible  
🔗 **Voir aussi** : Distributed Transaction, 2PC, Transaction

---

### XML 🟡
**eXtensible Markup Language** - Langage de balisage extensible

Format de données structurées textuelles.

📌 **Contexte** : Export/import, intégration legacy  
💡 **MariaDB** : Fonctions XML limitées, préférer JSON  
🔗 **Voir aussi** : JSON, Data Format, Integration

---

## Y

### YAML 🟡
**YAML Ain't Markup Language** - YAML n'est pas un langage de balisage

Format de sérialisation de données lisible.

📌 **Contexte** : Configuration, IaC, Kubernetes  
💡 **Usage** :
- Fichiers config Docker Compose
- Manifests Kubernetes
- Playbooks Ansible
- Pipelines CI/CD

💡 **Exemple** :
```yaml
mariadb:
  image: mariadb:11.8
  environment:
    MYSQL_ROOT_PASSWORD: secret
  volumes:
    - mariadb-data:/var/lib/mysql
```
🔗 **Voir aussi** : Docker, Kubernetes, Configuration, IaC

---

## Z

### ZIP 🟢
**ZIP** - Compression ZIP

Format de compression de fichiers.

📌 **Contexte** : Compression backups, archives  
💡 **MariaDB** : Compression backups mysqldump  
💡 **Exemple** :
```bash
mariadb-dump mydb | gzip > backup.sql.gz
```
🔗 **Voir aussi** : Compression, Backup, Archive

---

## ✅ Points Clés à Retenir

### SQL et Commandes
- **DDL** : Structure (CREATE, ALTER, DROP)
- **DML** : Données (SELECT, INSERT, UPDATE, DELETE)
- **DCL** : Droits (GRANT, REVOKE)
- **TCL** : Transactions (COMMIT, ROLLBACK)

### Clés et Contraintes
- **PK** : Primary Key (identifiant unique)
- **FK** : Foreign Key (référence autre table)
- **UK** : Unique Key (valeurs uniques)
- **NN** : Not Null (valeur obligatoire)

### Réplication
- **GTID** : Identifiant global transaction
- **SST** : Transfert complet état (Galera)
- **IST** : Transfert incrémental (Galera)
- **PITR** : Restauration point-in-time

### Performance
- **OLTP** : Transactions courtes et fréquentes
- **OLAP** : Analyses volumineuses
- **IOPS** : Opérations I/O par seconde
- **QPS** : Requêtes par seconde
- **TPS** : Transactions par seconde

### Haute Disponibilité
- **HA** : High Availability
- **DR** : Disaster Recovery
- **RTO** : Recovery Time Objective
- **RPO** : Recovery Point Objective
- **SLA** : Service Level Agreement

### DevOps et Cloud
- **CI/CD** : Intégration/Déploiement continu
- **IaC** : Infrastructure as Code
- **K8s** : Kubernetes
- **PaaS** : Platform as a Service

### IA/ML (Nouveautés 11.8)
- **RAG** : Retrieval-Augmented Generation
- **HNSW** : Index recherche vectorielle
- **k-NN** : k plus proches voisins
- **LLM** : Large Language Model
- **MCP** : Model Context Protocol

---

## 🔗 Références

### Documentation Officielle
- [MariaDB Glossary](https://mariadb.com/kb/en/mariadb-glossary/)
- [SQL Standards](https://www.iso.org/standard/76583.html)

### Standards et Spécifications
- [RFC 3986 - URI](https://www.rfc-editor.org/rfc/rfc3986)
- [Unicode Standard](https://unicode.org/standard/standard.html)
- [ISO SQL](https://www.iso.org/standard/76583.html)

---

## ➡️ Retour au Sommaire

**[← A.1 Termes MariaDB Essentiels](./01-termes-mariadb-essentiels.md)**  
Consultez les définitions détaillées des concepts fondamentaux

**[← A. Glossaire - Introduction](./README.md)**  
Retour à l'introduction du glossaire

---

**MariaDB** : 11.8 LTS  
**Acronymes définis** : 140+ abréviations techniques

⏭️ [Commandes mariadb CLI Essentielles](/annexes/commandes-cli/README.md)
