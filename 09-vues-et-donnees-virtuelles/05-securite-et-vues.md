🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.5 Sécurité et vues : Masquage de données

> **Niveau** : Intermédiaire
> **Durée estimée** : 2 heures
> **Prérequis** : Sections 9.1-9.4, Chapitre 10.1-10.3 (Gestion des utilisateurs et privilèges)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Utiliser les vues comme mécanisme de contrôle d'accès et masquage de données
- Masquer des colonnes sensibles tout en permettant l'accès aux données publiques
- Implémenter un filtrage de lignes par utilisateur, rôle ou tenant
- Combiner vues, privilèges et WITH CHECK OPTION pour une sécurité robuste
- Comprendre les implications de DEFINER et SQL SECURITY sur la sécurité
- Concevoir une architecture de sécurité multi-niveaux avec des vues
- Identifier les failles de sécurité courantes et les prévenir
- Mettre en place des audits et traces pour les accès via vues

---

## Introduction

### Les vues comme mécanisme de sécurité

Les vues constituent un **outil puissant** pour implémenter des stratégies de sécurité dans MariaDB. Elles permettent de créer une **couche d'abstraction** entre les utilisateurs et les données sensibles, en contrôlant précisément ce qui peut être vu et modifié.

**Principe fondamental** : Au lieu de donner un accès direct aux tables (qui expose toutes les données), on crée des vues qui exposent uniquement les données autorisées, puis on accorde les privilèges sur ces vues.

```sql
-- ❌ Approche non sécurisée : accès direct à la table
GRANT SELECT ON database.employes TO 'app_user'@'%';
-- L'utilisateur voit TOUTES les colonnes : nom, salaire, num_secu, etc.

-- ✅ Approche sécurisée : accès via une vue
CREATE VIEW v_employes_publics AS
SELECT id, nom, prenom, email, departement
FROM employes;
-- Masque : salaire, num_secu, date_naissance, adresse, etc.

GRANT SELECT ON database.v_employes_publics TO 'app_user'@'%';
-- L'utilisateur ne voit que les colonnes non sensibles
```

### Cas d'usage de sécurité

Les vues sont utilisées pour :

1. **Masquage de colonnes** : Cacher des informations sensibles (salaires, mots de passe, données personnelles)
2. **Filtrage de lignes** : Restreindre l'accès à un sous-ensemble de données (par département, par tenant, par rôle)
3. **Contrôle des modifications** : Empêcher certaines opérations (avec vues non-updatable)
4. **Anonymisation** : Transformer les données sensibles (hachage, troncature, agrégation)
5. **Audit** : Tracer les accès aux données sensibles

---

## Masquage de colonnes sensibles

### Principe de base

Le masquage de colonnes consiste à créer une vue qui **exclut** les colonnes sensibles de la table sous-jacente.

### Exemple 1 : Table utilisateurs avec données sensibles

```sql
-- Table complète avec toutes les données
CREATE TABLE utilisateurs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,  -- SENSIBLE
    password_salt VARCHAR(255) NOT NULL,  -- SENSIBLE
    num_securite_sociale VARCHAR(15),     -- SENSIBLE
    date_naissance DATE,                  -- SENSIBLE
    adresse_complete TEXT,                -- SENSIBLE
    telephone VARCHAR(20),
    role ENUM('USER', 'MODERATOR', 'ADMIN') DEFAULT 'USER',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login TIMESTAMP NULL,
    is_active BOOLEAN DEFAULT TRUE
) ENGINE=InnoDB;

-- Vue publique : masque les données sensibles
CREATE VIEW v_utilisateurs_publics AS
SELECT
    id,
    username,
    email,
    role,
    created_at,
    last_login,
    is_active
    -- Colonnes exclues : password_hash, password_salt, num_securite_sociale,
    --                    date_naissance, adresse_complete
FROM utilisateurs;

-- Vue pour les administrateurs : expose plus de données
CREATE VIEW v_utilisateurs_admin AS
SELECT
    id,
    username,
    email,
    telephone,
    date_naissance,
    role,
    created_at,
    last_login,
    is_active
    -- Toujours masqué : password_hash, password_salt, num_securite_sociale
FROM utilisateurs;

-- Attribution des privilèges
GRANT SELECT ON database.v_utilisateurs_publics TO 'app_user'@'%';
GRANT SELECT ON database.v_utilisateurs_admin TO 'app_admin'@'%';

-- Les utilisateurs ne peuvent jamais accéder aux mots de passe
-- même les admins !
```

### Exemple 2 : Table employés avec salaires

```sql
-- Table employés complète
CREATE TABLE employes (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nom VARCHAR(100),
    prenom VARCHAR(100),
    email VARCHAR(255),
    departement VARCHAR(50),
    poste VARCHAR(100),
    salaire_brut DECIMAL(10,2),      -- SENSIBLE
    bonus_annuel DECIMAL(10,2),      -- SENSIBLE
    date_embauche DATE,
    manager_id INT,
    FOREIGN KEY (manager_id) REFERENCES employes(id)
) ENGINE=InnoDB;

-- Vue pour les employés : masque les salaires
CREATE VIEW v_annuaire_employes AS
SELECT
    id,
    nom,
    prenom,
    email,
    departement,
    poste,
    manager_id
    -- Masque : salaire_brut, bonus_annuel
FROM employes;

-- Vue pour les RH : accès aux salaires
CREATE VIEW v_employes_rh AS
SELECT
    id,
    nom,
    prenom,
    email,
    departement,
    poste,
    salaire_brut,
    bonus_annuel,
    date_embauche,
    manager_id
FROM employes;

-- Vue pour les managers : salaires de leur équipe uniquement
CREATE VIEW v_employes_manager AS
SELECT
    e.id,
    e.nom,
    e.prenom,
    e.email,
    e.departement,
    e.poste,
    e.salaire_brut,
    e.manager_id
FROM employes e
WHERE e.manager_id = (SELECT id FROM employes WHERE email = CURRENT_USER());

-- Privilèges différenciés
GRANT SELECT ON database.v_annuaire_employes TO 'employe'@'%';
GRANT SELECT, UPDATE ON database.v_employes_rh TO 'rh'@'%';
GRANT SELECT ON database.v_employes_manager TO 'manager'@'%';
```

### Transformation et anonymisation de données

Parfois, il faut exposer une donnée mais de manière **transformée** :

```sql
-- Vue avec données partiellement masquées
CREATE VIEW v_clients_anonymises AS
SELECT
    id,
    nom,
    -- Email partiel : john.doe@example.com → j***e@e***.com
    CONCAT(
        LEFT(email, 1),
        REPEAT('*', LENGTH(SUBSTRING_INDEX(email, '@', 1)) - 2),
        RIGHT(SUBSTRING_INDEX(email, '@', 1), 1),
        '@',
        LEFT(SUBSTRING_INDEX(email, '@', -1), 1),
        REPEAT('*', LENGTH(SUBSTRING_INDEX(email, '@', -1)) - 5),
        RIGHT(SUBSTRING_INDEX(email, '@', -1), 4)
    ) AS email_masque,
    -- Téléphone partiel : 0612345678 → 06****5678
    CONCAT(
        LEFT(telephone, 2),
        REPEAT('*', LENGTH(telephone) - 6),
        RIGHT(telephone, 4)
    ) AS telephone_masque,
    -- Carte bancaire : 1234567890123456 → 1234********3456
    CONCAT(
        LEFT(carte_bancaire, 4),
        REPEAT('*', 8),
        RIGHT(carte_bancaire, 4)
    ) AS carte_masquee,
    -- Date de naissance : année uniquement
    YEAR(date_naissance) AS annee_naissance,
    -- Adresse : ville et code postal uniquement
    ville,
    code_postal
FROM clients;

-- Grant en lecture seule
GRANT SELECT ON database.v_clients_anonymises TO 'support_niveau1'@'%';
```

---

## Filtrage de lignes par utilisateur ou contexte

### Filtrage par utilisateur connecté

Les vues peuvent filtrer automatiquement les lignes selon l'utilisateur :

```sql
-- Table documents multi-utilisateurs
CREATE TABLE documents (
    id INT AUTO_INCREMENT PRIMARY KEY,
    titre VARCHAR(255),
    contenu TEXT,
    proprietaire_id INT,
    date_creation TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_proprietaire (proprietaire_id)
) ENGINE=InnoDB;

-- Table utilisateurs
CREATE TABLE users (
    id INT PRIMARY KEY,
    username VARCHAR(50) UNIQUE,
    email VARCHAR(255)
) ENGINE=InnoDB;

-- Vue : chaque utilisateur voit uniquement ses documents
CREATE DEFINER = 'admin'@'localhost'
    SQL SECURITY INVOKER
    VIEW v_mes_documents AS
SELECT
    d.id,
    d.titre,
    d.contenu,
    d.date_creation
FROM documents d
INNER JOIN users u ON d.proprietaire_id = u.id
WHERE u.username = SUBSTRING_INDEX(USER(), '@', 1);
-- USER() retourne 'username@hostname'

-- Grant sur la vue (pas sur la table)
GRANT SELECT, INSERT, UPDATE, DELETE ON database.v_mes_documents TO 'app_user'@'%';

-- L'utilisateur 'alice' connecté voit uniquement ses documents
-- SELECT * FROM v_mes_documents;
-- → Uniquement les documents où proprietaire_id = id d'alice
```

💡 **Important** : Utilisez `SQL SECURITY INVOKER` pour que la vue s'exécute avec les privilèges de l'utilisateur appelant, pas du créateur.

### Filtrage par rôle

```sql
-- Table avec niveau de confidentialité
CREATE TABLE fichiers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nom VARCHAR(255),
    contenu BLOB,
    niveau_securite ENUM('PUBLIC', 'CONFIDENTIEL', 'SECRET', 'TOP_SECRET'),
    departement VARCHAR(50),
    INDEX idx_niveau (niveau_securite)
) ENGINE=InnoDB;

-- Vue pour utilisateurs standards : uniquement PUBLIC
CREATE VIEW v_fichiers_publics AS
SELECT id, nom, departement
FROM fichiers
WHERE niveau_securite = 'PUBLIC';

-- Vue pour utilisateurs avec habilitation : PUBLIC + CONFIDENTIEL
CREATE VIEW v_fichiers_confidentiels AS
SELECT id, nom, contenu, niveau_securite, departement
FROM fichiers
WHERE niveau_securite IN ('PUBLIC', 'CONFIDENTIEL');

-- Vue pour utilisateurs SECRET : PUBLIC + CONFIDENTIEL + SECRET
CREATE VIEW v_fichiers_secrets AS
SELECT id, nom, contenu, niveau_securite, departement
FROM fichiers
WHERE niveau_securite IN ('PUBLIC', 'CONFIDENTIEL', 'SECRET');

-- Attribution selon les rôles
GRANT SELECT ON database.v_fichiers_publics TO 'role_standard'@'%';
GRANT SELECT ON database.v_fichiers_confidentiels TO 'role_habilite'@'%';
GRANT SELECT ON database.v_fichiers_secrets TO 'role_secret_defense'@'%';
```

### Filtrage multi-tenant

Isoler les données par tenant (client, entreprise, organisation) :

```sql
-- Table multi-tenant
CREATE TABLE produits (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nom VARCHAR(100),
    prix DECIMAL(10,2),
    tenant_id INT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_tenant (tenant_id)
) ENGINE=InnoDB;

-- Table de mapping utilisateur → tenant
CREATE TABLE user_tenants (
    user_id INT,
    tenant_id INT,
    PRIMARY KEY (user_id, tenant_id)
) ENGINE=InnoDB;

-- Vue qui filtre automatiquement par tenant de l'utilisateur
CREATE DEFINER = 'admin'@'localhost'
    SQL SECURITY INVOKER
    VIEW v_mes_produits AS
SELECT
    p.id,
    p.nom,
    p.prix,
    p.created_at
FROM produits p
WHERE p.tenant_id IN (
    SELECT ut.tenant_id
    FROM user_tenants ut
    INNER JOIN mysql.user u ON ut.user_id = u.user_id
    WHERE u.user = SUBSTRING_INDEX(USER(), '@', 1)
);

-- Avec WITH CHECK OPTION pour empêcher l'accès à d'autres tenants
CREATE OR REPLACE DEFINER = 'admin'@'localhost'
    SQL SECURITY INVOKER
    VIEW v_mes_produits AS
SELECT
    p.id,
    p.nom,
    p.prix,
    p.tenant_id,
    p.created_at
FROM produits p
WHERE p.tenant_id IN (
    SELECT ut.tenant_id
    FROM user_tenants ut
    INNER JOIN mysql.user u ON ut.user_id = u.user_id
    WHERE u.user = SUBSTRING_INDEX(USER(), '@', 1)
)
WITH CHECK OPTION;

-- Les utilisateurs ne peuvent modifier que leurs propres données
GRANT SELECT, INSERT, UPDATE, DELETE ON database.v_mes_produits TO 'app_user'@'%';
```

### Filtrage avec variables de session

Alternative moderne : utiliser des variables de session pour le contexte :

```sql
-- Vue qui filtre par variable de session
CREATE VIEW v_commandes_tenant AS
SELECT
    id,
    numero_commande,
    date_commande,
    montant_total,
    statut
FROM commandes
WHERE tenant_id = @current_tenant_id;

-- L'application définit le tenant au début de la session
SET @current_tenant_id = 42;

-- Toutes les requêtes sont automatiquement filtrées
SELECT * FROM v_commandes_tenant;
-- → Uniquement les commandes du tenant 42

-- Pour changer de tenant (si autorisé)
SET @current_tenant_id = 99;
SELECT * FROM v_commandes_tenant;
-- → Commandes du tenant 99
```

---

## Combiner vues et privilèges pour la sécurité

### Architecture de sécurité multi-niveaux

```sql
-- 1. Table principale (jamais accédée directement par les applications)
CREATE TABLE donnees_sensibles (
    id INT AUTO_INCREMENT PRIMARY KEY,
    reference VARCHAR(50),
    donnee_publique VARCHAR(255),
    donnee_confidentielle TEXT,
    donnee_secrete TEXT,
    niveau ENUM('PUBLIC', 'CONFIDENTIEL', 'SECRET')
) ENGINE=InnoDB;

-- 2. Vue niveau PUBLIC
CREATE DEFINER = 'security_admin'@'localhost'
    SQL SECURITY DEFINER
    VIEW v_niveau_public AS
SELECT
    id,
    reference,
    donnee_publique
FROM donnees_sensibles
WHERE niveau = 'PUBLIC';

-- 3. Vue niveau CONFIDENTIEL
CREATE DEFINER = 'security_admin'@'localhost'
    SQL SECURITY DEFINER
    VIEW v_niveau_confidentiel AS
SELECT
    id,
    reference,
    donnee_publique,
    donnee_confidentielle
FROM donnees_sensibles
WHERE niveau IN ('PUBLIC', 'CONFIDENTIEL');

-- 4. Vue niveau SECRET (tout)
CREATE DEFINER = 'security_admin'@'localhost'
    SQL SECURITY DEFINER
    VIEW v_niveau_secret AS
SELECT
    id,
    reference,
    donnee_publique,
    donnee_confidentielle,
    donnee_secrete
FROM donnees_sensibles;

-- 5. Privilèges strictement définis
-- Aucun privilège sur la table de base
REVOKE ALL PRIVILEGES ON database.donnees_sensibles FROM 'public'@'%';

-- Privilèges par niveau
GRANT SELECT ON database.v_niveau_public TO 'role_public'@'%';
GRANT SELECT ON database.v_niveau_confidentiel TO 'role_confidentiel'@'%';
GRANT SELECT ON database.v_niveau_secret TO 'role_secret'@'%';

-- Seul l'admin de sécurité a accès direct à la table
GRANT ALL PRIVILEGES ON database.donnees_sensibles TO 'security_admin'@'localhost';
```

### Privilèges granulaires sur les vues

```sql
-- Vue pour consultation
CREATE VIEW v_clients_lecture AS
SELECT id, nom, email, telephone
FROM clients;

-- Vue pour modification (avec WITH CHECK OPTION)
CREATE VIEW v_clients_modification AS
SELECT id, nom, email, telephone, statut
FROM clients
WHERE statut = 'ACTIF'
WITH CHECK OPTION;

-- Rôle lecture seule
GRANT SELECT ON database.v_clients_lecture TO 'role_lecture_seule'@'%';

-- Rôle modification limitée
GRANT SELECT, UPDATE ON database.v_clients_modification TO 'role_edition_limitee'@'%';

-- Rôle gestion complète (mais via la vue)
GRANT SELECT, INSERT, UPDATE, DELETE ON database.v_clients_modification TO 'role_gestion'@'%';
```

---

## DEFINER et SQL SECURITY : Implications de sécurité

### SQL SECURITY DEFINER : Élévation de privilèges

```sql
-- Cas d'usage : Permettre à un utilisateur d'accéder à des données
-- auxquelles il n'a normalement pas accès

-- Table accessible uniquement à l'admin
CREATE TABLE logs_systeme (
    id INT AUTO_INCREMENT PRIMARY KEY,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    user VARCHAR(50),
    action VARCHAR(100),
    details TEXT
) ENGINE=InnoDB;

-- Seul root a accès
GRANT ALL ON database.logs_systeme TO 'root'@'localhost';

-- Vue créée par root, accessible aux auditeurs
CREATE DEFINER = 'root'@'localhost'
    SQL SECURITY DEFINER
    VIEW v_logs_auditeurs AS
SELECT
    id,
    timestamp,
    user,
    action
    -- Masque : details (peut contenir des infos sensibles)
FROM logs_systeme
WHERE timestamp >= DATE_SUB(NOW(), INTERVAL 30 DAY);

-- Grant sur la vue (pas sur la table)
GRANT SELECT ON database.v_logs_auditeurs TO 'auditeur'@'%';

-- Comportement :
-- 1. 'auditeur' exécute : SELECT * FROM v_logs_auditeurs;
-- 2. La vue s'exécute avec les privilèges de 'root'
-- 3. 'auditeur' peut voir les logs sans avoir accès direct à la table
```

**Avantages de DEFINER** :
- ✅ Contrôle d'accès centralisé
- ✅ Les utilisateurs n'ont pas besoin de privilèges directs sur les tables
- ✅ Simplification de la gestion des privilèges

**Risques de DEFINER** :
- ⚠️ Si le DEFINER est supprimé, la vue ne fonctionne plus
- ⚠️ Si le DEFINER perd ses privilèges, la vue cesse de fonctionner
- ⚠️ Potentiel d'élévation de privilèges si mal configuré

### SQL SECURITY INVOKER : Privilèges de l'utilisateur

```sql
-- Cas d'usage : Chaque utilisateur utilise ses propres privilèges

-- Vue qui nécessite que l'utilisateur ait les privilèges
CREATE SQL SECURITY INVOKER
    VIEW v_mes_commandes AS
SELECT
    c.id,
    c.numero,
    c.date_commande,
    c.montant
FROM commandes c
INNER JOIN clients cl ON c.client_id = cl.id
WHERE cl.email = SUBSTRING_INDEX(USER(), '@', 1);

-- Avec INVOKER, chaque utilisateur doit avoir :
GRANT SELECT ON database.commandes TO 'user'@'%';
GRANT SELECT ON database.clients TO 'user'@'%';

-- Avantage : Pas de dépendance à un DEFINER spécifique
-- Inconvénient : Gestion plus complexe des privilèges
```

### Comparaison DEFINER vs INVOKER pour la sécurité

| Aspect | SQL SECURITY DEFINER | SQL SECURITY INVOKER |
|--------|---------------------|---------------------|
| **Privilèges utilisés** | Ceux du créateur (DEFINER) | Ceux de l'utilisateur appelant |
| **Accès aux tables** | Via privilèges du DEFINER | Utilisateur doit avoir les privilèges |
| **Élévation de privilèges** | ✅ Possible (contrôlée) | ❌ Non |
| **Gestion des privilèges** | ✅ Centralisée (simple) | ⚠️ Décentralisée (complexe) |
| **Dépendance au DEFINER** | ⚠️ Oui (si supprimé, vue KO) | ✅ Non |
| **Sécurité** | ⚠️ Risque si mal configuré | ✅ Plus sûr (principe du moindre privilège) |
| **Cas d'usage** | Masquage de données, audit | Vues génériques, filtrage par utilisateur |

💡 **Recommandation** : Utilisez DEFINER pour les vues de sécurité où vous voulez un contrôle centralisé, et INVOKER pour les vues où chaque utilisateur doit avoir ses propres privilèges.

---

## WITH CHECK OPTION pour la sécurité en écriture

### Empêcher les modifications hors périmètre

```sql
-- Vue : employés du département IT
CREATE VIEW v_employes_it AS
SELECT id, nom, prenom, email, departement, salaire
FROM employes
WHERE departement = 'IT'
WITH CHECK OPTION;

-- Grant avec modification
GRANT SELECT, UPDATE ON database.v_employes_it TO 'manager_it'@'%';

-- ✅ UPDATE valide : reste dans IT
UPDATE v_employes_it
SET salaire = 60000
WHERE id = 100;

-- ❌ UPDATE invalide : change le département hors IT
UPDATE v_employes_it
SET departement = 'RH'
WHERE id = 100;
-- Error: CHECK OPTION failed
-- → Empêche un manager IT de déplacer un employé vers RH
```

### Sécurité multi-tenant en écriture

```sql
-- Vue avec filtrage et validation tenant
CREATE DEFINER = 'admin'@'localhost'
    SQL SECURITY INVOKER
    VIEW v_documents_tenant AS
SELECT
    id,
    titre,
    contenu,
    tenant_id,
    created_at
FROM documents
WHERE tenant_id = @current_tenant_id
WITH CASCADED CHECK OPTION;

-- Grant avec modification
GRANT SELECT, INSERT, UPDATE, DELETE ON database.v_documents_tenant TO 'app_user'@'%';

-- L'application définit le tenant
SET @current_tenant_id = 42;

-- ✅ INSERT valide : tenant_id = 42
INSERT INTO v_documents_tenant (titre, contenu, tenant_id)
VALUES ('Doc', 'Contenu', 42);

-- ❌ INSERT invalide : tenant_id différent
INSERT INTO v_documents_tenant (titre, contenu, tenant_id)
VALUES ('Piratage', 'Contenu', 99);
-- Error: CHECK OPTION failed
-- → Empêche un utilisateur d'insérer des données pour un autre tenant
```

---

## Patterns de sécurité avancés

### Pattern 1 : Vues en cascade (chained views)

Créer plusieurs niveaux de vues pour une sécurité progressive :

```sql
-- Niveau 1 : Vue de base (filtrage principal)
CREATE VIEW v_base_transactions AS
SELECT
    id,
    date_transaction,
    montant,
    type_transaction,
    client_id,
    compte_id,
    statut
FROM transactions
WHERE statut != 'SUPPRIME';

-- Niveau 2 : Vue par type (sur v_base)
CREATE VIEW v_transactions_debit AS
SELECT * FROM v_base_transactions
WHERE type_transaction = 'DEBIT';

CREATE VIEW v_transactions_credit AS
SELECT * FROM v_base_transactions
WHERE type_transaction = 'CREDIT';

-- Niveau 3 : Vue avec masquage montant (sur vues niveau 2)
CREATE VIEW v_transactions_anonymisees AS
SELECT
    id,
    date_transaction,
    -- Montant arrondi à la dizaine
    ROUND(montant, -1) AS montant_approx,
    type_transaction,
    statut
FROM v_base_transactions;

-- Attribution différenciée
GRANT SELECT ON database.v_transactions_anonymisees TO 'role_analytique'@'%';
GRANT SELECT ON database.v_transactions_debit TO 'role_comptable'@'%';
GRANT SELECT ON database.v_base_transactions TO 'role_financier'@'%';
```

### Pattern 2 : Row-Level Security (RLS) avec vues

Simuler le Row-Level Security (comme PostgreSQL) avec des vues :

```sql
-- Table avec politique de sécurité implicite
CREATE TABLE projets (
    id INT PRIMARY KEY,
    nom VARCHAR(100),
    description TEXT,
    budget DECIMAL(15,2),
    proprietaire_id INT,
    niveau_confidentialite ENUM('PUBLIC', 'INTERNE', 'CONFIDENTIEL')
) ENGINE=InnoDB;

-- Table de permissions utilisateur-projet
CREATE TABLE projet_permissions (
    projet_id INT,
    user_id INT,
    can_read BOOLEAN DEFAULT FALSE,
    can_write BOOLEAN DEFAULT FALSE,
    can_delete BOOLEAN DEFAULT FALSE,
    PRIMARY KEY (projet_id, user_id)
) ENGINE=InnoDB;

-- Vue RLS : utilisateur voit uniquement ses projets autorisés
CREATE DEFINER = 'admin'@'localhost'
    SQL SECURITY INVOKER
    VIEW v_mes_projets_autorises AS
SELECT
    p.id,
    p.nom,
    p.description,
    CASE
        -- Masquer le budget si niveau CONFIDENTIEL et pas propriétaire
        WHEN p.niveau_confidentialite = 'CONFIDENTIEL'
             AND p.proprietaire_id != CURRENT_USER_ID()
        THEN NULL
        ELSE p.budget
    END AS budget,
    p.niveau_confidentialite
FROM projets p
INNER JOIN projet_permissions pp ON p.id = pp.projet_id
WHERE pp.user_id = CURRENT_USER_ID()
  AND pp.can_read = TRUE;

-- Fonction helper pour obtenir l'ID utilisateur
DELIMITER //

CREATE FUNCTION CURRENT_USER_ID() RETURNS INT
DETERMINISTIC
SQL SECURITY INVOKER
BEGIN
    RETURN (
        SELECT id FROM users
        WHERE username = SUBSTRING_INDEX(USER(), '@', 1)
    );
END//

DELIMITER ;
```

### Pattern 3 : Vues temporelles pour l'audit

Exposer uniquement les données dans une fenêtre temporelle :

```sql
-- Vue : données des 90 derniers jours uniquement
CREATE VIEW v_logs_recent AS
SELECT
    id,
    timestamp,
    user_id,
    action,
    table_name,
    record_id
FROM audit_logs
WHERE timestamp >= DATE_SUB(NOW(), INTERVAL 90 DAY)
WITH CHECK OPTION;

-- Empêche les utilisateurs de voir l'historique complet
GRANT SELECT ON database.v_logs_recent TO 'auditeur'@'%';

-- Seuls les admins ont accès à l'historique complet
GRANT SELECT ON database.audit_logs TO 'dba'@'localhost';
```

---

## Audit et traçabilité des accès via vues

### Logging des accès aux vues sensibles

```sql
-- Table de log des accès
CREATE TABLE vue_access_log (
    id INT AUTO_INCREMENT PRIMARY KEY,
    vue_name VARCHAR(100),
    user VARCHAR(100),
    access_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    query_type ENUM('SELECT', 'INSERT', 'UPDATE', 'DELETE'),
    INDEX idx_vue_time (vue_name, access_time)
) ENGINE=InnoDB;

-- Trigger sur la vue (via table sous-jacente)
DELIMITER //

CREATE TRIGGER trg_log_clients_access
AFTER INSERT ON clients
FOR EACH ROW
BEGIN
    INSERT INTO vue_access_log (vue_name, user, query_type)
    VALUES ('v_clients_sensibles', USER(), 'INSERT');
END//

CREATE TRIGGER trg_log_clients_update
AFTER UPDATE ON clients
FOR EACH ROW
BEGIN
    INSERT INTO vue_access_log (vue_name, user, query_type)
    VALUES ('v_clients_sensibles', USER(), 'UPDATE');
END//

DELIMITER ;

-- Vue pour consulter les accès (pour admins seulement)
CREATE VIEW v_audit_acces_vues AS
SELECT
    vue_name,
    user,
    query_type,
    COUNT(*) AS nb_acces,
    MAX(access_time) AS dernier_acces
FROM vue_access_log
WHERE access_time >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY vue_name, user, query_type
ORDER BY dernier_acces DESC;

GRANT SELECT ON database.v_audit_acces_vues TO 'security_admin'@'%';
```

### Détection d'accès suspects

```sql
-- Vue pour détecter les accès anormaux
CREATE VIEW v_acces_suspects AS
SELECT
    user,
    vue_name,
    COUNT(*) AS nb_acces,
    MIN(access_time) AS premier_acces,
    MAX(access_time) AS dernier_acces
FROM vue_access_log
WHERE access_time >= DATE_SUB(NOW(), INTERVAL 1 HOUR)
GROUP BY user, vue_name
HAVING COUNT(*) > 100  -- Plus de 100 accès en 1 heure = suspect
ORDER BY nb_acces DESC;

-- Alertes automatiques (via EVENT)
DELIMITER //

CREATE EVENT evt_check_acces_suspects
ON SCHEDULE EVERY 15 MINUTE
DO
BEGIN
    DECLARE v_count INT;

    SELECT COUNT(*) INTO v_count
    FROM v_acces_suspects;

    IF v_count > 0 THEN
        -- Insérer une alerte
        INSERT INTO alertes_securite (type, message, created_at)
        SELECT
            'ACCES_SUSPECT',
            CONCAT('Utilisateur ', user, ' a accédé ', nb_acces, ' fois à ', vue_name),
            NOW()
        FROM v_acces_suspects;
    END IF;
END//

DELIMITER ;
```

---

## Bonnes pratiques de sécurité avec les vues

### 1. Principe du moindre privilège

```sql
-- ❌ Mauvaise pratique : donner trop de privilèges
GRANT ALL PRIVILEGES ON database.* TO 'app_user'@'%';

-- ✅ Bonne pratique : privilèges granulaires via vues
-- Créer des vues spécifiques pour chaque besoin
CREATE VIEW v_app_readonly AS SELECT id, nom FROM table;
CREATE VIEW v_app_editable AS SELECT id, nom, description FROM table WHERE editable = TRUE;

GRANT SELECT ON database.v_app_readonly TO 'app_user_ro'@'%';
GRANT SELECT, UPDATE ON database.v_app_editable TO 'app_user_rw'@'%';
```

### 2. Toujours masquer les données critiques

```sql
-- Liste des colonnes à TOUJOURS masquer :
-- - Mots de passe (hachés ou non)
-- - Numéros de sécurité sociale
-- - Numéros de carte bancaire
-- - Informations de santé
-- - Données biométriques
-- - Clés d'API / tokens
-- - Salaires (sauf pour RH/Finance)

-- ✅ Exemple de vue sécurisée
CREATE VIEW v_users_app AS
SELECT
    id,
    username,
    email,
    -- PAS de password_hash, salt, api_key, etc.
    created_at,
    last_login
FROM users;
```

### 3. Utiliser WITH CHECK OPTION systématiquement pour les vues modifiables

```sql
-- ❌ Sans CHECK OPTION : risque de données incohérentes
CREATE VIEW v_produits_actifs AS
SELECT * FROM produits WHERE actif = TRUE;

-- ✅ Avec CHECK OPTION : garantit l'intégrité
CREATE VIEW v_produits_actifs AS
SELECT * FROM produits WHERE actif = TRUE
WITH CHECK OPTION;
```

### 4. Documenter les vues de sécurité

```sql
-- ✅ Documenter le rôle de sécurité de chaque vue
CREATE VIEW v_clients_support AS
-- Vue de sécurité pour l'équipe support
-- Masque : num_carte_bancaire, password_hash, num_secu
-- Filtre : uniquement clients actifs avec tickets ouverts
-- Mise à jour : Décembre 2025
-- Propriétaire : security_team
SELECT
    c.id,
    c.nom,
    c.email,
    c.telephone
FROM clients c
INNER JOIN support_tickets st ON c.id = st.client_id
WHERE c.statut = 'ACTIF'
  AND st.statut != 'CLOS';
```

### 5. Tester régulièrement les contrôles d'accès

```sql
-- Script de test des accès (à exécuter régulièrement)
-- Test 1 : Vérifier qu'un utilisateur standard ne peut pas accéder aux salaires
-- Se connecter en tant que 'app_user'
SELECT * FROM employes;  -- Devrait échouer (pas de privilèges)
SELECT * FROM v_annuaire_employes;  -- Devrait réussir (mais sans salaires)

-- Test 2 : Vérifier l'isolation multi-tenant
SET @current_tenant_id = 42;
SELECT COUNT(*) FROM v_mes_produits;  -- Devrait retourner uniquement produits du tenant 42

-- Test 3 : Vérifier WITH CHECK OPTION
INSERT INTO v_produits_actifs (nom, prix, actif)
VALUES ('Test', 10, FALSE);  -- Devrait échouer

-- Test 4 : Vérifier les privilèges en cascade
SELECT * FROM table_sensible;  -- Devrait échouer
SELECT * FROM v_securisee;     -- Devrait réussir via DEFINER
```

### 6. Monitorer et alerter sur les modifications de vues de sécurité

```sql
-- Table de log des modifications de schéma
CREATE TABLE schema_changes_log (
    id INT AUTO_INCREMENT PRIMARY KEY,
    object_type ENUM('TABLE', 'VIEW', 'PROCEDURE', 'FUNCTION'),
    object_name VARCHAR(100),
    change_type ENUM('CREATE', 'ALTER', 'DROP'),
    changed_by VARCHAR(100),
    change_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_time (change_time)
) ENGINE=InnoDB;

-- Trigger (via procédure appelée manuellement après changements)
DELIMITER //

CREATE PROCEDURE sp_log_schema_change(
    IN p_object_type VARCHAR(20),
    IN p_object_name VARCHAR(100),
    IN p_change_type VARCHAR(20)
)
BEGIN
    INSERT INTO schema_changes_log (object_type, object_name, change_type, changed_by)
    VALUES (p_object_type, p_object_name, p_change_type, USER());

    -- Si c'est une vue de sécurité critique, alerter immédiatement
    IF p_object_name IN ('v_users_app', 'v_clients_sensibles', 'v_employes_rh') THEN
        INSERT INTO alertes_securite (type, message, severity)
        VALUES (
            'SCHEMA_CHANGE_CRITICAL',
            CONCAT('Vue de sécurité ', p_object_name, ' a été modifiée par ', USER()),
            'CRITICAL'
        );
    END IF;
END//

DELIMITER ;

-- Utilisation après chaque modification
ALTER VIEW v_users_app AS ...;
CALL sp_log_schema_change('VIEW', 'v_users_app', 'ALTER');
```

### 7. Réviser périodiquement les privilèges

```sql
-- Audit annuel des privilèges sur les vues
SELECT
    table_schema,
    table_name,
    grantee,
    privilege_type,
    is_grantable
FROM information_schema.table_privileges
WHERE table_schema = 'production_db'
  AND table_name LIKE 'v_%'  -- Vues uniquement
ORDER BY table_name, grantee;

-- Identifier les privilèges suspects
SELECT
    grantee,
    COUNT(*) AS nb_privileges,
    GROUP_CONCAT(DISTINCT privilege_type) AS privileges
FROM information_schema.table_privileges
WHERE table_schema = 'production_db'
GROUP BY grantee
HAVING COUNT(*) > 50  -- Plus de 50 privilèges = suspect
ORDER BY nb_privileges DESC;
```

---

## ⚠️ Failles de sécurité courantes et prévention

### 1. Oublier de révoquer les privilèges directs sur les tables

```sql
-- ❌ Faille : Donner accès à la vue MAIS laisser les privilèges sur la table
GRANT SELECT ON database.v_clients_publics TO 'app_user'@'%';
-- L'utilisateur a AUSSI accès direct à la table !
SELECT * FROM clients;  -- ⚠️ Voit toutes les colonnes sensibles

-- ✅ Solution : Révoquer explicitement les privilèges sur la table
REVOKE ALL PRIVILEGES ON database.clients FROM 'app_user'@'%';
GRANT SELECT ON database.v_clients_publics TO 'app_user'@'%';
```

### 2. Utiliser des vues sans WITH CHECK OPTION pour les modifications

```sql
-- ❌ Faille : Vue modifiable sans CHECK OPTION
CREATE VIEW v_employes_actifs AS
SELECT * FROM employes WHERE statut = 'ACTIF';

GRANT UPDATE ON database.v_employes_actifs TO 'manager'@'%';

-- Manager peut rendre un employé inactif !
UPDATE v_employes_actifs SET statut = 'INACTIF' WHERE id = 10;
-- ✅ Réussit mais viole la logique métier

-- ✅ Solution : Toujours utiliser WITH CHECK OPTION
CREATE OR REPLACE VIEW v_employes_actifs AS
SELECT * FROM employes WHERE statut = 'ACTIF'
WITH CHECK OPTION;
```

### 3. Exposer des données via des jointures non filtrées

```sql
-- ❌ Faille : Jointure qui expose des données non autorisées
CREATE VIEW v_commandes_details AS
SELECT
    c.id,
    c.numero,
    cl.nom AS client_nom,
    cl.email AS client_email,
    cl.telephone AS client_telephone  -- ⚠️ Expose le téléphone
FROM commandes c
INNER JOIN clients cl ON c.client_id = cl.id;

-- ✅ Solution : Limiter les colonnes de la table jointe
CREATE OR REPLACE VIEW v_commandes_details AS
SELECT
    c.id,
    c.numero,
    cl.nom AS client_nom
    -- NE PAS exposer email, telephone
FROM commandes c
INNER JOIN clients cl ON c.client_id = cl.id;
```

### 4. Utiliser SQL SECURITY DEFINER sans précaution

```sql
-- ❌ Faille : DEFINER = root avec vue modifiable
CREATE DEFINER = 'root'@'localhost'
    SQL SECURITY DEFINER
    VIEW v_users AS
SELECT * FROM users;

GRANT UPDATE ON database.v_users TO 'app_user'@'%';

-- 'app_user' peut maintenant modifier TOUTES les données utilisateurs
-- avec les privilèges de root !

-- ✅ Solution 1 : Utiliser INVOKER
CREATE OR REPLACE SQL SECURITY INVOKER VIEW v_users AS
SELECT id, username, email FROM users;

-- ✅ Solution 2 : Vue en lecture seule avec DEFINER
CREATE OR REPLACE DEFINER = 'readonly_admin'@'localhost'
    SQL SECURITY DEFINER
    VIEW v_users_readonly AS
SELECT id, username, email FROM users;

GRANT SELECT ON database.v_users_readonly TO 'app_user'@'%';
-- Lecture seulement, pas d'UPDATE possible
```

### 5. Ne pas valider les variables de session

```sql
-- ❌ Faille : Utiliser @current_tenant_id sans validation
CREATE VIEW v_tenant_data AS
SELECT * FROM data WHERE tenant_id = @current_tenant_id;

-- Attaquant peut changer le tenant !
SET @current_tenant_id = 999;  -- Tenant non autorisé
SELECT * FROM v_tenant_data;   -- ⚠️ Accède aux données du tenant 999

-- ✅ Solution : Valider le tenant au niveau application
-- ET utiliser une table de permissions

CREATE VIEW v_tenant_data_secure AS
SELECT d.*
FROM data d
INNER JOIN user_tenant_permissions utp
    ON d.tenant_id = utp.tenant_id
WHERE utp.user_id = CURRENT_USER_ID()
  AND utp.tenant_id = @current_tenant_id;
-- Double vérification : permission ET variable correspondent
```

---

## ✅ Points clés à retenir

- Les **vues sont un outil puissant** pour implémenter la sécurité au niveau base de données
- **Masquage de colonnes** : Exclure les données sensibles des vues exposées aux applications
- **Filtrage de lignes** : Restreindre l'accès par utilisateur, rôle, tenant, niveau de sécurité
- **Combiner vues et privilèges** : Révoquer l'accès direct aux tables, accorder uniquement via vues
- **SQL SECURITY DEFINER** : Élévation contrôlée de privilèges (utile mais risqué si mal configuré)
- **SQL SECURITY INVOKER** : Chaque utilisateur utilise ses propres privilèges (plus sûr)
- **WITH CHECK OPTION** : Indispensable pour les vues modifiables (empêche les violations)
- **Architecture multi-niveaux** : Créer des vues en cascade selon les niveaux de sécurité
- **Audit et traçabilité** : Logger les accès aux vues sensibles, détecter les comportements suspects
- **Bonnes pratiques** : Moindre privilège, masquer les données critiques, tester régulièrement, documenter
- **Failles courantes** : Privilèges directs non révoqués, absence de CHECK OPTION, jointures non filtrées

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 GRANT Syntax](https://mariadb.com/kb/en/grant/) - Gestion des privilèges
- [📖 REVOKE Syntax](https://mariadb.com/kb/en/revoke/) - Révocation de privilèges
- [📖 View DEFINER and SQL SECURITY](https://mariadb.com/kb/en/create-view/#definer-and-sql-security-clause) - Sécurité des vues
- [📖 Information Schema](https://mariadb.com/kb/en/information-schema/) - Audit des privilèges

### Standards et réglementation
- **RGPD** : Réglementation sur la protection des données personnelles
- **PCI DSS** : Normes de sécurité pour les données de cartes bancaires
- **HIPAA** : Réglementation US sur les données de santé

### Articles complémentaires
- **"Database Security Best Practices"** - OWASP
- **"Row-Level Security Patterns"** - Database Security Guide
- **"Implementing Multi-Tenancy with Views"** - MariaDB Blog

---

## ➡️ Section suivante

**[9.6 Performance des vues : MERGE vs TEMPTABLE](./06-performance-vues.md)** : Analyse approfondie des algorithmes de traitement des vues, impact sur les performances, comment forcer un algorithme, quand utiliser quel algorithme, et stratégies d'optimisation pour des vues performantes.

---


⏭️ [Performance des vues : MERGE vs TEMPTABLE](/09-vues-et-donnees-virtuelles/06-performance-vues.md)
