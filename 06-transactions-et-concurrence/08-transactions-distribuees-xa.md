🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.8 Transactions Distribuées (XA)

> **Niveau** : Avancé
> **Durée estimée** : 4-5 heures

> **Prérequis** :
> - Section 6.1 (Propriétés ACID)
> - Section 6.2 (Gestion des transactions)
> - Architecture distribuée
> - Compréhension des systèmes distribués

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :
- Comprendre le protocole XA et le Two-Phase Commit (2PC)
- Implémenter des transactions distribuées avec MariaDB
- Gérer les états et la récupération des transactions XA
- Identifier les limitations et problèmes du protocole XA
- Comparer XA avec les alternatives modernes (Saga, Event Sourcing)
- Décider quand utiliser ou éviter les transactions distribuées
- Implémenter des patterns de cohérence éventuelle

---

## Introduction

Les **transactions distribuées** permettent de coordonner des opérations sur plusieurs bases de données ou ressources de manière atomique. Le protocole **XA** (eXtended Architecture) est le standard industriel pour implémenter ce type de transactions, en utilisant un algorithme appelé **Two-Phase Commit** (2PC).

### Le Problème : Cohérence Multi-Bases

```sql
-- Scénario : Transfert d'argent entre deux banques

-- Banque A (MariaDB 1)
START TRANSACTION;
UPDATE comptes SET solde = solde - 1000 WHERE id = 123;
COMMIT;

-- Banque B (MariaDB 2)
START TRANSACTION;
UPDATE comptes SET solde = solde + 1000 WHERE id = 456;
COMMIT;

-- 💥 PROBLÈME : Que se passe-t-il si :
-- - Le serveur B crash après COMMIT de A ?
-- - Le réseau tombe entre les deux ?
-- - Une contrainte échoue sur B après succès sur A ?

-- Résultat : 1000€ perdus ou dupliqués !
```

### La Solution XA

```
Transaction Distribuée XA :
┌────────────────────────────────────────────────────┐
│ Coordinateur de Transaction (Transaction Manager)  │
│                                                    │
│  1. BEGIN                                          │
│  2. Coordonne les participants                     │
│  3. Two-Phase Commit                               │
│     Phase 1: PREPARE tous les participants         │
│     Phase 2: COMMIT ou ROLLBACK selon résultat     │
└────────────────────────────────────────────────────┘
         │                           │
         ↓                           ↓
┌─────────────────┐         ┌─────────────────┐
│ Participant A   │         │ Participant B   │
│ (MariaDB 1)     │         │ (MariaDB 2)     │
│                 │         │                 │
│ XA START        │         │ XA START        │
│ UPDATE compte A │         │ UPDATE compte B │
│ XA END          │         │ XA END          │
│ XA PREPARE ✅   │         │ XA PREPARE ✅   │
│ XA COMMIT       │         │ XA COMMIT       │
└─────────────────┘         └─────────────────┘

✅ Garantie : Soit TOUT commit, soit TOUT rollback
```

---

## 1. Le Protocole XA : Concepts Fondamentaux

### 1.1 Architecture XA

```
┌────────────────────────────────────────────────────┐
│                 Application                        │
│  (Transaction Manager intégré ou externe)          │
└─────────────────┬──────────────────┬───────────────┘
                  │                  │
                  │ XA Protocol      │ XA Protocol
                  ↓                  ↓
    ┌──────────────────────┐  ┌──────────────────────┐
    │ Resource Manager 1   │  │ Resource Manager 2   │
    │ (MariaDB Server 1)   │  │ (MariaDB Server 2)   │
    │                      │  │                      │
    │ XA-compliant         │  │ XA-compliant         │
    │ Storage Engine       │  │ Storage Engine       │
    │ (InnoDB)             │  │ (InnoDB)             │
    └──────────────────────┘  └──────────────────────┘
```

**Composants** :

1. **Application** : Initie la transaction distribuée
2. **Transaction Manager (TM)** : Coordonne la transaction
3. **Resource Manager (RM)** : Base de données participant (MariaDB)

### 1.2 Two-Phase Commit (2PC)

Le **2PC** est l'algorithme central du protocole XA :

```
Phase 1 : PREPARE (Vote)
┌──────────────────────────────────────────┐
│ Coordinateur : "Êtes-vous prêts ?"       │
└──────────────┬───────────────┬───────────┘
               │               │
               ↓               ↓
         ┌─────────┐     ┌─────────┐
         │  RM A   │     │  RM B   │
         │ ✅ OUI  │     │ ✅ OUI  │
         └─────────┘     └─────────┘

Phase 2 : COMMIT (Decision)
┌──────────────────────────────────────────┐
│ Coordinateur : "COMMIT !"                │
└──────────────┬───────────────┬───────────┘
               │               │
               ↓               ↓
         ┌─────────┐     ┌─────────┐
         │  RM A   │     │  RM B   │
         │ COMMIT  │     │ COMMIT  │
         └─────────┘     └─────────┘

✅ Transaction réussie atomiquement
```

**Cas d'échec** :

```
Phase 1 : PREPARE
┌──────────────────────────────────────────┐
│ Coordinateur : "Êtes-vous prêts ?"       │
└──────────────┬───────────────┬───────────┘
               │               │
               ↓               ↓
         ┌─────────┐     ┌─────────┐
         │  RM A   │     │  RM B   │
         │ ✅ OUI  │     │ ❌ NON  │
         └─────────┘     └─────────┘
                              ↑
                         (Erreur détectée)

Phase 2 : ROLLBACK (Abort)
┌──────────────────────────────────────────┐
│ Coordinateur : "ROLLBACK !"              │
└──────────────┬───────────────┬───────────┘
               │               │
               ↓               ↓
         ┌─────────┐     ┌─────────┐
         │  RM A   │     │  RM B   │
         │ROLLBACK │     │ROLLBACK │
         └─────────┘     └─────────┘

✅ Aucun changement persisté
```

### 1.3 XID : Transaction Identifier

Chaque transaction XA a un identifiant unique (XID) composé de trois parties :

```sql
-- Structure XID
XID = 'gtrid,bqual,formatID'

-- gtrid  : Global Transaction ID (unique globalement)
-- bqual  : Branch Qualifier (identifie la branche/participant)
-- formatID : Format identifier (généralement 1)

-- Exemple
XID = 'tx-2025-001,branch-db1,1'
```

---

## 2. Syntaxe XA dans MariaDB

### 2.1 Vérifier le Support XA

```sql
-- Vérifier si InnoDB supporte XA
SHOW ENGINES;

-- Vérifier les variables XA
SHOW VARIABLES LIKE '%xa%';
-- innodb_support_xa = ON (default)
```

### 2.2 Commandes XA

#### XA START : Démarrer une Transaction Distribuée

```sql
-- Syntaxe
XA START 'xid';
-- ou
XA BEGIN 'xid';

-- Exemple
XA START 'tx-001,branch-1,1';

-- Maintenant en mode XA
UPDATE comptes SET solde = solde - 100 WHERE id = 1;
INSERT INTO logs (message) VALUES ('Transfert en cours');

-- La transaction est active mais pas encore préparée
```

#### XA END : Terminer la Phase Active

```sql
-- Syntaxe
XA END 'xid';

-- Exemple
XA END 'tx-001,branch-1,1';

-- La transaction passe en état IDLE
-- Prête pour PREPARE
```

#### XA PREPARE : Phase 1 du 2PC

```sql
-- Syntaxe
XA PREPARE 'xid';

-- Exemple
XA PREPARE 'tx-001,branch-1,1';

-- Le Resource Manager :
-- 1. Écrit toutes les modifications dans le redo log
-- 2. Garantit la possibilité de COMMIT ou ROLLBACK
-- 3. Répond "PRÊT" au coordinateur
-- 4. Transaction en état PREPARED
```

#### XA COMMIT : Phase 2 (Succès)

```sql
-- Syntaxe
XA COMMIT 'xid';
-- ou avec ONE PHASE (optimisation si 1 seul participant)
XA COMMIT 'xid' ONE PHASE;

-- Exemple - Commit normal (après PREPARE)
XA COMMIT 'tx-001,branch-1,1';

-- Exemple - Commit one-phase (sans PREPARE)
XA START 'tx-002,branch-1,1';
UPDATE comptes SET solde = solde + 100 WHERE id = 2;
XA END 'tx-002,branch-1,1';
XA COMMIT 'tx-002,branch-1,1' ONE PHASE;
-- Optimisation : saute PREPARE si 1 seul participant
```

#### XA ROLLBACK : Annuler la Transaction

```sql
-- Syntaxe
XA ROLLBACK 'xid';

-- Exemple
XA ROLLBACK 'tx-001,branch-1,1';

-- Peut être appelé à tout moment avant COMMIT
```

### 2.3 Commandes de Gestion

```sql
-- Lister les transactions XA en cours
XA RECOVER;
-- Retourne : formatID, gtrid_length, bqual_length, data

-- Exemple de sortie
+----------+--------------+--------------+----------------+
| formatID | gtrid_length | bqual_length | data           |
+----------+--------------+--------------+----------------+
|        1 |            7 |            8 | tx-001branch-1 |
+----------+--------------+--------------+----------------+

-- Parser le XID
-- gtrid = substr(data, 1, gtrid_length) = 'tx-001'
-- bqual = substr(data, gtrid_length+1, bqual_length) = 'branch-1'
```

---

## 3. États d'une Transaction XA

### 3.1 Diagramme d'États

```
                    XA START
                        ↓
            ┌───────────────────────┐
            │   ACTIVE              │
            │  (Modifications en    │
            │   cours)              │
            └───────────┬───────────┘
                        │ XA END
                        ↓
            ┌───────────────────────┐
            │   IDLE                │
            │  (Prêt pour PREPARE)  │
            └───────┬───────────────┘
                    │ XA PREPARE
                    ↓
            ┌───────────────────────┐
            │   PREPARED            │
            │  (Vote : Prêt)        │
            └───┬───────────────┬───┘
                │               │
      XA COMMIT │               │ XA ROLLBACK
                │               │
                ↓               ↓
        ┌──────────────┐   ┌─────────────┐
        │  COMMITTED   │   │ ROLLED BACK │
        └──────────────┘   └─────────────┘
```

### 3.2 États Détaillés

| État | Description | Transitions possibles |
|------|-------------|----------------------|
| **ACTIVE** | Transaction active, modifications en cours | XA END → IDLE<br>XA ROLLBACK → ROLLED BACK |
| **IDLE** | Modifications terminées, en attente de PREPARE | XA PREPARE → PREPARED<br>XA ROLLBACK → ROLLED BACK<br>XA COMMIT ONE PHASE → COMMITTED |
| **PREPARED** | Prêt à commit, vote "OUI" dans 2PC | XA COMMIT → COMMITTED<br>XA ROLLBACK → ROLLED BACK |
| **COMMITTED** | Transaction commitée | - (état final) |
| **ROLLED BACK** | Transaction annulée | - (état final) |

### 3.3 Transitions Interdites

```sql
-- ❌ ERREUR : XA COMMIT sans XA END
XA START 'tx-001,b1,1';
UPDATE t1 SET col = 1;
XA COMMIT 'tx-001,b1,1';
-- ERROR 1399: XAER_RMFAIL: Transaction in ACTIVE state

-- ✅ CORRECT
XA START 'tx-001,b1,1';
UPDATE t1 SET col = 1;
XA END 'tx-001,b1,1';
XA COMMIT 'tx-001,b1,1' ONE PHASE;

-- ❌ ERREUR : XA COMMIT sans XA PREPARE (mode 2PC)
XA START 'tx-002,b1,1';
UPDATE t1 SET col = 2;
XA END 'tx-002,b1,1';
XA COMMIT 'tx-002,b1,1';  -- Sans ONE PHASE
-- ERROR 1399: Transaction in IDLE state, needs PREPARE

-- ✅ CORRECT
XA START 'tx-002,b1,1';
UPDATE t1 SET col = 2;
XA END 'tx-002,b1,1';
XA PREPARE 'tx-002,b1,1';
XA COMMIT 'tx-002,b1,1';
```

---

## 4. Implémentation Complète : Transaction Distribuée

### 4.1 Exemple Python : Transfert Multi-Banques

```python
import pymysql
from typing import List, Tuple

class XACoordinator:
    """Coordinateur de transaction distribuée XA"""

    def __init__(self, connections: List[Tuple[str, pymysql.Connection]]):
        """
        Args:
            connections: Liste de (branch_id, connection)
        """
        self.connections = connections
        self.gtrid = None

    def begin(self, gtrid: str):
        """Démarre une transaction XA sur tous les participants"""
        self.gtrid = gtrid

        for branch_id, conn in self.connections:
            xid = f"{gtrid},{branch_id},1"
            cursor = conn.cursor()
            cursor.execute(f"XA START '{xid}'")
            print(f"[{branch_id}] XA START '{xid}'")

    def execute(self, branch_id: str, sql: str, params=None):
        """Execute une requête sur un participant"""
        conn = self._get_connection(branch_id)
        cursor = conn.cursor()
        cursor.execute(sql, params or ())
        print(f"[{branch_id}] Executed: {sql}")

    def commit(self):
        """Commit 2PC sur tous les participants"""
        try:
            # Phase 1: XA END + XA PREPARE
            print("\n=== PHASE 1: PREPARE ===")
            for branch_id, conn in self.connections:
                xid = f"{self.gtrid},{branch_id},1"
                cursor = conn.cursor()

                # XA END
                cursor.execute(f"XA END '{xid}'")
                print(f"[{branch_id}] XA END '{xid}'")

                # XA PREPARE
                cursor.execute(f"XA PREPARE '{xid}'")
                print(f"[{branch_id}] XA PREPARE '{xid}' ✅")

            # Phase 2: XA COMMIT
            print("\n=== PHASE 2: COMMIT ===")
            for branch_id, conn in self.connections:
                xid = f"{self.gtrid},{branch_id},1"
                cursor = conn.cursor()
                cursor.execute(f"XA COMMIT '{xid}'")
                print(f"[{branch_id}] XA COMMIT '{xid}' ✅")

            print("\n✅ Transaction distribuée RÉUSSIE")
            return True

        except Exception as e:
            print(f"\n❌ ERREUR lors du commit: {e}")
            self.rollback()
            return False

    def rollback(self):
        """Rollback sur tous les participants"""
        print("\n=== ROLLBACK ===")
        for branch_id, conn in self.connections:
            xid = f"{self.gtrid},{branch_id},1"
            try:
                cursor = conn.cursor()
                cursor.execute(f"XA ROLLBACK '{xid}'")
                print(f"[{branch_id}] XA ROLLBACK '{xid}'")
            except Exception as e:
                print(f"[{branch_id}] Erreur rollback: {e}")

    def _get_connection(self, branch_id: str):
        for bid, conn in self.connections:
            if bid == branch_id:
                return conn
        raise ValueError(f"Branch {branch_id} not found")


# Utilisation
def transfer_between_banks(from_account, to_account, amount):
    """
    Transfert d'argent entre deux banques (deux serveurs MariaDB)
    """
    # Connexions aux deux bases
    conn_bank_a = pymysql.connect(
        host='bank-a.example.com',
        user='app',
        password='xxx',
        database='accounts'
    )

    conn_bank_b = pymysql.connect(
        host='bank-b.example.com',
        user='app',
        password='xxx',
        database='accounts'
    )

    # Créer le coordinateur XA
    coordinator = XACoordinator([
        ('bank-a', conn_bank_a),
        ('bank-b', conn_bank_b)
    ])

    try:
        # Démarrer la transaction distribuée
        gtrid = f"transfer-{from_account}-{to_account}-{int(time.time())}"
        coordinator.begin(gtrid)

        # Débiter le compte source (Bank A)
        coordinator.execute(
            'bank-a',
            "UPDATE accounts SET balance = balance - %s WHERE id = %s",
            (amount, from_account)
        )

        # Vérifier le solde
        conn_a = coordinator._get_connection('bank-a')
        cursor_a = conn_a.cursor()
        cursor_a.execute("SELECT balance FROM accounts WHERE id = %s", (from_account,))
        balance = cursor_a.fetchone()[0]

        if balance < 0:
            raise ValueError("Solde insuffisant")

        # Créditer le compte destination (Bank B)
        coordinator.execute(
            'bank-b',
            "UPDATE accounts SET balance = balance + %s WHERE id = %s",
            (amount, to_account)
        )

        # Commit 2PC
        success = coordinator.commit()

        if success:
            print(f"\n✅ Transfert réussi : {amount}€ de {from_account} vers {to_account}")

        return success

    except Exception as e:
        print(f"\n❌ Erreur : {e}")
        coordinator.rollback()
        return False

    finally:
        conn_bank_a.close()
        conn_bank_b.close()


# Test
if __name__ == '__main__':
    transfer_between_banks(
        from_account=123,  # Compte dans Bank A
        to_account=456,    # Compte dans Bank B
        amount=1000.00
    )
```

### 4.2 Exemple Java avec JDBC

```java
import javax.sql.XAConnection;
import javax.sql.XADataSource;
import javax.transaction.xa.XAResource;
import javax.transaction.xa.Xid;
import com.mysql.cj.jdbc.MysqlXADataSource;

public class XATransactionExample {

    public static void main(String[] args) throws Exception {
        // Créer les datasources XA
        MysqlXADataSource ds1 = new MysqlXADataSource();
        ds1.setUrl("jdbc:mysql://bank-a.example.com:3306/accounts");
        ds1.setUser("app");
        ds1.setPassword("xxx");

        MysqlXADataSource ds2 = new MysqlXADataSource();
        ds2.setUrl("jdbc:mysql://bank-b.example.com:3306/accounts");
        ds2.setUser("app");
        ds2.setPassword("xxx");

        // Obtenir les connexions XA
        XAConnection xaConn1 = ds1.getXAConnection();
        XAConnection xaConn2 = ds2.getXAConnection();

        XAResource xaRes1 = xaConn1.getXAResource();
        XAResource xaRes2 = xaConn2.getXAResource();

        // Créer les XIDs
        Xid xid1 = new MyXid(100, new byte[]{0x01}, new byte[]{0x01});
        Xid xid2 = new MyXid(100, new byte[]{0x01}, new byte[]{0x02});

        try {
            // Démarrer les transactions XA
            xaRes1.start(xid1, XAResource.TMNOFLAGS);
            xaRes2.start(xid2, XAResource.TMNOFLAGS);

            // Exécuter les opérations
            Connection conn1 = xaConn1.getConnection();
            Connection conn2 = xaConn2.getConnection();

            // Débiter compte source
            PreparedStatement ps1 = conn1.prepareStatement(
                "UPDATE accounts SET balance = balance - ? WHERE id = ?"
            );
            ps1.setDouble(1, 1000.00);
            ps1.setInt(2, 123);
            ps1.executeUpdate();

            // Créditer compte destination
            PreparedStatement ps2 = conn2.prepareStatement(
                "UPDATE accounts SET balance = balance + ? WHERE id = ?"
            );
            ps2.setDouble(1, 1000.00);
            ps2.setInt(2, 456);
            ps2.executeUpdate();

            // Terminer la phase active
            xaRes1.end(xid1, XAResource.TMSUCCESS);
            xaRes2.end(xid2, XAResource.TMSUCCESS);

            // Phase 1: PREPARE
            int prp1 = xaRes1.prepare(xid1);
            int prp2 = xaRes2.prepare(xid2);

            if (prp1 == XAResource.XA_OK && prp2 == XAResource.XA_OK) {
                // Phase 2: COMMIT
                xaRes1.commit(xid1, false);
                xaRes2.commit(xid2, false);
                System.out.println("✅ Transaction distribuée réussie");
            } else {
                // Rollback
                xaRes1.rollback(xid1);
                xaRes2.rollback(xid2);
                System.out.println("❌ Transaction distribuée annulée");
            }

        } catch (Exception e) {
            // Rollback en cas d'erreur
            xaRes1.rollback(xid1);
            xaRes2.rollback(xid2);
            e.printStackTrace();
        } finally {
            xaConn1.close();
            xaConn2.close();
        }
    }
}
```

---

## 5. Récupération et Gestion des Pannes

### 5.1 Transactions XA en Attente

Après un crash, des transactions peuvent rester en état PREPARED :

```sql
-- Après redémarrage du serveur
-- Vérifier les transactions en attente
XA RECOVER;

+----------+--------------+--------------+------------------+
| formatID | gtrid_length | bqual_length | data             |
+----------+--------------+--------------+------------------+
|        1 |           10 |            7 | tx-001-123bank-a |
|        1 |           10 |            7 | tx-002-456bank-b |
+----------+--------------+--------------+------------------+

-- Ces transactions étaient PREPARED avant le crash
-- Le coordinateur doit décider : COMMIT ou ROLLBACK ?
```

### 5.2 Récupération Manuelle

```sql
-- Scénario : Le coordinateur a décidé de COMMIT
-- mais n'a pas pu envoyer la commande avant le crash

-- Reconstruire le XID depuis XA RECOVER
SET @xid = 'tx-001-123,bank-a,1';

-- Commit la transaction
XA COMMIT @xid;

-- Ou rollback si le coordinateur a décidé d'annuler
XA ROLLBACK @xid;
```

### 5.3 Script de Récupération Automatique

```python
import pymysql
import logging

class XARecoveryManager:
    """Gestionnaire de récupération des transactions XA"""

    def __init__(self, connection):
        self.conn = connection
        self.logger = logging.getLogger(__name__)

    def recover_prepared_transactions(self):
        """Récupère et résout les transactions PREPARED"""
        cursor = self.conn.cursor()

        # Lister les transactions PREPARED
        cursor.execute("XA RECOVER")
        prepared_txs = cursor.fetchall()

        if not prepared_txs:
            self.logger.info("Aucune transaction XA en attente")
            return

        self.logger.warning(f"{len(prepared_txs)} transaction(s) XA en attente")

        for tx in prepared_txs:
            format_id = tx[0]
            gtrid_length = tx[1]
            bqual_length = tx[2]
            data = tx[3]

            # Parser le XID
            gtrid = data[:gtrid_length].decode('utf-8')
            bqual = data[gtrid_length:gtrid_length+bqual_length].decode('utf-8')
            xid = f"{gtrid},{bqual},{format_id}"

            self.logger.info(f"Transaction trouvée: {xid}")

            # Consulter le coordinateur pour savoir quoi faire
            # (Dans un vrai système, interroger un service de coordination)
            decision = self.consult_coordinator(gtrid)

            if decision == 'COMMIT':
                self.logger.info(f"Commit de {xid}")
                cursor.execute(f"XA COMMIT '{xid}'")
            elif decision == 'ROLLBACK':
                self.logger.info(f"Rollback de {xid}")
                cursor.execute(f"XA ROLLBACK '{xid}'")
            else:
                # Décision inconnue, attendre
                self.logger.warning(f"Décision inconnue pour {xid}, en attente")

    def consult_coordinator(self, gtrid):
        """
        Consulte le coordinateur de transaction pour obtenir la décision

        Dans un système réel, cela pourrait être :
        - Une requête à un service de coordination
        - Une consultation dans une base de données de log
        - Un message dans une queue
        """
        # Implémentation simplifiée
        # À adapter selon votre architecture

        # Exemple : Consulter une table de log
        cursor = self.conn.cursor()
        cursor.execute(
            "SELECT decision FROM xa_coordinator_log WHERE gtrid = %s",
            (gtrid,)
        )
        result = cursor.fetchone()

        if result:
            return result[0]  # 'COMMIT' ou 'ROLLBACK'
        else:
            return 'UNKNOWN'

# Utilisation au démarrage du serveur
def on_server_startup():
    conn = pymysql.connect(...)
    recovery_manager = XARecoveryManager(conn)
    recovery_manager.recover_prepared_transactions()
    conn.close()
```

---

## 6. Limitations et Problèmes du Protocole XA

### 6.1 Problème du Coordinateur (Single Point of Failure)

```
Scénario : Coordinateur crash après PREPARE

Phase 1: PREPARE
Coordinateur  →  [PREPARE] → RM A ✅
Coordinateur  →  [PREPARE] → RM B ✅

💥 CRASH DU COORDINATEUR

RM A : État PREPARED (bloqué, attend la décision)
RM B : État PREPARED (bloqué, attend la décision)

Problème :
- Les RMs tiennent des verrous
- Les transactions sont bloquées
- Personne ne peut décider (COMMIT ou ROLLBACK)
- Deadlock distribué jusqu'à récupération du coordinateur

⏱️ Timeout nécessaire (typiquement plusieurs minutes)
```

### 6.2 Problème de Performance

**Latence élevée** :

```
Transaction Locale (1 base) :
┌─────────────────────────────────┐
│ BEGIN                           │
│ UPDATE                          │
│ COMMIT                          │
└─────────────────────────────────┘
Durée : ~10ms

Transaction XA (2 bases) :
┌─────────────────────────────────┐
│ XA START (2 bases)              │  +2ms
│ UPDATE (2 bases)                │  +20ms (réseau)
│ XA END (2 bases)                │  +2ms
│ XA PREPARE (2 bases)            │  +20ms (réseau + fsync)
│   Attente réponses              │  +10ms
│ XA COMMIT (2 bases)             │  +20ms (réseau + fsync)
└─────────────────────────────────┘
Durée : ~74ms (7x plus lent)

Impact :
- Throughput divisé par 7
- Latence multipliée par 7
- Pas de scalabilité linéaire
```

**Benchmark réel** :

```sql
-- Test sur infrastructure réelle

-- Transaction locale (1 base)
-- TPS : 10,000 transactions/seconde

-- Transaction XA (2 bases)
-- TPS : 800 transactions/seconde

-- Perte : 92% de performance
```

### 6.3 Verrous Gardés Plus Longtemps

```sql
-- Transaction locale
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
-- Verrou gardé : 10ms

-- Transaction XA
XA START 'tx-1,b1,1';
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
XA END 'tx-1,b1,1';
XA PREPARE 'tx-1,b1,1';
-- ⏳ Attente décision coordinateur (peut être 100ms+)
XA COMMIT 'tx-1,b1,1';
-- Verrou gardé : 100ms+ (10x plus longtemps)

💥 Conséquences :
- Plus de contention
- Plus de deadlocks
- Moins de concurrence
```

### 6.4 Problème de Cohérence

**Heuristic Completion** : Décision unilatérale d'un RM

```sql
-- Scénario catastrophe

Phase 1: PREPARE
RM A : ✅ PREPARED
RM B : ✅ PREPARED

💥 Coordinateur inaccessible pendant 30 minutes

RM A : Timeout atteint, décide ROLLBACK (heuristic)
RM B : Timeout atteint, décide COMMIT (heuristic)

💥 INCOHÉRENCE :
- RM A a rollback
- RM B a commit
- Données incohérentes entre les bases
```

### 6.5 Complexité Opérationnelle

```
Défis opérationnels :

1. Monitoring
   - Surveiller l'état de chaque RM
   - Détecter les transactions bloquées
   - Alerter sur les heuristic completions

2. Debugging
   - Tracer les transactions distribuées
   - Identifier la cause d'un échec
   - Reconstruire l'historique

3. Récupération
   - Script de récupération automatique
   - Procédures de rollback manuel
   - Documentation des XIDs

4. Testing
   - Tests de pannes réseau
   - Tests de crash coordinateur
   - Tests de crash RM

🔴 Complexité >> Transaction locale
```

---

## 7. Alternatives Modernes au XA

### 7.1 Saga Pattern

**Principe** : Séquence de transactions locales avec compensations.

```
Au lieu de :
┌────────────────────────────────────┐
│ Transaction XA (atomique)          │
│  - Débiter compte A                │
│  - Créditer compte B               │
│  - Logger audit                    │
└────────────────────────────────────┘

Faire :
┌────────────────────────────────────┐
│ Saga (séquence)                    │
│                                    │
│ 1. Transaction locale : Débiter A  │
│    Compensation : Re-créditer A    │
│                                    │
│ 2. Transaction locale : Créditer B │
│    Compensation : Re-débiter B     │
│                                    │
│ 3. Transaction locale : Logger     │
│    Compensation : Logger rollback  │
└────────────────────────────────────┘

Si échec à l'étape 3 :
→ Compenser 2 (re-débiter B)
→ Compenser 1 (re-créditer A)
```

**Implémentation** :

```python
from typing import Callable, List, Tuple

class SagaStep:
    def __init__(self,
                 action: Callable,
                 compensation: Callable,
                 name: str):
        self.action = action
        self.compensation = compensation
        self.name = name

class SagaOrchestrator:
    def __init__(self):
        self.steps: List[SagaStep] = []
        self.completed_steps: List[SagaStep] = []

    def add_step(self, step: SagaStep):
        self.steps.append(step)

    def execute(self):
        """Execute la saga avec compensations si nécessaire"""
        try:
            # Exécuter chaque étape
            for step in self.steps:
                print(f"Exécution : {step.name}")
                step.action()
                self.completed_steps.append(step)
                print(f"✅ {step.name} réussi")

            print("\n✅ Saga complète réussie")
            return True

        except Exception as e:
            print(f"\n❌ Erreur lors de {step.name}: {e}")
            print("Compensation en cours...")

            # Compenser dans l'ordre inverse
            for completed_step in reversed(self.completed_steps):
                try:
                    print(f"Compensation : {completed_step.name}")
                    completed_step.compensation()
                    print(f"✅ Compensation {completed_step.name} réussie")
                except Exception as comp_error:
                    print(f"❌ Échec compensation {completed_step.name}: {comp_error}")

            return False

# Exemple : Transfert avec Saga
def transfer_with_saga(from_account, to_account, amount):
    saga = SagaOrchestrator()

    # Étape 1 : Débiter compte source
    def debit_source():
        conn = get_db_connection('bank-a')
        cursor = conn.cursor()
        cursor.execute(
            "UPDATE accounts SET balance = balance - %s WHERE id = %s",
            (amount, from_account)
        )
        conn.commit()
        conn.close()

    def compensate_debit():
        conn = get_db_connection('bank-a')
        cursor = conn.cursor()
        cursor.execute(
            "UPDATE accounts SET balance = balance + %s WHERE id = %s",
            (amount, from_account)
        )
        conn.commit()
        conn.close()

    saga.add_step(SagaStep(
        action=debit_source,
        compensation=compensate_debit,
        name="Débiter compte source"
    ))

    # Étape 2 : Créditer compte destination
    def credit_destination():
        conn = get_db_connection('bank-b')
        cursor = conn.cursor()
        cursor.execute(
            "UPDATE accounts SET balance = balance + %s WHERE id = %s",
            (amount, to_account)
        )
        conn.commit()
        conn.close()

    def compensate_credit():
        conn = get_db_connection('bank-b')
        cursor = conn.cursor()
        cursor.execute(
            "UPDATE accounts SET balance = balance - %s WHERE id = %s",
            (amount, to_account)
        )
        conn.commit()
        conn.close()

    saga.add_step(SagaStep(
        action=credit_destination,
        compensation=compensate_credit,
        name="Créditer compte destination"
    ))

    # Étape 3 : Logger la transaction
    def log_transfer():
        conn = get_db_connection('logging-db')
        cursor = conn.cursor()
        cursor.execute(
            "INSERT INTO transfer_log (from_account, to_account, amount) "
            "VALUES (%s, %s, %s)",
            (from_account, to_account, amount)
        )
        conn.commit()
        conn.close()

    def compensate_log():
        # Logger que le transfert a été annulé
        conn = get_db_connection('logging-db')
        cursor = conn.cursor()
        cursor.execute(
            "INSERT INTO transfer_log (from_account, to_account, amount, status) "
            "VALUES (%s, %s, %s, 'COMPENSATED')",
            (from_account, to_account, amount)
        )
        conn.commit()
        conn.close()

    saga.add_step(SagaStep(
        action=log_transfer,
        compensation=compensate_log,
        name="Logger le transfert"
    ))

    # Exécuter la saga
    return saga.execute()

# Utilisation
transfer_with_saga(123, 456, 1000.00)
```

**Avantages du Saga** :
- ✅ Pas de coordinateur centralisé
- ✅ Pas de verrous longue durée
- ✅ Meilleure performance (transactions locales)
- ✅ Plus résilient aux pannes

**Inconvénients** :
- ⚠️ Cohérence éventuelle (pas immédiate)
- ⚠️ Compensations peuvent échouer
- ⚠️ Complexité applicative plus élevée

### 7.2 Event Sourcing + CQRS

**Principe** : Stocker les événements, pas l'état final.

```
Au lieu de :
UPDATE accounts SET balance = balance - 100

Faire :
INSERT INTO events (type, account_id, amount, timestamp)
VALUES ('DEBIT', 123, 100, NOW())

Le solde est calculé en rejouant les événements :
SELECT SUM(
    CASE WHEN type = 'CREDIT' THEN amount
         WHEN type = 'DEBIT' THEN -amount
    END
) FROM events WHERE account_id = 123
```

**Avantages** :
- ✅ Audit complet (historique immuable)
- ✅ Pas de transactions distribuées
- ✅ Scalabilité horizontale
- ✅ Time-travel (état à n'importe quel moment)

### 7.3 Outbox Pattern

**Principe** : Garantir la cohérence via une table outbox locale.

```sql
-- Table outbox dans chaque base
CREATE TABLE outbox (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    event_type VARCHAR(100),
    payload JSON,
    published BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Transaction locale incluant l'outbox
START TRANSACTION;

-- 1. Modifier les données
UPDATE accounts SET balance = balance - 100 WHERE id = 123;

-- 2. Insérer l'événement dans l'outbox
INSERT INTO outbox (event_type, payload)
VALUES ('ACCOUNT_DEBITED', JSON_OBJECT(
    'account_id', 123,
    'amount', 100,
    'timestamp', NOW()
));

COMMIT;

-- Un worker lit l'outbox et publie les événements
-- Une fois publié, marquer published = TRUE
```

**Avantages** :
- ✅ Transaction locale atomique
- ✅ Garantie de publication d'événement
- ✅ Pattern simple

---

## 8. Quand Utiliser (ou Éviter) XA

### 8.1 Utiliser XA Si...

```
✅ Cas d'usage légitimes pour XA :

1. Conformité réglementaire stricte
   - Banking : Transactions financières critiques
   - Healthcare : Données patient multi-systèmes
   - Contexte : Atomicité EXIGÉE par la loi

2. Migration legacy
   - Système existant utilise déjà XA
   - Coût de refactoring prohibitif
   - Contexte : Dette technique acceptée temporairement

3. Transactions rares et critiques
   - Fréquence : < 10/jour
   - Importance : Correction manuelle inacceptable
   - Contexte : Performance non critique

4. Infrastructure homogène et fiable
   - Réseau stable, latence < 2ms
   - Infra redondante (coordinateur HA)
   - Contexte : Risque de panne minimisé
```

### 8.2 Éviter XA Si...

```
❌ Éviter XA dans ces cas :

1. High-throughput
   - > 100 TPS par participant
   - Performance critique
   - Alternative : Saga ou Event Sourcing

2. Latence réseau élevée
   - Participants géographiquement distribués
   - Latence > 10ms entre participants
   - Alternative : Cohérence éventuelle

3. Microservices modernes
   - Architecture découplée
   - Scalabilité horizontale
   - Alternative : Event-driven architecture

4. Début de projet
   - Pas de legacy XA existant
   - Flexibilité de design
   - Alternative : Saga pattern dès le départ

5. Pas d'expertise XA en équipe
   - Complexité opérationnelle sous-estimée
   - Debugging difficile
   - Alternative : Patterns plus simples
```

### 8.3 Tableau de Décision

| Critère | XA | Saga | Event Sourcing |
|---------|----|----|----------------|
| **Atomicité stricte** | ✅ Oui | ⚠️ Éventuelle | ⚠️ Éventuelle |
| **Performance** | ❌ Faible | ✅ Élevée | ✅ Élevée |
| **Complexité** | 🔴 Élevée | 🟡 Moyenne | 🔴 Élevée |
| **Resilience** | ❌ SPOF | ✅ Distribuée | ✅ Distribuée |
| **Audit trail** | ⚠️ Logs | ✅ Compensations | ✅ Complet |
| **Latence** | ❌ Élevée | ✅ Faible | ✅ Faible |
| **Cas d'usage** | Banking legacy | E-commerce | Audit-heavy |

---

## ✅ Points clés à retenir

- **XA** : Protocole standard pour transactions distribuées
- **2PC** : Two-Phase Commit (PREPARE puis COMMIT)
- **XID** : Identifiant unique (gtrid, bqual, formatID)
- **États** : ACTIVE → IDLE → PREPARED → COMMITTED/ROLLED BACK
- **Performance** : 7-10x plus lent que transaction locale
- **SPOF** : Coordinateur est un point de défaillance unique
- **Récupération** : XA RECOVER pour transactions bloquées
- **Alternatives** : Saga, Event Sourcing, Outbox pattern
- **Quand utiliser** : Conformité stricte, legacy, < 10 TPS
- **Quand éviter** : High-throughput, microservices, latence réseau

---

## 🔗 Ressources et références

### Documentation officielle MariaDB
- [📖 XA Transactions](https://mariadb.com/kb/en/xa-transactions/)
- [📖 XA Transaction States](https://mariadb.com/kb/en/xa-transaction-states/)
- [📖 XA Transaction SQL Syntax](https://mariadb.com/kb/en/xa-transactions-sql-statements/)

### Standards
- [X/Open XA Specification](https://pubs.opengroup.org/onlinepubs/009680699/toc.pdf)
- [DTP Model (Distributed Transaction Processing)](https://pubs.opengroup.org/onlinepubs/9294999599/toc.pdf)

### Articles et livres
- "Designing Data-Intensive Applications" - Martin Kleppmann (Chapitre 9: Consistency and Consensus)
- [Pattern: Saga](https://microservices.io/patterns/data/saga.html)
- [Two-Phase Commit Protocol](https://en.wikipedia.org/wiki/Two-phase_commit_protocol)

---

## 🎓 Conclusion du Chapitre

Vous avez maintenant une compréhension approfondie des **transactions et de la concurrence** dans MariaDB :

1. **Propriétés ACID** : Garanties fondamentales
2. **Niveaux d'isolation** : Trade-offs performance vs cohérence
3. **MVCC** : Mécanisme permettant lectures sans blocage
4. **Verrous** : Synchronisation des accès concurrents
5. **Deadlocks** : Détection et résolution
6. **Savepoints** : Rollback partiel
7. **Transactions XA** : Coordination distribuée avec compromis

**Message final** : Les transactions distribuées (XA) sont puissantes mais complexes. Dans 90% des cas, les alternatives modernes (Saga, Event Sourcing) offrent un meilleur compromis entre cohérence et performance. Utilisez XA uniquement quand absolument nécessaire.

---


⏭️ [Moteurs de Stockage](/07-moteurs-de-stockage/README.md)
