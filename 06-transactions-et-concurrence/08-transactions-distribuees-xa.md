🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.8 Transactions distribuées (XA)

Une transaction ordinaire est confinée à **une seule** base ([§6.2](02-gestion-transactions.md)). Mais certaines opérations doivent être **atomiques à travers plusieurs ressources** : deux serveurs MariaDB distincts, ou une base de données **et** un courtier de messages, par exemple. Les **transactions XA** répondent à ce besoin : elles permettent de valider (ou d'annuler) **ensemble**, de façon atomique, un travail réparti sur plusieurs systèmes.

XA est un **standard** (X/Open XA, modèle de traitement transactionnel distribué) implémenté par InnoDB.

## Le protocole de validation en deux phases (2PC)

La coordination repose sur le **two-phase commit** (2PC, validation en deux phases). Deux rôles :

- un **gestionnaire de transactions** (*Transaction Manager*, TM) — le **coordinateur** ;
- plusieurs **gestionnaires de ressources** (*Resource Managers*, RM) — par exemple chaque instance MariaDB impliquée.

Le déroulement :

1. **Phase 1 — `PREPARE`** : le coordinateur demande à chaque ressource de **se préparer**. Chacune effectue son travail, l'enregistre de façon **durable**, puis **vote** : « prêt à valider » ou « impossible ». Une fois préparée, une ressource **garantit** pouvoir aussi bien valider qu'annuler, **même après un crash**.
2. **Phase 2 — `COMMIT` (ou `ROLLBACK`)** : si **toutes** les ressources ont voté « prêt », le coordinateur leur ordonne de **valider** ; si **l'une** a échoué ou refusé, il ordonne à toutes d'**annuler**.

C'est ce vote unanime suivi d'une décision unique qui assure l'**atomicité** à l'échelle distribuée.

## Syntaxe XA dans MariaDB

Une transaction XA est identifiée par un **`xid`** (identifiant global), de forme `gtrid[, bqual[, formatID]]`. Les instructions suivent les phases du 2PC :

```sql
XA START 'trx1';                       -- ouvre la transaction XA (état ACTIVE)
  UPDATE comptes SET solde = solde - 100 WHERE id = 1;
XA END 'trx1';                         -- fin de la phase active (état IDLE)

XA PREPARE 'trx1';                     -- phase 1 : préparation durable (état PREPARED)
-- … le coordinateur vérifie les autres ressources …
XA COMMIT 'trx1';                      -- phase 2 : validation
-- ou : XA ROLLBACK 'trx1';
```

- **`XA START` / `XA END`** délimitent les opérations de la branche.
- **`XA PREPARE`** réalise la phase 1 : après elle, la transaction est **recouvrable**.
- **`XA COMMIT`** (phase 2) accepte l'option **`ONE PHASE`**, qui combine préparation et validation lorsqu'**une seule** ressource est concernée (le 2PC complet devient alors inutile).
- **`XA RECOVER`** liste les transactions à l'état `PREPARED` — essentiel pour la **reprise après incident**.

## États d'une transaction XA

```
 XA START            XA END           XA PREPARE        XA COMMIT / ROLLBACK
    │                   │                  │                      │
    ▼                   ▼                  ▼                      ▼
 ACTIVE  ─────────►   IDLE   ─────────► PREPARED ─────────►  (terminée)
```

À l'état **`PREPARED`**, la transaction survit à un redémarrage du serveur : c'est ce qui permet au coordinateur de prendre sa décision même après une panne. On retrouve ces transactions en attente via `XA RECOVER`, pour les valider ou les annuler.

## Contraintes et points de vigilance

- **Une seule transaction XA à la fois par connexion** ; on ne peut pas la mêler à une transaction ordinaire ni l'imbriquer.
- ⚠️ **Transactions « en suspens » (*in-doubt*)** : si le coordinateur disparaît **après** la phase `PREPARE` mais **avant** la décision, les transactions préparées restent à l'état `PREPARED` — **conservant leurs verrous** — jusqu'à résolution manuelle (`XA RECOVER`, puis `XA COMMIT` / `XA ROLLBACK`). C'est le principal risque opérationnel du 2PC.
- **Coût en performance** : le 2PC ajoute des allers-retours réseau et une écriture durable supplémentaire (la préparation). À réserver aux cas qui l'exigent réellement.

## 2PC interne : binlog ↔ InnoDB

Le 2PC n'est pas réservé au XA explicite. Lorsque la **journalisation binaire** est active, MariaDB utilise en interne un mécanisme de type 2PC pour garder le **binlog** et le **redo log d'InnoDB** cohérents au moment du `COMMIT` (afin qu'après un crash, journal binaire et données soient toujours d'accord).

> 🆕 **MariaDB 12.3** : un nouveau **binlog intégré à InnoDB** (MDEV-34705, livré en 12.3.1) stocke le journal binaire dans les *tablespaces* InnoDB et **supprime cette synchronisation** coûteuse entre deux journaux séparés — MariaDB le présente comme la **plus importante amélioration de performance OLTP de la 12.3** (avec, en prime, une meilleure sûreté au crash, binlog et données InnoDB restant toujours cohérents). ⚠️ C'est une fonctionnalité **optionnelle** (à activer explicitement), assortie de limitations à connaître avant tout usage en production. Voir [§11.5.4](../11-administration-configuration/05.4-binlog-innodb-performance.md).

## Cas d'usage et alternatives

Les transactions XA conviennent quand une atomicité **stricte** est requise entre ressources hétérogènes : plusieurs bases, ou une base couplée à un système de messagerie.

> 💡 **Tendance actuelle** : dans les architectures **microservices**, on évite souvent le XA — lourd et fragile en disponibilité (problème des transactions en suspens) — au profit de patrons à **cohérence à terme** : *saga*, *transactional outbox*, capture de changements (**CDC**). Voir [§20.2 — Architecture microservices](../20-cas-usage-architectures/02-architecture-microservices.md) et [§20.8 — Architectures Event-Driven](../20-cas-usage-architectures/08-architectures-event-driven.md).

## À retenir

- Les **transactions XA** assurent l'atomicité d'une transaction répartie sur **plusieurs ressources**, via le **two-phase commit** (2PC).
- 2PC = **phase 1 `PREPARE`** (chaque ressource se prépare durablement et vote), puis **phase 2 `COMMIT`/`ROLLBACK`** (décision unanime).
- Instructions MariaDB : `XA START` / `XA END` / `XA PREPARE` / `XA COMMIT [ONE PHASE]` / `XA ROLLBACK` / `XA RECOVER` ; états `ACTIVE → IDLE → PREPARED → terminée`.
- ⚠️ Risque des transactions **en suspens** (`PREPARED`) qui retiennent des verrous si le coordinateur tombe ; coût de performance du 2PC.
- Un 2PC **interne** coordonne binlog et InnoDB ; en 12.3, un **binlog intégré à InnoDB** (optionnel) **élimine cette synchro** ([§11.5.4](../11-administration-configuration/05.4-binlog-innodb-performance.md)).
- Souvent remplacé en microservices par des patrons à cohérence à terme (saga, outbox, CDC).

---

> **Section suivante** : [6.9 — Snapshot Isolation](09-snapshot-isolation.md), l'isolation par instantané que MariaDB applique par défaut à `REPEATABLE READ`.

⏭️ [Snapshot Isolation : innodb_snapshot_isolation = ON par défaut (impact REPEATABLE READ)](/06-transactions-et-concurrence/09-snapshot-isolation.md)
