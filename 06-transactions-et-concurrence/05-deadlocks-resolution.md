🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.5 Deadlocks : Détection et Résolution

> **Niveau** : Avancé
> **Durée estimée** : 3-4 heures

> **Prérequis** :
> - Section 6.4 (Verrous)
> - Compréhension des transactions
> - Notions de graphes (wait-for graph)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre ce qu'est un deadlock et comment il se forme
- Analyser un deadlock avec SHOW ENGINE INNODB STATUS
- Implémenter des stratégies de prévention
- Créer une logique de retry robuste
- Configurer InnoDB pour la détection optimale
- Monitorer et alerter sur les deadlocks en production
- Diagnostiquer les causes racines et les corriger

---

## Introduction

Un **deadlock** (interblocage) est une situation où deux ou plusieurs transactions s'attendent mutuellement pour obtenir des verrous, créant un cycle d'attente sans issue. C'est un problème inévitable dans tout système concurrent, mais qu'on peut largement minimiser et gérer correctement.

### Analogie du Monde Réel

Imaginez deux voitures à une intersection, chacune voulant tourner mais bloquant le passage de l'autre :

```
     ↑
     A (attend B)
     |
←----+----→
     |
     B (attend A)
     ↓

💥 Deadlock : Aucune ne peut avancer
```

En base de données, c'est exactement pareil avec les verrous.

### Impact en Production

```
Symptômes d'un deadlock :
- ERROR 1213 (40001): Deadlock found when trying to get lock
- Transactions qui échouent aléatoirement
- Dégradation soudaine des performances
- Frustration des utilisateurs (retry manuel nécessaire)
```

---

## 1. Anatomie d'un Deadlock

### 1.1 Le Deadlock Classique

**Scénario le plus courant** : Deux transactions verrouillent des ressources dans l'ordre inverse.

```sql
┌────────────────────────────────────────────────────────────────┐
│ Timeline : Deadlock Classique                                  │
├────────────────────────────────────────────────────────────────┤
│ Transaction A                     Transaction B                │
├────────────────────────────────────────────────────────────────┤
│ START TRANSACTION;                                             │
│                                                                │
│ UPDATE comptes                                                 │
│ SET solde = solde - 100                                        │
│ WHERE id = 5;                                                  │
│ -- 🔒 Lock acquis sur ligne 5                                  │
│                                  START TRANSACTION;            │
│                                                                │
│                                  UPDATE comptes                │
│                                  SET solde = solde - 50        │
│                                  WHERE id = 3;                 │
│                                  -- 🔒 Lock acquis sur ligne 3 │
│                                                                │
│ UPDATE comptes                                                 │
│ SET solde = solde + 100                                        │
│ WHERE id = 3;                                                  │
│ -- ⏳ Attend lock sur ligne 3    |                             │
│    (détenu par B)                |                             │
│                                  |                             │
│                                  UPDATE comptes                │
│                                  SET solde = solde + 50        │
│                                  WHERE id = 5;                 │
│                                  -- ⏳ Attend lock sur ligne 5 │
│                                     (détenu par A)             │
│                                                                │
│ 💥 DEADLOCK DÉTECTÉ !                                          │
│                                                                │
│ InnoDB choisit une victime → B                                 │
│                                                                │
│ COMMIT; (A réussit)              ROLLBACK automatique          │
│                                  ERROR 1213 retourné           │
└────────────────────────────────────────────────────────────────┘
```

### 1.2 Représentation en Graphe

Un deadlock se représente comme un **cycle dans un graphe d'attente** (wait-for graph) :

```
Transaction A → attend → Transaction B
      ↑                        ↓
      ← ← ← ← attend ← ← ← ← ←

Cycle détecté = Deadlock
```

**Conditions nécessaires pour un deadlock** (toutes doivent être vraies) :

1. **Mutual Exclusion** : Les verrous ne peuvent être partagés
2. **Hold and Wait** : Une transaction garde ses verrous en attendant d'autres
3. **No Preemption** : Les verrous ne peuvent être forcés/volés
4. **Circular Wait** : Cycle dans le graphe d'attente

### 1.3 Types de Deadlocks

#### Type 1 : Lock-Order Deadlock

**Le plus fréquent** : Ordre inversé d'acquisition des verrous.

```sql
-- Transaction A : Lock(5) puis Lock(3)
-- Transaction B : Lock(3) puis Lock(5)
-- 💥 Deadlock
```

#### Type 2 : Write Skew Deadlock

```sql
-- Transaction A
START TRANSACTION;
SELECT @sum := SUM(solde) FROM comptes WHERE client_id = 100 FOR UPDATE;

-- Transaction B (parallèle)
START TRANSACTION;
SELECT @sum := SUM(solde) FROM comptes WHERE client_id = 100 FOR UPDATE;

-- Les deux peuvent lire, mais au moment de l'UPDATE...
-- 💥 Potentiel deadlock selon l'ordre des lignes
```

#### Type 3 : Gap Lock Deadlock

```sql
-- Transaction A
SELECT * FROM produits WHERE prix BETWEEN 10 AND 50 FOR UPDATE;
-- 🔒 Gap locks posés

-- Transaction B
INSERT INTO produits (nom, prix) VALUES ('Widget', 30);
-- ⏳ Attend gap lock de A

-- Transaction A
INSERT INTO produits (nom, prix) VALUES ('Gadget', 25);
-- ⏳ Attend que B libère... mais B attend A !
-- 💥 Deadlock
```

---

## 2. Détection Automatique par InnoDB

### 2.1 Algorithme de Détection

InnoDB détecte les deadlocks en maintenant un **graphe d'attente** et en cherchant des cycles.

```
Détection périodique :
┌─────────────────────────────────────────┐
│ 1. Transaction demande un verrou        │
│ 2. Si verrou indisponible :             │
│    - Ajouter arête au graphe            │
│    - Chercher cycle (DFS/BFS)           │
│ 3. Si cycle trouvé :                    │
│    - DEADLOCK détecté                   │
│    - Choisir une victime                │
│    - Rollback de la victime             │
│ 4. Retourner ERROR 1213 à la victime    │
└─────────────────────────────────────────┘
```

### 2.2 Choix de la Victime

InnoDB choisit la transaction à rollback selon un **algorithme de coût** :

```sql
-- Critères (par ordre de priorité) :
-- 1. Nombre de lignes modifiées (InnoDB préfère rollback la plus petite)
-- 2. Nombre de verrous détenus
-- 3. Poids de la transaction (trx_weight)

-- Exemple :
-- Transaction A : 1000 lignes modifiées
-- Transaction B : 10 lignes modifiées
-- → B sera la victime (coût de rollback plus faible)
```

### 2.3 Configuration de la Détection

```sql
-- Vérifier la détection de deadlock
SHOW VARIABLES LIKE 'innodb_deadlock_detect';
-- ON par défaut (recommandé)

-- Désactiver (NON recommandé, sauf cas très spécifique)
SET GLOBAL innodb_deadlock_detect = OFF;
-- ⚠️ Les transactions attendront jusqu'au timeout au lieu de résoudre le deadlock
```

**Pourquoi garder la détection activée ?**

```sql
-- AVEC détection (ON) :
-- Deadlock détecté en ~10ms
-- Transaction victime rollback immédiatement
-- Autre transaction continue
-- Performance ✅

-- SANS détection (OFF) :
-- Les deux transactions attendent le timeout (50s par défaut)
-- Blocage de 50 secondes !
-- Performance ❌
```

---

## 3. Analyse d'un Deadlock

### 3.1 SHOW ENGINE INNODB STATUS

La commande principale pour diagnostiquer un deadlock :

```sql
SHOW ENGINE INNODB STATUS\G
```

**Section importante** : `LATEST DETECTED DEADLOCK`

```
------------------------
LATEST DETECTED DEADLOCK
------------------------
2025-12-12 14:30:45 0x7f8a1c0f4700
*** (1) TRANSACTION:
TRANSACTION 421523821992080, ACTIVE 0 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 10, OS thread handle 140234567890, query id 100 localhost root updating
UPDATE comptes SET solde = solde + 100 WHERE id = 3

*** (1) HOLDS THE LOCK(S):
RECORD LOCKS space id 58 page no 3 n bits 72 index PRIMARY of table `banque`.`comptes`
trx id 421523821992080 lock_mode X locks rec but not gap
Record lock, heap no 6 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 4; hex 80000005; asc     ;;
 1: len 6; hex 000061f8e3a0; asc   a   ;;
 2: len 7; hex 820000012c0110; asc     ,  ;;
 3: len 7; hex 416c696365; asc Alice;;
 4: len 4; hex 80000064; asc    d;;

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 58 page no 3 n bits 72 index PRIMARY of table `banque`.`comptes`
trx id 421523821992080 lock_mode X locks rec but not gap waiting
Record lock, heap no 4 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 4; hex 80000003; asc     ;;

*** (2) TRANSACTION:
TRANSACTION 421523821992081, ACTIVE 0 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 11, OS thread handle 140234567891, query id 101 localhost root updating
UPDATE comptes SET solde = solde + 50 WHERE id = 5

*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 58 page no 3 n bits 72 index PRIMARY of table `banque`.`comptes`
trx id 421523821992081 lock_mode X locks rec but not gap
Record lock, heap no 4 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 4; hex 80000003; asc     ;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 58 page no 3 n bits 72 index PRIMARY of table `banque`.`comptes`
trx id 421523821992081 lock_mode X locks rec but not gap waiting
Record lock, heap no 6 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 4; hex 80000005; asc     ;;

*** WE ROLL BACK TRANSACTION (2)
```

### 3.2 Décrypter le Output

**Éléments clés à identifier** :

1. **Les deux transactions impliquées**
   - Transaction (1) : Thread 10
   - Transaction (2) : Thread 11

2. **Ce que chacune détient**
   - (1) HOLDS THE LOCK : ligne id=5
   - (2) HOLDS THE LOCK : ligne id=3

3. **Ce que chacune attend**
   - (1) WAITING FOR : ligne id=3 (détenue par 2)
   - (2) WAITING FOR : ligne id=5 (détenue par 1)

4. **Le cycle** :
   ```
   (1) détient 5, attend 3
   (2) détient 3, attend 5
   → Cycle → Deadlock
   ```

5. **La victime choisie**
   - `WE ROLL BACK TRANSACTION (2)`

### 3.3 Extraire les Informations Utiles

```sql
-- Parser le dernier deadlock automatiquement
SELECT
    SUBSTRING_INDEX(
        SUBSTRING_INDEX(variable_value, '*** (1) TRANSACTION:', -1),
        '*** WE ROLL BACK',
        1
    ) AS deadlock_info
FROM information_schema.GLOBAL_STATUS
WHERE variable_name = 'Innodb_deadlock_details';

-- Ou utiliser un script externe pour parser le SHOW ENGINE INNODB STATUS
```

### 3.4 Logging des Deadlocks

```sql
-- Activer le logging de TOUS les deadlocks (pas juste le dernier)
SET GLOBAL innodb_print_all_deadlocks = ON;

-- Les deadlocks seront loggés dans le error log
-- Exemple dans /var/log/mysql/error.log :
```

```
2025-12-12 14:30:45 140234567890 [Note] InnoDB: Transactions deadlock detected, dumping detailed information.
2025-12-12 14:30:45 140234567890 [Note] InnoDB:
*** (1) TRANSACTION:
TRANSACTION 421523821992080, ACTIVE 0 sec starting index read
...
```

**Configuration production recommandée** :

```ini
[mysqld]
# Logger tous les deadlocks
innodb_print_all_deadlocks = 1

# Rotation des logs pour éviter la saturation
log_error = /var/log/mysql/error.log
max_binlog_size = 100M
```

---

## 4. Prévention des Deadlocks

### 4.1 Stratégie #1 : Ordre Cohérent des Accès

**Principe** : Toujours acquérir les verrous dans le même ordre.

```sql
-- ❌ MAUVAIS : Ordre aléatoire
PROCEDURE transfer_v1(from_id, to_id, amount) {
    START TRANSACTION;
    UPDATE comptes SET solde = solde - amount WHERE id = from_id;
    UPDATE comptes SET solde = solde + amount WHERE id = to_id;
    COMMIT;
}

-- Transaction A : transfer(5, 3, 100) → Lock 5, puis 3
-- Transaction B : transfer(3, 5, 50)  → Lock 3, puis 5
-- 💥 Deadlock possible

-- ✅ BON : Ordre déterministe (par ID croissant)
PROCEDURE transfer_v2(from_id, to_id, amount) {
    START TRANSACTION;

    -- Verrouiller dans l'ordre des IDs
    IF from_id < to_id THEN
        SELECT * FROM comptes WHERE id = from_id FOR UPDATE;
        SELECT * FROM comptes WHERE id = to_id FOR UPDATE;
    ELSE
        SELECT * FROM comptes WHERE id = to_id FOR UPDATE;
        SELECT * FROM comptes WHERE id = from_id FOR UPDATE;
    END IF;

    -- Maintenant faire les modifications
    UPDATE comptes SET solde = solde - amount WHERE id = from_id;
    UPDATE comptes SET solde = solde + amount WHERE id = to_id;

    COMMIT;
}

-- Transaction A : transfer(5, 3, 100) → Lock 3, puis 5
-- Transaction B : transfer(3, 5, 50)  → Lock 3, puis 5
-- ✅ Pas de deadlock : même ordre
```

**Implémentation générique** :

```sql
DELIMITER //

CREATE PROCEDURE lock_in_order(IN ids VARCHAR(255))
BEGIN
    DECLARE id INT;
    DECLARE done BOOLEAN DEFAULT FALSE;
    DECLARE ids_cursor CURSOR FOR
        SELECT CAST(value AS UNSIGNED) AS id
        FROM (
            SELECT SUBSTRING_INDEX(SUBSTRING_INDEX(ids, ',', numbers.n), ',', -1) AS value
            FROM (SELECT 1 n UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5) numbers
            WHERE CHAR_LENGTH(ids) - CHAR_LENGTH(REPLACE(ids, ',', '')) >= numbers.n - 1
        ) AS split
        ORDER BY CAST(value AS UNSIGNED);

    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;

    OPEN ids_cursor;

    read_loop: LOOP
        FETCH ids_cursor INTO id;
        IF done THEN
            LEAVE read_loop;
        END IF;

        -- Verrouiller dans l'ordre
        SELECT * FROM comptes WHERE id = id FOR UPDATE;
    END LOOP;

    CLOSE ids_cursor;
END//

DELIMITER ;

-- Utilisation
START TRANSACTION;
CALL lock_in_order('5,3,8,1');  -- Verrouille 1, 3, 5, 8 (ordre croissant)
-- Modifications...
COMMIT;
```

### 4.2 Stratégie #2 : Transactions Courtes

**Principe** : Moins de temps avec verrous = moins de chances de deadlock.

```sql
-- ❌ MAUVAIS : Transaction longue
START TRANSACTION;

SELECT * FROM produits WHERE id = 42 FOR UPDATE;
-- 🔒 Verrou posé

-- Appel API externe (500ms)
CALL external_api();

-- Calculs complexes (2s)
SET @nouveau_prix = complex_calculation();

-- Validation métier (1s)
IF validate(@nouveau_prix) THEN
    UPDATE produits SET prix = @nouveau_prix WHERE id = 42;
END IF;

COMMIT;  -- Verrou gardé pendant 3.5s

-- ✅ BON : Transaction minimale
-- 1. Calculs AVANT la transaction
CALL external_api();
SET @nouveau_prix = complex_calculation();
SET @is_valid = validate(@nouveau_prix);

-- 2. Transaction ultra-courte
START TRANSACTION;
IF @is_valid THEN
    UPDATE produits SET prix = @nouveau_prix WHERE id = 42;
END IF;
COMMIT;  -- Verrou gardé <10ms
```

**Règle d'or** : Les transactions doivent être **atomiques mais pas lentes**.

```
Transaction = Unité logique, pas unité de temps
```

### 4.3 Stratégie #3 : Granularité Fine

**Principe** : Verrouiller seulement ce qui est nécessaire.

```sql
-- ❌ MAUVAIS : Verrous trop larges
START TRANSACTION;
SELECT * FROM produits FOR UPDATE;  -- 🔒 Toute la table !
-- Modifie 1 ligne
UPDATE produits SET stock = stock - 1 WHERE id = 42;
COMMIT;

-- ✅ BON : Verrou ciblé
START TRANSACTION;
SELECT * FROM produits WHERE id = 42 FOR UPDATE;  -- 🔒 1 ligne
UPDATE produits SET stock = stock - 1 WHERE id = 42;
COMMIT;

-- ✅ MEILLEUR : Pas de SELECT si pas nécessaire
START TRANSACTION;
UPDATE produits SET stock = stock - 1 WHERE id = 42 AND stock > 0;
-- Verrou implicite sur la ligne
IF ROW_COUNT() = 0 THEN
    ROLLBACK;
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Stock insuffisant';
ELSE
    COMMIT;
END IF;
```

### 4.4 Stratégie #4 : Réduire la Contention

**Principe** : Moins de transactions concurrentes sur les mêmes ressources.

```sql
-- Pattern : Partitionnement des données chaudes

-- ❌ MAUVAIS : Une seule table de compteurs
CREATE TABLE compteur_global (
    id INT PRIMARY KEY DEFAULT 1,
    valeur BIGINT
);

-- Toutes les transactions se battent pour la même ligne
UPDATE compteur_global SET valeur = valeur + 1;  -- 🔥 Hot spot

-- ✅ BON : Compteurs partitionnés
CREATE TABLE compteurs (
    partition_id INT,
    valeur BIGINT,
    PRIMARY KEY (partition_id)
);

-- Distribuer les updates sur plusieurs lignes
SET @partition = FLOOR(RAND() * 10);  -- 0-9
UPDATE compteurs SET valeur = valeur + 1 WHERE partition_id = @partition;

-- Lecture du total
SELECT SUM(valeur) FROM compteurs;
```

### 4.5 Stratégie #5 : Isolation Level Adapté

```sql
-- SERIALIZABLE → Plus de verrous → Plus de deadlocks
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
START TRANSACTION;
SELECT * FROM produits WHERE categorie = 'Livres';
-- 🔒 Pose des verrous de lecture automatiques
-- ⚠️ Risque de deadlock élevé

-- READ COMMITTED → Moins de verrous → Moins de deadlocks
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
SELECT * FROM produits WHERE categorie = 'Livres';
-- ✅ Pas de verrous de lecture
-- ✅ Moins de risque de deadlock
```

💡 **Compromis** : SERIALIZABLE = plus sûr mais plus de deadlocks. Choisir le niveau d'isolation minimum nécessaire.

---

## 5. Résolution : Retry Logic

### 5.1 Pattern de Base

```python
import pymysql
import time

def execute_with_retry(func, max_retries=3, backoff=0.1):
    """
    Retry automatique en cas de deadlock

    Args:
        func: Fonction à exécuter (doit inclure la transaction)
        max_retries: Nombre maximum de tentatives
        backoff: Délai initial entre tentatives (secondes)

    Returns:
        Résultat de func() si succès

    Raises:
        Exception si échec après max_retries
    """
    for attempt in range(max_retries):
        try:
            return func()
        except pymysql.err.OperationalError as e:
            if e.args[0] == 1213:  # Deadlock error code
                if attempt < max_retries - 1:
                    # Backoff exponentiel
                    wait_time = backoff * (2 ** attempt)
                    print(f"Deadlock détecté, retry {attempt + 1}/{max_retries} après {wait_time}s")
                    time.sleep(wait_time)
                    continue
                else:
                    # Max retries atteint
                    print(f"Échec après {max_retries} tentatives")
                    raise
            else:
                # Autre erreur, ne pas retry
                raise

    raise Exception("Retry logic failed")

# Utilisation
def transfer_money(from_account, to_account, amount):
    conn = pymysql.connect(...)
    try:
        with conn.cursor() as cursor:
            cursor.execute("START TRANSACTION")

            # Ordre déterministe
            accounts = sorted([from_account, to_account])
            for acc in accounts:
                cursor.execute("SELECT id FROM comptes WHERE id = %s FOR UPDATE", (acc,))

            # Transfert
            cursor.execute(
                "UPDATE comptes SET solde = solde - %s WHERE id = %s",
                (amount, from_account)
            )
            cursor.execute(
                "UPDATE comptes SET solde = solde + %s WHERE id = %s",
                (amount, to_account)
            )

            cursor.execute("COMMIT")
            return True
    except Exception:
        cursor.execute("ROLLBACK")
        raise
    finally:
        conn.close()

# Appel avec retry automatique
try:
    execute_with_retry(lambda: transfer_money(100, 200, 50.00))
    print("Transfert réussi")
except Exception as e:
    print(f"Transfert échoué : {e}")
```

### 5.2 Backoff Exponentiel

**Pourquoi ?** Éviter que tous les clients retry en même temps.

```python
# ❌ MAUVAIS : Backoff fixe
for attempt in range(3):
    try:
        transfer()
        break
    except DeadlockError:
        time.sleep(0.1)  # Toujours 100ms
        # Si 100 clients retry simultanément, deadlock persiste

# ✅ BON : Backoff exponentiel + jitter
import random

for attempt in range(3):
    try:
        transfer()
        break
    except DeadlockError:
        # Backoff : 100ms, 200ms, 400ms (+ random jitter)
        wait_time = 0.1 * (2 ** attempt) + random.uniform(0, 0.05)
        time.sleep(wait_time)
        # Les clients retry à des moments différents
```

### 5.3 Circuit Breaker Pattern

**Pour éviter l'avalanche de retries** :

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.last_failure_time = None
        self.state = 'CLOSED'  # CLOSED, OPEN, HALF_OPEN

    def call(self, func):
        if self.state == 'OPEN':
            if time.time() - self.last_failure_time > self.timeout:
                self.state = 'HALF_OPEN'
            else:
                raise Exception("Circuit breaker is OPEN")

        try:
            result = func()
            self.on_success()
            return result
        except Exception as e:
            self.on_failure()
            raise

    def on_success(self):
        self.failure_count = 0
        self.state = 'CLOSED'

    def on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()

        if self.failure_count >= self.failure_threshold:
            self.state = 'OPEN'
            print(f"Circuit breaker OPEN after {self.failure_count} failures")

# Utilisation
breaker = CircuitBreaker(failure_threshold=5, timeout=60)

try:
    breaker.call(lambda: transfer_money(100, 200, 50))
except Exception as e:
    print(f"Opération échouée : {e}")
```

---

## 6. Monitoring et Alerting

### 6.1 Métriques Clés

```sql
-- 1. Nombre total de deadlocks depuis le démarrage
SHOW GLOBAL STATUS LIKE 'Innodb_deadlocks';
-- Exemple : 142

-- 2. Deadlocks par heure (via monitoring externe)
-- Calculer : (Innodb_deadlocks_now - Innodb_deadlocks_1h_ago)

-- 3. Ratio deadlock/transaction
SELECT
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS
     WHERE VARIABLE_NAME = 'Innodb_deadlocks')
    /
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS
     WHERE VARIABLE_NAME = 'Com_commit') * 100 AS deadlock_ratio_pct;
```

### 6.2 Table de Log Personnalisée

```sql
-- Créer une table pour logger les deadlocks
CREATE TABLE deadlock_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    detected_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    victim_thread_id INT,
    victim_query TEXT,
    blocking_thread_id INT,
    blocking_query TEXT,
    deadlock_details TEXT,
    INDEX idx_detected_at (detected_at)
) ENGINE=InnoDB;

-- Trigger ou procédure pour logger (via monitoring externe)
-- Exemple avec un script Python qui parse le error log
```

### 6.3 Script de Monitoring

```python
#!/usr/bin/env python3
import pymysql
import re
import time
from datetime import datetime, timedelta

def monitor_deadlocks(interval=60):
    """
    Monitore les deadlocks et alerte si seuil dépassé

    Args:
        interval: Intervalle de vérification en secondes
    """
    conn = pymysql.connect(
        host='localhost',
        user='monitor',
        password='xxx',
        database='mysql'
    )

    last_deadlock_count = 0

    while True:
        with conn.cursor() as cursor:
            # Compter les deadlocks
            cursor.execute(
                "SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS "
                "WHERE VARIABLE_NAME = 'Innodb_deadlocks'"
            )
            current_deadlock_count = int(cursor.fetchone()[0])

            # Calcul des deadlocks dans l'intervalle
            deadlocks_in_interval = current_deadlock_count - last_deadlock_count
            deadlocks_per_hour = (deadlocks_in_interval / interval) * 3600

            print(f"{datetime.now()}: {deadlocks_in_interval} deadlocks dans les {interval}s dernières secondes")
            print(f"  → ~{deadlocks_per_hour:.1f} deadlocks/heure")

            # Alerter si seuil dépassé
            if deadlocks_per_hour > 100:  # Seuil : 100 deadlocks/heure
                send_alert(f"ALERTE: {deadlocks_per_hour:.1f} deadlocks/heure détectés !")

            last_deadlock_count = current_deadlock_count

        time.sleep(interval)

def send_alert(message):
    """Envoyer une alerte (email, Slack, PagerDuty, etc.)"""
    print(f"🚨 {message}")
    # Implémenter l'envoi réel (SMTP, webhook, etc.)

if __name__ == '__main__':
    monitor_deadlocks(interval=60)
```

### 6.4 Dashboard Grafana

**Requêtes Prometheus/InfluxDB** :

```promql
# Taux de deadlocks par minute
rate(mysql_global_status_innodb_deadlocks[1m])

# Alerte si > 1 deadlock/min pendant 5 minutes
rate(mysql_global_status_innodb_deadlocks[5m]) > 1
```

**Configuration alerting** :

```yaml
# prometheus/alerts.yml
groups:
  - name: mariadb_deadlocks
    rules:
      - alert: HighDeadlockRate
        expr: rate(mysql_global_status_innodb_deadlocks[5m]) > 0.017  # ~1/min
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Taux élevé de deadlocks sur {{ $labels.instance }}"
          description: "{{ $value }} deadlocks/sec détectés"

      - alert: CriticalDeadlockRate
        expr: rate(mysql_global_status_innodb_deadlocks[5m]) > 0.083  # ~5/min
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Taux CRITIQUE de deadlocks sur {{ $labels.instance }}"
```

---

## 7. Diagnostic Avancé

### 7.1 Identifier les Patterns

**Analyse historique des deadlocks** :

```sql
-- Parser le error log (via script externe)
-- Identifier les requêtes fréquemment impliquées

-- Exemple de résultat :
┌────────────────────────────────────────┬───────────┐
│ Query Pattern                          │ Count     │
├────────────────────────────────────────┼───────────┤
│ UPDATE comptes SET solde = ...         │ 45        │
│ UPDATE commandes SET statut = ...      │ 23        │
│ INSERT INTO logs ...                   │ 12        │
└────────────────────────────────────────┴───────────┘

-- Pattern : Les UPDATE sur comptes sont la cause principale
```

**Script d'analyse** :

```bash
#!/bin/bash
# analyze_deadlocks.sh

# Extraire tous les deadlocks du error log
grep -A 50 "LATEST DETECTED DEADLOCK" /var/log/mysql/error.log > /tmp/deadlocks.txt

# Compter les occurrences de chaque type de requête
echo "Top 10 des requêtes impliquées dans les deadlocks :"
grep "MySQL thread id" /tmp/deadlocks.txt | \
    grep -o "UPDATE\|INSERT\|DELETE\|SELECT" | \
    sort | uniq -c | sort -rn | head -10
```

### 7.2 Reproduire un Deadlock

**Pour comprendre et corriger, il faut reproduire** :

```sql
-- Script de test : Générer un deadlock intentionnellement

-- Session 1
START TRANSACTION;
UPDATE test_deadlock SET val = val + 1 WHERE id = 1;
SELECT SLEEP(2);  -- Attendre que session 2 démarre
UPDATE test_deadlock SET val = val + 1 WHERE id = 2;
COMMIT;

-- Session 2 (démarrer 1 seconde après session 1)
START TRANSACTION;
UPDATE test_deadlock SET val = val + 1 WHERE id = 2;
UPDATE test_deadlock SET val = val + 1 WHERE id = 1;  -- 💥 Deadlock
COMMIT;
```

**Test automatisé** :

```python
import threading
import pymysql
import time

def session1():
    conn = pymysql.connect(...)
    cursor = conn.cursor()
    try:
        cursor.execute("START TRANSACTION")
        cursor.execute("UPDATE test_deadlock SET val = val + 1 WHERE id = 1")
        time.sleep(0.5)  # Laisser session2 démarrer
        cursor.execute("UPDATE test_deadlock SET val = val + 1 WHERE id = 2")
        cursor.execute("COMMIT")
        print("Session 1: SUCCESS")
    except pymysql.err.OperationalError as e:
        if e.args[0] == 1213:
            print("Session 1: DEADLOCK (victime)")
        cursor.execute("ROLLBACK")
    finally:
        conn.close()

def session2():
    time.sleep(0.1)  # Démarrer légèrement après session1
    conn = pymysql.connect(...)
    cursor = conn.cursor()
    try:
        cursor.execute("START TRANSACTION")
        cursor.execute("UPDATE test_deadlock SET val = val + 1 WHERE id = 2")
        time.sleep(0.5)
        cursor.execute("UPDATE test_deadlock SET val = val + 1 WHERE id = 1")
        cursor.execute("COMMIT")
        print("Session 2: SUCCESS")
    except pymysql.err.OperationalError as e:
        if e.args[0] == 1213:
            print("Session 2: DEADLOCK (victime)")
        cursor.execute("ROLLBACK")
    finally:
        conn.close()

# Exécuter les deux sessions en parallèle
t1 = threading.Thread(target=session1)
t2 = threading.Thread(target=session2)

t1.start()
t2.start()

t1.join()
t2.join()

print("Test terminé - Un deadlock devrait avoir été détecté")
```

### 7.3 Analyse avec pt-deadlock-logger

**Outil Percona Toolkit** :

```bash
# Installation
apt-get install percona-toolkit

# Logger les deadlocks en continu
pt-deadlock-logger \
    --user=root \
    --password=xxx \
    --database=mysql \
    --dest D=deadlocks,t=deadlock_log \
    --run-time=86400  # 24 heures

# Analyse des résultats
mysql -e "
SELECT
    DATE_FORMAT(ts, '%Y-%m-%d %H:00:00') AS hour,
    COUNT(*) AS deadlock_count
FROM deadlocks.deadlock_log
GROUP BY DATE_FORMAT(ts, '%Y-%m-%d %H:00:00')
ORDER BY hour DESC
LIMIT 24;
"
```

---

## 8. Cas d'Usage et Solutions

### 8.1 Cas #1 : E-commerce - Commandes Concurrentes

**Problème** : Deux clients commandent le dernier article.

```sql
-- ❌ Version avec deadlock potentiel
-- Client A
START TRANSACTION;
SELECT @stock := stock FROM produits WHERE id = 42 FOR UPDATE;
UPDATE commandes SET ... WHERE ...;
UPDATE produits SET stock = @stock - 1 WHERE id = 42;
COMMIT;

-- Client B (parallèle)
START TRANSACTION;
SELECT @stock := stock FROM produits WHERE id = 42 FOR UPDATE;
UPDATE commandes SET ... WHERE ...;  -- Accède à une autre table
UPDATE produits SET stock = @stock - 1 WHERE id = 42;
COMMIT;

-- 💥 Deadlock possible si l'ordre des verrous diffère
```

**✅ Solution : Verrouillage immédiat et ordre cohérent**

```sql
CREATE PROCEDURE passer_commande(
    IN p_produit_id INT,
    IN p_client_id INT,
    IN p_quantite INT,
    OUT p_success BOOLEAN
)
BEGIN
    DECLARE v_stock INT;

    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SET p_success = FALSE;
    END;

    START TRANSACTION;

    -- 1. Verrouiller immédiatement le produit
    SELECT stock INTO v_stock
    FROM produits
    WHERE id = p_produit_id
    FOR UPDATE;

    -- 2. Vérifier le stock
    IF v_stock < p_quantite THEN
        ROLLBACK;
        SET p_success = FALSE;
    ELSE
        -- 3. Créer la commande
        INSERT INTO commandes (client_id, produit_id, quantite)
        VALUES (p_client_id, p_produit_id, p_quantite);

        -- 4. Décrémenter le stock
        UPDATE produits
        SET stock = stock - p_quantite
        WHERE id = p_produit_id;

        COMMIT;
        SET p_success = TRUE;
    END IF;
END;
```

### 8.2 Cas #2 : Banking - Transferts Multiples

**Problème** : Deadlocks sur les comptes lors de transferts simultanés.

```sql
-- ❌ Version problématique
-- Transaction A : 5 → 3
-- Transaction B : 3 → 5
-- 💥 Deadlock

-- ✅ Solution : Ordre déterministe + batch lock
CREATE PROCEDURE transfer_securise(
    IN p_de_compte INT,
    IN p_vers_compte INT,
    IN p_montant DECIMAL(10,2)
)
BEGIN
    DECLARE v_solde DECIMAL(10,2);
    DECLARE v_min_id INT;
    DECLARE v_max_id INT;

    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Transfert échoué';
    END;

    -- Ordre déterministe
    SET v_min_id = LEAST(p_de_compte, p_vers_compte);
    SET v_max_id = GREATEST(p_de_compte, p_vers_compte);

    START TRANSACTION;

    -- Verrouiller dans l'ordre
    SELECT solde INTO v_solde FROM comptes WHERE id = v_min_id FOR UPDATE;
    SELECT 1 FROM comptes WHERE id = v_max_id FOR UPDATE;

    -- Vérifier le solde du compte source
    SELECT solde INTO v_solde FROM comptes WHERE id = p_de_compte;
    IF v_solde < p_montant THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Solde insuffisant';
    END IF;

    -- Effectuer le transfert
    UPDATE comptes SET solde = solde - p_montant WHERE id = p_de_compte;
    UPDATE comptes SET solde = solde + p_montant WHERE id = p_vers_compte;

    -- Audit log
    INSERT INTO transactions_log (de, vers, montant)
    VALUES (p_de_compte, p_vers_compte, p_montant);

    COMMIT;
END;
```

### 8.3 Cas #3 : Inventory - Mise à Jour de Stocks

**Problème** : Gap lock deadlock lors d'INSERT concurrents.

```sql
-- ❌ Problématique : Gap locks qui causent deadlocks
-- Table avec index sur categorie_id
CREATE TABLE produits (
    id INT PRIMARY KEY AUTO_INCREMENT,
    categorie_id INT,
    nom VARCHAR(100),
    INDEX idx_categorie (categorie_id)
);

-- Transaction A
START TRANSACTION;
SELECT * FROM produits WHERE categorie_id = 5 FOR UPDATE;
-- 🔒 Gap locks posés

-- Transaction B
INSERT INTO produits (categorie_id, nom) VALUES (5, 'Widget');
-- ⏳ Bloqué par gap lock de A

-- Transaction A
INSERT INTO produits (categorie_id, nom) VALUES (5, 'Gadget');
-- 💥 Deadlock

-- ✅ Solution 1 : READ COMMITTED (pas de gap locks)
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
SELECT * FROM produits WHERE categorie_id = 5 FOR UPDATE;
-- Pas de gap locks
COMMIT;

-- ✅ Solution 2 : Verrouiller une ligne fictive
CREATE TABLE category_locks (
    categorie_id INT PRIMARY KEY
);

INSERT INTO category_locks VALUES (1), (2), (3), (4), (5), ...;

-- Au lieu de verrouiller les produits, verrouiller la catégorie
START TRANSACTION;
SELECT * FROM category_locks WHERE categorie_id = 5 FOR UPDATE;
-- Maintenant on peut insérer sans deadlock
INSERT INTO produits (categorie_id, nom) VALUES (5, 'Widget');
COMMIT;
```

---

## ✅ Points clés à retenir

- **Deadlock** : Cycle d'attente entre transactions, inévitable mais gérable
- **Détection automatique** : InnoDB détecte et choisit une victime (la plus petite)
- **ERROR 1213** : Code d'erreur du deadlock, doit déclencher un retry
- **Ordre cohérent** : Stratégie #1 de prévention (par ID croissant)
- **Transactions courtes** : Moins de temps avec verrous = moins de deadlocks
- **Retry logic** : Backoff exponentiel + jitter
- **SHOW ENGINE INNODB STATUS** : Outil de diagnostic principal
- **innodb_print_all_deadlocks = ON** : Logger tous les deadlocks
- **Monitoring** : Alerter si > 1 deadlock/min
- **Gap locks** : Peuvent causer des deadlocks, considérer READ COMMITTED

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 InnoDB Deadlock Detection](https://mariadb.com/kb/en/innodb-deadlock-detection-and-rollback/)
- [📖 SHOW ENGINE INNODB STATUS](https://mariadb.com/kb/en/show-engine-innodb-status/)
- [📖 InnoDB Transaction Model](https://mariadb.com/kb/en/innodb-transaction-model/)

### Outils
- [Percona Toolkit - pt-deadlock-logger](https://www.percona.com/doc/percona-toolkit/LATEST/pt-deadlock-logger.html)
- [Percona Monitoring and Management](https://www.percona.com/software/database-tools/percona-monitoring-and-management)

### Articles
- [Understanding InnoDB Deadlocks](https://dev.mysql.com/doc/refman/8.0/en/innodb-deadlocks.html)
- [Deadlock Resolution Strategies](https://www.percona.com/blog/)

---

## ➡️ Section suivante

**6.6 MVCC (Multi-Version Concurrency Control)** : Plongée profonde dans le mécanisme MVCC d'InnoDB.

---


⏭️ [MVCC (Multi-Version Concurrency Control)](/06-transactions-et-concurrence/06-mvcc.md)
