🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.3 Application Time Period Tables 🆕

> **Niveau** : Avancé  
> **Durée estimée** : 1.5-2 heures  
> **Prérequis** : Section 18.2 (System-Versioned Tables), compréhension des contraintes d'intégrité

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Comprendre la différence entre **System-Time** (18.2) et **Application-Time** periods
- Créer des **périodes temporelles applicatives** avec `PERIOD FOR`
- Implémenter la contrainte **WITHOUT OVERLAPS** pour éviter les chevauchements
- Utiliser l'opérateur **OVERLAPS** pour requêtes temporelles
- Concevoir des systèmes de **réservation** (chambres, salles, véhicules)
- Gérer des **contrats** et **tarifications** avec dates de validité
- Optimiser les performances avec **index sur périodes**
- Combiner Application-Time et System-Time (tables bi-temporelles)

---

## Introduction

Les **Application Time Period Tables** sont une **nouveauté majeure de MariaDB 11.8** qui permet de gérer des **périodes temporelles définies par l'application** (par opposition aux timestamps système automatiques des System-Versioned Tables).

### System-Time vs Application-Time

**Comparaison fondamentale** :

| Aspect | System-Time (18.2) | Application-Time (18.3) |
|--------|-------------------|------------------------|
| **Qui gère les timestamps** | MariaDB (automatique) | Application (manuel) |
| **Objectif** | Audit, historique système | Logique métier, validité business |
| **Colonnes** | row_start, row_end (cachées) | Colonnes métier explicites |
| **Contraintes** | Aucune (historique seulement) | WITHOUT OVERLAPS possible |
| **Cas d'usage** | Conformité, forensique | Réservations, contrats, planning |
| **Requêtes** | FOR SYSTEM_TIME | Requêtes SQL normales + OVERLAPS |

**Exemple concret** :

```sql
-- System-Time (audit automatique)
CREATE TABLE employees (
  employee_id INT PRIMARY KEY,
  salary DECIMAL(10,2)
) WITH SYSTEM VERSIONING;
-- MariaDB track automatiquement QUAND les changements ont eu lieu

-- Application-Time (validité métier)
CREATE TABLE employee_contracts (
  employee_id INT,
  contract_start DATE,
  contract_end DATE,
  salary DECIMAL(10,2),
  PERIOD FOR contract_period (contract_start, contract_end)
);
-- L'application définit QUAND le contrat est valide (logique métier)
```

### Pourquoi Application Time Period Tables ?

**Problématiques résolues** :

1. **🏨 Systèmes de Réservation**
   - Chambres d'hôtel, salles de réunion, véhicules de location
   - Garantir qu'une ressource n'est pas doublement réservée sur une même période

2. **📄 Gestion de Contrats**
   - Contrats clients avec dates de validité
   - Tarifs et conditions par période
   - Interdire contrats qui se chevauchent pour un même client

3. **📅 Planification de Ressources**
   - Affectation d'employés à des projets
   - Planning de maintenance d'équipements
   - Occupation de postes de travail

4. **💰 Tarification Temporelle**
   - Prix par période (haute/basse saison)
   - Promotions avec dates de validité
   - Tarifs contractuels avec début et fin

**Avant Application Time Period Tables** :
```sql
-- Logique applicative complexe pour éviter chevauchements
SELECT COUNT(*) FROM bookings
WHERE room_number = 101
  AND (
    (new_start >= booking_start AND new_start < booking_end)
    OR (new_end > booking_start AND new_end <= booking_end)
    OR (new_start <= booking_start AND new_end >= booking_end)
  );
-- Si COUNT > 0 → chevauchement, refuser la réservation
```

**Avec Application Time Period Tables** :
```sql
-- Contrainte d'intégrité au niveau base de données
UNIQUE (room_number, booking_period WITHOUT OVERLAPS)
-- MariaDB refuse automatiquement les chevauchements !
```

💡 **Avantage clé** : La logique métier de non-chevauchement est **garantie par le SGBD**, pas par l'application.

---

## Syntaxe et Concepts

### Définition d'une Période avec PERIOD FOR

```sql
CREATE TABLE table_name (
  -- Colonnes métier
  id INT PRIMARY KEY,
  resource_id INT,
  
  -- Colonnes définissant la période
  start_column DATE,      -- ou DATETIME, TIMESTAMP
  end_column DATE,        -- ou DATETIME, TIMESTAMP
  
  -- Déclaration de la période
  PERIOD FOR period_name (start_column, end_column)
);
```

**Composants** :
- **period_name** : Nom logique de la période (ex: `booking_period`, `validity_period`)
- **start_column** : Colonne de début (DATE, DATETIME, TIMESTAMP)
- **end_column** : Colonne de fin (même type que start)

**Conventions** :
- Période **semi-ouverte** : `[start, end)` (start inclus, end exclus)
- `end_column` doit être **> start_column** (validé par contrainte CHECK implicite)

### Contrainte WITHOUT OVERLAPS

La contrainte **WITHOUT OVERLAPS** garantit qu'aucune période ne se chevauche pour une même ressource.

```sql
CREATE TABLE reservations (
  reservation_id INT PRIMARY KEY AUTO_INCREMENT,
  room_number INT,
  guest_name VARCHAR(100),
  check_in DATE,
  check_out DATE,
  
  PERIOD FOR stay_period (check_in, check_out),
  
  -- Contrainte : pas de chevauchement pour une même chambre
  UNIQUE (room_number, stay_period WITHOUT OVERLAPS)
);
```

**Sémantique** :
- Deux réservations de la **même chambre** ne peuvent se chevaucher
- Deux réservations de **chambres différentes** peuvent se chevaucher librement

**Exemple de comportement** :
```sql
-- Réservation 1 : chambre 101 du 01 au 05 juin
INSERT INTO reservations (room_number, guest_name, check_in, check_out)
VALUES (101, 'Alice', '2025-06-01', '2025-06-05');
-- ✅ OK

-- Réservation 2 : chambre 101 du 03 au 07 juin (CHEVAUCHEMENT !)
INSERT INTO reservations (room_number, guest_name, check_in, check_out)
VALUES (101, 'Bob', '2025-06-03', '2025-06-07');
-- ❌ ERROR: Duplicate entry for key 'room_number'

-- Réservation 3 : chambre 101 du 05 au 10 juin (pas de chevauchement)
INSERT INTO reservations (room_number, guest_name, check_in, check_out)
VALUES (101, 'Bob', '2025-06-05', '2025-06-10');
-- ✅ OK (check_in = ancien check_out, période semi-ouverte [))

-- Réservation 4 : chambre 102 du 01 au 10 juin
INSERT INTO reservations (room_number, guest_name, check_in, check_out)
VALUES (102, 'Charlie', '2025-06-01', '2025-06-10');
-- ✅ OK (chambre différente)
```

### Types de Colonnes Supportées

```sql
-- DATE (le plus courant pour réservations)
CREATE TABLE hotel_bookings (
  room_id INT,
  check_in DATE,
  check_out DATE,
  PERIOD FOR stay (check_in, check_out),
  UNIQUE (room_id, stay WITHOUT OVERLAPS)
);

-- DATETIME (pour granularité horaire)
CREATE TABLE meeting_rooms (
  room_id INT,
  meeting_start DATETIME,
  meeting_end DATETIME,
  PERIOD FOR meeting_time (meeting_start, meeting_end),
  UNIQUE (room_id, meeting_time WITHOUT OVERLAPS)
);

-- TIMESTAMP (avec timezone awareness)
CREATE TABLE server_allocations (
  server_id INT,
  allocation_start TIMESTAMP,
  allocation_end TIMESTAMP,
  PERIOD FOR allocation_period (allocation_start, allocation_end),
  UNIQUE (server_id, allocation_period WITHOUT OVERLAPS)
);

-- INTEGER (pour versioning, ranges numériques)
CREATE TABLE ip_allocations (
  subnet_id INT,
  ip_start INT,
  ip_end INT,
  PERIOD FOR ip_range (ip_start, ip_end),
  UNIQUE (subnet_id, ip_range WITHOUT OVERLAPS)
);
```

---

## Cas d'Usage Détaillés

### 1. Système de Réservation d'Hôtel

**Besoin** : Gérer réservations de chambres avec garantie de non-chevauchement.

```sql
-- Schéma complet
CREATE TABLE hotels (
  hotel_id INT PRIMARY KEY AUTO_INCREMENT,
  hotel_name VARCHAR(100),
  city VARCHAR(50)
);

CREATE TABLE hotel_rooms (
  room_id INT PRIMARY KEY AUTO_INCREMENT,
  hotel_id INT,
  room_number VARCHAR(10),
  room_type ENUM('SINGLE','DOUBLE','SUITE'),
  daily_rate DECIMAL(10,2),
  FOREIGN KEY (hotel_id) REFERENCES hotels(hotel_id),
  UNIQUE (hotel_id, room_number)
);

CREATE TABLE room_reservations (
  reservation_id INT PRIMARY KEY AUTO_INCREMENT,
  room_id INT,
  guest_name VARCHAR(100),
  guest_email VARCHAR(255),
  check_in DATE,
  check_out DATE,
  total_amount DECIMAL(10,2),
  status ENUM('PENDING','CONFIRMED','CANCELLED','COMPLETED'),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  -- Période de séjour
  PERIOD FOR stay_period (check_in, check_out),
  
  -- Contrainte : pas de double réservation
  UNIQUE (room_id, stay_period WITHOUT OVERLAPS),
  
  -- Contrainte : check_out après check_in (automatique avec PERIOD)
  FOREIGN KEY (room_id) REFERENCES hotel_rooms(room_id),
  
  INDEX idx_guest (guest_email),
  INDEX idx_dates (check_in, check_out)
);

-- Insertion réussie
INSERT INTO room_reservations 
  (room_id, guest_name, guest_email, check_in, check_out, total_amount, status)
VALUES 
  (1, 'Alice Martin', 'alice@example.com', '2025-07-01', '2025-07-05', 400.00, 'CONFIRMED');

-- Tentative de double réservation (ÉCHEC)
INSERT INTO room_reservations 
  (room_id, guest_name, guest_email, check_in, check_out, total_amount, status)
VALUES 
  (1, 'Bob Smith', 'bob@example.com', '2025-07-03', '2025-07-08', 500.00, 'CONFIRMED');
-- ERROR 1062: Duplicate entry '1' for key 'room_id'

-- Réservation consécutive (SUCCÈS)
INSERT INTO room_reservations 
  (room_id, guest_name, guest_email, check_in, check_out, total_amount, status)
VALUES 
  (1, 'Bob Smith', 'bob@example.com', '2025-07-05', '2025-07-10', 500.00, 'CONFIRMED');

-- Rechercher chambres disponibles pour période donnée
-- (nécessite requête inverse - voir section suivante)
```

**Procédure de réservation** :
```sql
DELIMITER $$
CREATE PROCEDURE book_room(
  IN p_room_id INT,
  IN p_guest_name VARCHAR(100),
  IN p_guest_email VARCHAR(255),
  IN p_check_in DATE,
  IN p_check_out DATE,
  OUT p_reservation_id INT,
  OUT p_error_message VARCHAR(255)
)
BEGIN
  DECLARE v_rate DECIMAL(10,2);
  DECLARE v_nights INT;
  DECLARE v_total DECIMAL(10,2);
  DECLARE CONTINUE HANDLER FOR 1062
  BEGIN
    SET p_reservation_id = NULL;
    SET p_error_message = 'Room not available for selected dates';
  END;
  
  -- Validation
  IF p_check_out <= p_check_in THEN
    SET p_reservation_id = NULL;
    SET p_error_message = 'Check-out must be after check-in';
    LEAVE proc_label;
  END IF;
  
  -- Récupérer tarif journalier
  SELECT daily_rate INTO v_rate
  FROM hotel_rooms
  WHERE room_id = p_room_id;
  
  IF v_rate IS NULL THEN
    SET p_reservation_id = NULL;
    SET p_error_message = 'Room not found';
    LEAVE proc_label;
  END IF;
  
  -- Calculer montant total
  SET v_nights = DATEDIFF(p_check_out, p_check_in);
  SET v_total = v_rate * v_nights;
  
  -- Tenter insertion
  INSERT INTO room_reservations 
    (room_id, guest_name, guest_email, check_in, check_out, total_amount, status)
  VALUES 
    (p_room_id, p_guest_name, p_guest_email, p_check_in, p_check_out, v_total, 'CONFIRMED');
  
  SET p_reservation_id = LAST_INSERT_ID();
  SET p_error_message = NULL;
END$$
DELIMITER ;

-- Utilisation
CALL book_room(1, 'Charlie Brown', 'charlie@example.com', '2025-08-01', '2025-08-05', @res_id, @error);
SELECT @res_id, @error;
```

### 2. Gestion de Contrats Clients

**Besoin** : Contrats avec dates de validité, un seul contrat actif par client à la fois.

```sql
CREATE TABLE customers (
  customer_id INT PRIMARY KEY AUTO_INCREMENT,
  company_name VARCHAR(100),
  contact_email VARCHAR(255)
);

CREATE TABLE customer_contracts (
  contract_id INT PRIMARY KEY AUTO_INCREMENT,
  customer_id INT,
  contract_number VARCHAR(50) UNIQUE,
  contract_type ENUM('ANNUAL','MONTHLY','PROJECT'),
  
  -- Période de validité du contrat
  valid_from DATE,
  valid_until DATE,
  
  -- Conditions contractuelles
  discount_rate DECIMAL(5,2),
  payment_terms VARCHAR(100),
  
  PERIOD FOR validity_period (valid_from, valid_until),
  
  -- Contrainte : un seul contrat actif par client sur une période donnée
  UNIQUE (customer_id, validity_period WITHOUT OVERLAPS),
  
  FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- Contrat 1 : Client 1, du 01/01/2025 au 31/12/2025
INSERT INTO customer_contracts 
  (customer_id, contract_number, contract_type, valid_from, valid_until, discount_rate)
VALUES 
  (1, 'C2025-001', 'ANNUAL', '2025-01-01', '2025-12-31', 10.00);

-- Tentative de contrat qui chevauche (ÉCHEC)
INSERT INTO customer_contracts 
  (customer_id, contract_number, contract_type, valid_from, valid_until, discount_rate)
VALUES 
  (1, 'C2025-002', 'MONTHLY', '2025-06-01', '2025-07-01', 5.00);
-- ERROR: Duplicate entry

-- Renouvellement consécutif (SUCCÈS)
INSERT INTO customer_contracts 
  (customer_id, contract_number, contract_type, valid_from, valid_until, discount_rate)
VALUES 
  (1, 'C2026-001', 'ANNUAL', '2025-12-31', '2026-12-31', 12.00);

-- Requête : Contrat actif pour un client à une date donnée
SELECT 
  contract_id,
  contract_number,
  contract_type,
  discount_rate,
  valid_from,
  valid_until
FROM customer_contracts
WHERE customer_id = 1
  AND '2025-06-15' >= valid_from
  AND '2025-06-15' < valid_until;
```

### 3. Planning d'Affectation d'Employés

**Besoin** : Affecter employés à des projets, éviter conflits de planning.

```sql
CREATE TABLE employees (
  employee_id INT PRIMARY KEY,
  employee_name VARCHAR(100),
  department VARCHAR(50)
);

CREATE TABLE projects (
  project_id INT PRIMARY KEY AUTO_INCREMENT,
  project_name VARCHAR(100),
  project_status ENUM('PLANNED','ACTIVE','COMPLETED','CANCELLED')
);

CREATE TABLE project_assignments (
  assignment_id INT PRIMARY KEY AUTO_INCREMENT,
  employee_id INT,
  project_id INT,
  role VARCHAR(50),
  
  -- Période d'affectation
  assignment_start DATE,
  assignment_end DATE,
  
  -- Pourcentage d'allocation (0-100)
  allocation_percent INT CHECK (allocation_percent BETWEEN 0 AND 100),
  
  PERIOD FOR assignment_period (assignment_start, assignment_end),
  
  -- Contrainte : Un employé ne peut être affecté à 100% sur périodes qui se chevauchent
  -- Note : Cette contrainte empêche tout chevauchement
  -- Pour gérer allocation partielle, voir variante ci-dessous
  UNIQUE (employee_id, assignment_period WITHOUT OVERLAPS),
  
  FOREIGN KEY (employee_id) REFERENCES employees(employee_id),
  FOREIGN KEY (project_id) REFERENCES projects(project_id)
);

-- Affectation 1 : Alice sur Projet A, 01/06 au 30/06, 100%
INSERT INTO project_assignments 
  (employee_id, project_id, role, assignment_start, assignment_end, allocation_percent)
VALUES 
  (1, 1, 'Developer', '2025-06-01', '2025-06-30', 100);

-- Affectation 2 : Alice sur Projet B, 15/06 au 15/07, 50% (ÉCHEC)
INSERT INTO project_assignments 
  (employee_id, project_id, role, assignment_start, assignment_end, allocation_percent)
VALUES 
  (1, 2, 'Analyst', '2025-06-15', '2025-07-15', 50);
-- ERROR: Chevauchement interdit par la contrainte

-- Affectation consécutive (SUCCÈS)
INSERT INTO project_assignments 
  (employee_id, project_id, role, assignment_start, assignment_end, allocation_percent)
VALUES 
  (1, 2, 'Analyst', '2025-06-30', '2025-07-31', 100);
```

**Variante : Autoriser allocations partielles** :
```sql
-- Pour gérer plusieurs projets en parallèle (allocation < 100% chacun),
-- il faudrait une logique applicative ou un trigger
-- car WITHOUT OVERLAPS ne supporte pas conditions complexes

-- Solution : Trigger de validation
DELIMITER $$
CREATE TRIGGER validate_allocation
BEFORE INSERT ON project_assignments
FOR EACH ROW
BEGIN
  DECLARE total_allocation INT;
  
  -- Calculer allocation totale de l'employé sur période qui chevauche
  SELECT COALESCE(SUM(allocation_percent), 0)
  INTO total_allocation
  FROM project_assignments
  WHERE employee_id = NEW.employee_id
    AND assignment_start < NEW.assignment_end
    AND assignment_end > NEW.assignment_start;
  
  -- Vérifier que total + nouvelle allocation <= 100%
  IF total_allocation + NEW.allocation_percent > 100 THEN
    SIGNAL SQLSTATE '45000'
    SET MESSAGE_TEXT = 'Total allocation exceeds 100%';
  END IF;
END$$
DELIMITER ;
```

### 4. Tarification Temporelle

**Besoin** : Tarifs variables selon la période (saisons, promotions).

```sql
CREATE TABLE products (
  product_id INT PRIMARY KEY,
  product_name VARCHAR(100),
  base_price DECIMAL(10,2)
);

CREATE TABLE seasonal_pricing (
  pricing_id INT PRIMARY KEY AUTO_INCREMENT,
  product_id INT,
  season_name VARCHAR(50),
  
  -- Période tarifaire
  price_from DATE,
  price_until DATE,
  
  price DECIMAL(10,2),
  
  PERIOD FOR pricing_period (price_from, price_until),
  
  -- Contrainte : un seul prix par produit à la fois
  UNIQUE (product_id, pricing_period WITHOUT OVERLAPS),
  
  FOREIGN KEY (product_id) REFERENCES products(product_id)
);

-- Prix haute saison : 01/07 au 31/08
INSERT INTO seasonal_pricing 
  (product_id, season_name, price_from, price_until, price)
VALUES 
  (1, 'High Season', '2025-07-01', '2025-08-31', 150.00);

-- Prix basse saison : 01/01 au 30/06
INSERT INTO seasonal_pricing 
  (product_id, season_name, price_from, price_until, price)
VALUES 
  (1, 'Low Season', '2025-01-01', '2025-07-01', 100.00);

-- Prix basse saison suite : 31/08 au 31/12
INSERT INTO seasonal_pricing 
  (product_id, season_name, price_from, price_until, price)
VALUES 
  (1, 'Low Season 2', '2025-08-31', '2025-12-31', 100.00);

-- Fonction pour obtenir le prix à une date
DELIMITER $$
CREATE FUNCTION get_price_at_date(
  p_product_id INT,
  p_date DATE
)
RETURNS DECIMAL(10,2)
READS SQL DATA
BEGIN
  DECLARE v_price DECIMAL(10,2);
  
  SELECT price INTO v_price
  FROM seasonal_pricing
  WHERE product_id = p_product_id
    AND p_date >= price_from
    AND p_date < price_until;
  
  -- Si pas de tarif spécifique, retourner prix de base
  IF v_price IS NULL THEN
    SELECT base_price INTO v_price
    FROM products
    WHERE product_id = p_product_id;
  END IF;
  
  RETURN v_price;
END$$
DELIMITER ;

-- Utilisation
SELECT get_price_at_date(1, '2025-07-15');  -- Retourne 150.00 (haute saison)
SELECT get_price_at_date(1, '2025-03-15');  -- Retourne 100.00 (basse saison)
```

---

## Opérateur OVERLAPS

L'opérateur **OVERLAPS** permet de vérifier si deux périodes se chevauchent.

### Syntaxe

```sql
-- Vérifier si deux périodes se chevauchent
(start1, end1) OVERLAPS (start2, end2)

-- Vérifier si période chevauche une période définie par PERIOD FOR
period_column OVERLAPS PERIOD(start_value, end_value)
```

### Exemples d'Utilisation

```sql
-- Table avec périodes
CREATE TABLE events (
  event_id INT PRIMARY KEY,
  event_name VARCHAR(100),
  start_date DATE,
  end_date DATE,
  PERIOD FOR event_period (start_date, end_date)
);

INSERT INTO events VALUES
  (1, 'Conference A', '2025-06-01', '2025-06-05'),
  (2, 'Conference B', '2025-06-10', '2025-06-15'),
  (3, 'Conference C', '2025-06-03', '2025-06-12');

-- Requête 1 : Événements qui se chevauchent avec une période donnée
SELECT * FROM events
WHERE event_period OVERLAPS PERIOD('2025-06-04', '2025-06-08');
-- Retourne : Conference A et Conference C

-- Requête 2 : Paires d'événements qui se chevauchent
SELECT 
  e1.event_name AS event1,
  e2.event_name AS event2
FROM events e1
CROSS JOIN events e2
WHERE e1.event_id < e2.event_id
  AND e1.event_period OVERLAPS e2.event_period;
-- Retourne : (Conference A, Conference C)

-- Requête 3 : Disponibilité (périodes NON chevauchantes)
-- Chambres disponibles pour check_in='2025-07-01', check_out='2025-07-05'
SELECT r.room_id, r.room_number
FROM hotel_rooms r
WHERE NOT EXISTS (
  SELECT 1 FROM room_reservations res
  WHERE res.room_id = r.room_id
    AND res.stay_period OVERLAPS PERIOD('2025-07-01', '2025-07-05')
);
```

### Sémantique de OVERLAPS

```sql
-- Deux périodes se chevauchent si :
(start1, end1) OVERLAPS (start2, end2)
-- Équivalent logique :
start1 < end2 AND end1 > start2

-- Exemples :
SELECT ('2025-06-01', '2025-06-05') OVERLAPS ('2025-06-03', '2025-06-08');  -- TRUE
SELECT ('2025-06-01', '2025-06-05') OVERLAPS ('2025-06-05', '2025-06-10');  -- FALSE (edge case)
SELECT ('2025-06-01', '2025-06-10') OVERLAPS ('2025-06-03', '2025-06-08');  -- TRUE (contenu)
SELECT ('2025-06-01', '2025-06-05') OVERLAPS ('2025-06-10', '2025-06-15');  -- FALSE (disjoint)
```

💡 **Convention [)** : Les périodes sont **semi-ouvertes** - le début est inclus, la fin est exclue.

---

## Index et Performance

### Index sur Périodes

```sql
-- Index sur les colonnes start et end
CREATE INDEX idx_period ON reservations(check_in, check_out);

-- Pour WITHOUT OVERLAPS, MariaDB crée automatiquement index approprié
CREATE TABLE bookings (
  room_id INT,
  check_in DATE,
  check_out DATE,
  PERIOD FOR stay (check_in, check_out),
  UNIQUE (room_id, stay WITHOUT OVERLAPS)
);
-- Index automatique sur (room_id, check_in, check_out)

-- Index supplémentaire pour requêtes de recherche
CREATE INDEX idx_dates ON bookings(check_in, check_out);
```

### Optimisation de Requêtes

```sql
-- Requête optimisée : Chambres disponibles
EXPLAIN SELECT r.room_id, r.room_number
FROM hotel_rooms r
WHERE NOT EXISTS (
  SELECT 1 FROM room_reservations res
  WHERE res.room_id = r.room_id
    AND res.stay_period OVERLAPS PERIOD('2025-07-01', '2025-07-05')
);

-- Alternative avec LEFT JOIN (parfois plus performante)
SELECT r.room_id, r.room_number
FROM hotel_rooms r
LEFT JOIN room_reservations res 
  ON res.room_id = r.room_id
  AND res.stay_period OVERLAPS PERIOD('2025-07-01', '2025-07-05')
WHERE res.reservation_id IS NULL;
```

### Performance de WITHOUT OVERLAPS

**Mécanisme interne** :
- MariaDB utilise un **index B-Tree** sur (resource_id, start, end)
- Lors d'INSERT/UPDATE, vérifie tous les enregistrements potentiellement chevauchants
- Complexité : O(log n) pour recherche, O(k) pour validation (k = nombre de chevauchements possibles)

**Impact sur performance** :

| Opération | Sans WITHOUT OVERLAPS | Avec WITHOUT OVERLAPS | Overhead |
|-----------|----------------------|----------------------|----------|
| INSERT | Baseline | +10-30% | Validation chevauchement |
| UPDATE | Baseline | +15-40% | Double validation (old + new) |
| DELETE | Baseline | Baseline | Aucun |
| SELECT | Baseline | Baseline | Aucun |

💡 **Best practice** : Les SELECT normaux ne sont pas affectés. L'overhead est uniquement sur INSERT/UPDATE.

---

## Combinaison Application-Time + System-Time

Les tables **bi-temporelles** combinent les deux types de périodes.

### Table Bi-Temporelle

```sql
-- Table avec les deux types de versioning
CREATE TABLE employee_salaries (
  employee_id INT,
  
  -- Application-Time : Période de validité contractuelle
  valid_from DATE,
  valid_until DATE,
  
  salary DECIMAL(10,2),
  department VARCHAR(50),
  
  PERIOD FOR validity (valid_from, valid_until),
  UNIQUE (employee_id, validity WITHOUT OVERLAPS)
  
) WITH SYSTEM VERSIONING;  -- System-Time : Audit automatique

-- Résultat : 4 colonnes temporelles
-- valid_from, valid_until (Application-Time, explicites)
-- row_start, row_end (System-Time, cachées)
```

**Cas d'usage** :
- **Application-Time** : Quand le salaire/contrat est valide (métier)
- **System-Time** : Quand la ligne a été modifiée dans le système (audit)

**Exemple** :
```sql
-- 1er janvier 2025 : Créer contrat pour 2025
INSERT INTO employee_salaries 
  (employee_id, valid_from, valid_until, salary, department)
VALUES 
  (1, '2025-01-01', '2025-12-31', 50000, 'IT');

-- État actuel :
-- Application-Time : valide du 01/01/2025 au 31/12/2025
-- System-Time : créé le 01/01/2025 à 10h00

-- 15 juin 2025 : Correction erreur, augmenter salaire
UPDATE employee_salaries
SET salary = 55000
WHERE employee_id = 1
  AND valid_from = '2025-01-01';

-- État actuel :
-- Application-Time : TOUJOURS valide du 01/01/2025 au 31/12/2025 (inchangé)
-- System-Time : modifié le 15/06/2025 à 14h30

-- Table courante :
-- valid_from='2025-01-01', valid_until='2025-12-31', salary=55000
-- row_start='2025-06-15 14:30:00', row_end='2106-02-07...'

-- Table historique (System-Time) :
-- valid_from='2025-01-01', valid_until='2025-12-31', salary=50000
-- row_start='2025-01-01 10:00:00', row_end='2025-06-15 14:30:00'
```

**Requêtes bi-temporelles** :
```sql
-- Quel était le salaire contractuel pour 2025, tel que connu le 01/02/2025 ?
SELECT salary
FROM employee_salaries
FOR SYSTEM_TIME AS OF '2025-02-01 12:00:00'
WHERE employee_id = 1
  AND '2025-06-15' >= valid_from
  AND '2025-06-15' < valid_until;
-- Retourne : 50000 (avant correction)

-- Quel est le salaire contractuel actuel pour juin 2025 ?
SELECT salary
FROM employee_salaries
WHERE employee_id = 1
  AND '2025-06-15' >= valid_from
  AND '2025-06-15' < valid_until;
-- Retourne : 55000 (après correction)
```

---

## Best Practices

### 1. Nommage Cohérent

```sql
-- ✅ Bon : Noms explicites
PERIOD FOR contract_validity (valid_from, valid_until)
PERIOD FOR booking_period (check_in, check_out)
PERIOD FOR assignment_duration (start_date, end_date)

-- ❌ Mauvais : Noms génériques
PERIOD FOR p (c1, c2)
PERIOD FOR period (start, end)
```

### 2. Convention Semi-Ouverte [)

```sql
-- ✅ Adopter convention start inclus, end exclus
-- Réservation du 01 au 05 = 4 nuits (01, 02, 03, 04)
INSERT INTO reservations (check_in, check_out)
VALUES ('2025-06-01', '2025-06-05');

-- Réservation consécutive du 05 au 10 = 5 nuits (05, 06, 07, 08, 09)
INSERT INTO reservations (check_in, check_out)
VALUES ('2025-06-05', '2025-06-10');

-- Pas de gap, pas de chevauchement !
```

### 3. Validation Côté Application

```sql
-- ✅ Vérifier end > start avant INSERT
IF p_check_out <= p_check_in THEN
  SIGNAL SQLSTATE '45000' 
  SET MESSAGE_TEXT = 'Check-out must be after check-in';
END IF;

-- MariaDB le vérifie aussi, mais mieux fail fast
```

### 4. Gestion des Annulations

```sql
-- Option 1 : Soft delete (recommandé)
ALTER TABLE reservations ADD COLUMN status ENUM('ACTIVE','CANCELLED');
UPDATE reservations SET status = 'CANCELLED' WHERE reservation_id = 123;

-- Option 2 : Hard delete (libère la contrainte)
DELETE FROM reservations WHERE reservation_id = 123;

-- Option 1 permet historique, option 2 permet nouvelle réservation immédiate
```

### 5. Index Stratégiques

```sql
-- Index sur resource + period pour recherches
CREATE INDEX idx_resource_period ON bookings(room_id, check_in, check_out);

-- Index sur dates seules pour range queries
CREATE INDEX idx_dates ON bookings(check_in, check_out);

-- Index sur status si soft delete
CREATE INDEX idx_active ON bookings(room_id, status, check_in, check_out);
```

### 6. Documentation des Contraintes

```sql
-- Commenter les tables et contraintes
CREATE TABLE room_reservations (
  /* ... colonnes ... */
  PERIOD FOR stay_period (check_in, check_out),
  
  -- Contrainte métier : Pas de double réservation
  -- La période est semi-ouverte : [check_in, check_out)
  -- Une réservation se terminant le 05/06 permet une nouvelle le 05/06
  UNIQUE (room_id, stay_period WITHOUT OVERLAPS)
) COMMENT='Réservations de chambres avec période semi-ouverte [start, end)';
```

---

## Migration et Adoption

### Ajouter Application Time à Table Existante

```sql
-- Table existante sans périodes
CREATE TABLE old_bookings (
  booking_id INT PRIMARY KEY,
  room_id INT,
  guest_name VARCHAR(100),
  check_in DATE,
  check_out DATE
);

-- Données existantes
INSERT INTO old_bookings VALUES
  (1, 101, 'Alice', '2025-06-01', '2025-06-05'),
  (2, 101, 'Bob', '2025-06-10', '2025-06-15');

-- Migration : Ajouter PERIOD et contrainte
ALTER TABLE old_bookings
  ADD PERIOD FOR stay_period (check_in, check_out);

-- ⚠️ Attention : Vérifier d'abord qu'il n'y a PAS de chevauchements existants !
SELECT b1.booking_id, b2.booking_id
FROM old_bookings b1
CROSS JOIN old_bookings b2
WHERE b1.booking_id < b2.booking_id
  AND b1.room_id = b2.room_id
  AND b1.check_in < b2.check_out
  AND b1.check_out > b2.check_in;
-- Si résultats → nettoyer données avant d'ajouter contrainte

-- Ajouter contrainte (si pas de chevauchements)
ALTER TABLE old_bookings
  ADD UNIQUE (room_id, stay_period WITHOUT OVERLAPS);
```

### Stratégie de Migration Progressive

```sql
-- Étape 1 : Nouvelle table avec contraintes
CREATE TABLE new_bookings (
  /* ... avec PERIOD FOR et WITHOUT OVERLAPS ... */
);

-- Étape 2 : Migrer données propres
INSERT INTO new_bookings
SELECT * FROM old_bookings
WHERE booking_id NOT IN (
  -- Exclure chevauchements détectés
  SELECT DISTINCT b1.booking_id
  FROM old_bookings b1
  CROSS JOIN old_bookings b2
  WHERE b1.booking_id < b2.booking_id
    AND b1.room_id = b2.room_id
    AND (b1.check_in, b1.check_out) OVERLAPS (b2.check_in, b2.check_out)
);

-- Étape 3 : Traiter manuellement cas problématiques
SELECT * FROM old_bookings
WHERE booking_id NOT IN (SELECT booking_id FROM new_bookings);

-- Étape 4 : Basculer application vers new_bookings
-- Étape 5 : Renommer tables
RENAME TABLE old_bookings TO old_bookings_archive;
RENAME TABLE new_bookings TO bookings;
```

---

## ✅ Points clés à retenir

### Concepts Fondamentaux
- ✅ **Application Time Period** : Périodes métier définies par l'application (vs System Time automatique)
- ✅ **PERIOD FOR** : Déclare une période sur deux colonnes (start, end)
- ✅ **WITHOUT OVERLAPS** : Contrainte d'intégrité empêchant chevauchements
- ✅ **Convention [)** : Périodes semi-ouvertes (start inclus, end exclus)
- 🆕 **MariaDB 11.8** : Fonctionnalité nouvelle conforme SQL:2011

### Syntaxe
- ✅ `PERIOD FOR period_name (start_col, end_col)`
- ✅ `UNIQUE (resource_id, period_name WITHOUT OVERLAPS)`
- ✅ `period_col OVERLAPS PERIOD(start, end)` pour requêtes

### Cas d'Usage Principaux
- ✅ **Réservations** : Chambres, salles, véhicules, équipements
- ✅ **Contrats** : Clients, fournisseurs avec dates de validité
- ✅ **Planning** : Affectation employés, allocation ressources
- ✅ **Tarification** : Prix par saison, promotions temporelles

### Performance
- ✅ **Overhead écritures** : 10-40% sur INSERT/UPDATE (validation chevauchement)
- ✅ **SELECT** : Aucun impact (0%)
- ✅ **Index automatique** : (resource_id, start, end) créé par WITHOUT OVERLAPS
- ✅ **Complexité** : O(log n) pour insertion avec validation

### Best Practices
- ✅ Convention semi-ouverte [) pour périodes consécutives sans gap/chevauchement
- ✅ Validation end > start côté application (fail fast)
- ✅ Soft delete avec status pour conserver historique
- ✅ Index sur (resource_id, start, end) et (start, end)
- ✅ Vérifier absence de chevauchements AVANT migration

### Différences System-Time vs Application-Time

| Aspect | System-Time | Application-Time |
|--------|-------------|------------------|
| Gestion | Automatique (MariaDB) | Manuelle (application) |
| Objectif | Audit système | Logique métier |
| Contraintes | Aucune | WITHOUT OVERLAPS possible |
| Requêtes | FOR SYSTEM_TIME | SQL normal + OVERLAPS |
| Combinaison | ✅ Bi-temporel possible | ✅ Bi-temporel possible |

---

## 🔗 Ressources et références

### Documentation Officielle MariaDB 11.8
- 📖 [Application-Time Period Tables](https://mariadb.com/kb/en/application-time-period-tables/) 🆕 - Guide complet
- 📖 [PERIOD FOR Clause](https://mariadb.com/kb/en/period-for/) - Syntaxe
- 📖 [WITHOUT OVERLAPS Constraint](https://mariadb.com/kb/en/without-overlaps/) - Contrainte
- 📖 [OVERLAPS Operator](https://mariadb.com/kb/en/overlaps/) - Opérateur de comparaison

### Standards SQL
- 📚 [SQL:2011 Temporal Support](https://en.wikipedia.org/wiki/SQL:2011#Temporal_support) - Standard
- 📚 [Bitemporal Data](https://en.wikipedia.org/wiki/Bitemporal_data) - Concept bi-temporel

### Articles et Cas d'Usage
- 📝 [Implementing Reservation Systems with Application Time](https://mariadb.com/resources/blog/reservation-systems-application-time/) 🆕
- 📝 [Contract Management with Temporal Periods](https://mariadb.com/resources/blog/contract-management-periods/)
- 📝 [Bitemporal Tables in MariaDB](https://mariadb.com/kb/en/bitemporal-tables/)

### Comparaison avec Autres SGBD
- 🔄 [SQL Server Temporal Tables](https://docs.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables) - Application-time supporté
- 🔄 [PostgreSQL Range Types](https://www.postgresql.org/docs/current/rangetypes.html) - Approche alternative

---

## ➡️ Section suivante

**[18.4 Colonnes Virtuelles et Générées (VIRTUAL vs STORED)](./04-virtual-generated-columns.md)** : Découvrez comment créer des colonnes calculées automatiquement, leur différence fondamentale, et comment les indexer pour optimiser vos requêtes.

---


⏭️ [Colonnes virtuelles et générées](/18-fonctionnalites-avancees/04-virtual-generated-columns.md)
