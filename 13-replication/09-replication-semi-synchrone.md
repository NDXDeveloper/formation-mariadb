🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.9 — Réplication semi-synchrone

> **Chapitre 13 — Réplication** · Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

La réplication **semi-synchrone** fait **attendre la source** qu'au moins un réplica ait **accusé réception** d'une transaction avant de confirmer la validation au client. Elle réduit ainsi fortement la **fenêtre de perte** lors d'un *failover*, au prix d'un aller-retour réseau supplémentaire.

Les **concepts** (asynchrone vs semi-synchrone, points d'attente, compromis) sont présentés en **13.1** ; cette section en détaille la **configuration** et l'**exploitation**. Sous MariaDB, la fonctionnalité est **intégrée au serveur** depuis la 10.3.3 — **aucun plugin** à installer.

---

## 1. Rappel du principe

Lors du `COMMIT`, la source attend l'**ACK** (réception **et** journalisation dans le relay log) d'au moins un réplica avant de rendre la main au client. Un thread dédié côté source, l'**ACK Receiver Thread**, collecte ces acquittements. On garantit ainsi qu'au moins un réplica **détient** chaque transaction confirmée — sans toutefois attendre qu'il l'ait *appliquée* (d'où « semi »). Voir 13.1 pour la comparaison complète avec l'asynchrone.

---

## 2. Activation

> 💡 En MariaDB, la semi-synchrone est **native** (depuis 10.3.3) : contrairement à d'anciens tutoriels (et à MySQL historique), **aucun `plugin_load_add`** n'est requis. Il suffit d'activer les variables.

**Sur la source** et **sur le réplica** :

```ini
# Source
[mariadb]
rpl_semi_sync_master_enabled = ON

# Réplica
[mariadb]
rpl_semi_sync_slave_enabled = ON
```

L'activation est aussi **dynamique** (sans redémarrage du serveur) ; côté réplica, on relance le thread IO pour qu'elle prenne effet :

```sql
-- Source
SET GLOBAL rpl_semi_sync_master_enabled = ON;

-- Réplica
SET GLOBAL rpl_semi_sync_slave_enabled = ON;
STOP REPLICA IO_THREAD; START REPLICA IO_THREAD;
```

**Configuration prête pour le failover :** pour qu'un serveur puisse être **indifféremment source ou réplica** (topologie destinée aux bascules, souvent pilotée par MaxScale), on active **les deux** variables sur **chaque** nœud :

```ini
[mariadb]
rpl_semi_sync_master_enabled = ON
rpl_semi_sync_slave_enabled  = ON
```

---

## 3. Le point d'attente : `AFTER_COMMIT` vs `AFTER_SYNC`

La variable `rpl_semi_sync_master_wait_point` détermine **quand** la source attend l'acquittement (détaillé en 13.1) :

| Valeur | Comportement | Conséquence |
|--------|--------------|-------------|
| **`AFTER_COMMIT`** *(défaut MariaDB)* | Commit local **puis** attente de l'ACK | D'autres clients peuvent voir une transaction **pas encore acquittée** par un réplica → risque de lecture « fantôme » au *failover*. |
| **`AFTER_SYNC`** *(« sans perte »)* | Attente de l'ACK **puis** commit local | Aucun client ne voit une donnée qu'aucun réplica ne possède → variante à **privilégier pour la durabilité**. |

```ini
[mariadb]
rpl_semi_sync_master_wait_point = AFTER_SYNC
```

> ⚠️ **Binlog InnoDB (12.3) — semi-synchrone indisponible :** dans sa **première implémentation** (12.3), le binlog intégré à InnoDB (13.2.1) **ne prend pas en charge la réplication semi-synchrone** *du tout* : `SET GLOBAL rpl_semi_sync_master_enabled = ON` échoue avec l'erreur **`ERROR 4248 : Semi-synchronous replication is not yet supported with --binlog-storage-engine`** (vérifié sur 12.3.2). La cause de fond — la suppression du commit en deux phases (2PC) entre binlog et InnoDB — interdit en particulier `AFTER_SYNC`, mais cette première version ne gère **aucun** mode semi-synchrone. **Pour utiliser la semi-synchrone, il faut donc conserver le binlog traditionnel** (basé sur fichiers). C'est un arbitrage majeur entre le gain en écriture du binlog InnoDB et la semi-synchrone.

---

## 4. Le timeout et le repli vers l'asynchrone

La semi-synchrone ne doit **jamais** bloquer indéfiniment la production. Si aucun réplica n'acquitte dans le délai imparti, la source **bascule automatiquement en asynchrone** :

```ini
[mariadb]
rpl_semi_sync_master_timeout = 10000   # millisecondes — 10 s par défaut
```

Au-delà de ce délai, `Rpl_semi_sync_master_status` passe à **`OFF`** et la réplication continue en mode asynchrone. Dès qu'un réplica semi-synchrone **rattrape** son retard, la source **rebascule** en semi-synchrone (`Rpl_semi_sync_master_status` repasse à `ON`).

> ⚠️ Ce repli **préserve la disponibilité** mais **dégrade silencieusement la durabilité** : il faut donc **surveiller** `Rpl_semi_sync_master_status` (§6).

---

## 5. Combien de réplicas doivent acquitter ?

Sous MariaDB, **l'acquittement d'un seul réplica suffit**, et ce nombre **n'est pas configurable**. Contrairement à MySQL (variable `rpl_semi_sync_master_wait_for_slave_count`), **MariaDB ne dispose pas** de cette variable : l'attente porte toujours sur **un** ACK (le portage demandé par MDEV-18983 reste non implémenté — `SELECT @@rpl_semi_sync_master_wait_for_slave_count` renvoie *Unknown system variable* sur 12.3.2).

MariaDB propose en revanche **`rpl_semi_sync_master_wait_no_slave`** (`ON` par défaut) : si `ON`, la source **continue d'attendre** un ACK (jusqu'au timeout) même lorsqu'**aucun réplica n'est connecté**, plutôt que de basculer aussitôt en asynchrone.

```ini
[mariadb]
rpl_semi_sync_master_wait_no_slave = ON   # attendre même sans réplica connecté (défaut)
```

---

## 6. Surveiller la semi-synchrone

L'état s'inspecte via les variables `Rpl_semi_sync%` :

```sql
SHOW STATUS LIKE 'Rpl_semi_sync%';
```

| Variable d'état | Signification |
|-----------------|---------------|
| `Rpl_semi_sync_master_status` | `ON` si la semi-synchrone est **effectivement active** (passe à `OFF` après un timeout). |
| `Rpl_semi_sync_master_clients` | Nombre de réplicas semi-synchrones **connectés**. |
| `Rpl_semi_sync_master_yes_tx` / `Rpl_semi_sync_master_no_tx` | Transactions **acquittées** vs **non acquittées** (repli async). |
| `Rpl_semi_sync_master_wait_sessions` | Sessions en attente d'acquittement. |
| `Rpl_semi_sync_slave_status` | `ON` sur le réplica si la semi-synchrone y est active et le thread IO en marche. |

> ✅ **À alerter en priorité :** `Rpl_semi_sync_master_status = OFF` signale un **repli silencieux en asynchrone** — la protection escomptée n'est plus en vigueur.

---

## 7. Durabilité, crash et robustesse (amélioration récente)

Au-delà du point d'attente, un scénario de **crash** méritait attention : par le passé, une source configurée avec `rpl_semi_sync_master_enabled` **et** `rpl_semi_sync_slave_enabled` pouvait, après redémarrage, **tronquer son binlog** et perdre des transactions que des réplicas avaient pourtant déjà reçues et exécutées — plaçant un réplica en **état d'erreur** (`gtid_slave_pos` en avance sur le `gtid_binlog_pos` de la source).

Ce comportement a été **corrigé** (dans les versions de maintenance 10.6.19, 10.11.9, 11.x — donc **présent en 12.3**) : sous réserve de **ne pas** forcer `--init-rpl-role=SLAVE`, une source redémarrée **ne tronque plus** les transactions nécessaires aux réplicas. On peut donc conserver **les deux** variables semi-synchrones actives sur un même serveur **sans risque de perte au redémarrage** — ce qui est idéal pour les topologies prêtes au failover (§2, 13.8). Par ailleurs, la **récupération du réplica semi-synchrone** après crash garantit l'absence de transactions surnuméraires (le binlog est tronqué des transactions non prouvées comme validées).

---

## 8. Exemple de configuration complète

### Source

```ini
[mariadb]
rpl_semi_sync_master_enabled    = ON
rpl_semi_sync_master_wait_point = AFTER_SYNC   # sans perte (nécessite le binlog fichiers ;
                                               # semi-sync indisponible avec le binlog InnoDB)
rpl_semi_sync_master_timeout    = 10000        # 10 s
# MariaDB attend l'ACK d'UN réplica (non configurable — pas de wait_for_slave_count)
```

### Réplica

```ini
[mariadb]
rpl_semi_sync_slave_enabled = ON
```

### Vérification

```sql
-- Sur la source
SHOW STATUS LIKE 'Rpl_semi_sync_master_status';   -- attendu : ON
SHOW STATUS LIKE 'Rpl_semi_sync_master_clients';  -- attendu : ≥ 1

-- Sur le réplica
SHOW STATUS LIKE 'Rpl_semi_sync_slave_status';    -- attendu : ON
```

---

## 9. Quand utiliser la semi-synchrone ?

- **À privilégier** lorsqu'il faut **minimiser la perte de données** au *failover* (données financières, commandes…), avec des réplicas **proches** (faible latence réseau). Choisir alors `AFTER_SYNC` quand l'implémentation le permet, et combiner avec un **failover GTID** (13.8).
- **À éviter** sur des liens à **forte latence** (réplicas distants) ou lorsque le **débit maximal** prime : la semi-synchrone ajoute un aller-retour par transaction et **ne réduit pas le lag d'application** (13.7.2).
- **Ce n'est pas** une cohérence forte : pour une garantie absolue (zéro perte, multi-maître synchrone), c'est **Galera** qu'il faut envisager (chapitre 14).

---

## Idées clés à retenir

- La semi-synchrone est **native** (depuis 10.3.3) : `rpl_semi_sync_master_enabled` (source) et `rpl_semi_sync_slave_enabled` (réplica), activables **à chaud**.
- **Point d'attente** : `AFTER_COMMIT` (défaut) ou **`AFTER_SYNC`** (sans perte) ; **le binlog InnoDB (12.3) ne prend en charge aucune semi-synchrone** dans sa première implémentation (ERROR 4248) → conserver le binlog fichiers pour la semi-sync.
- Le **timeout** (`rpl_semi_sync_master_timeout`, 10 s) fait **basculer en asynchrone** : à **surveiller** via `Rpl_semi_sync_master_status`.
- MariaDB attend l'ACK d'**un** réplica (**non configurable**) : la variable MySQL `rpl_semi_sync_master_wait_for_slave_count` **n'existe pas** sous MariaDB.
- **12.x** : une source redémarrée ne tronque plus les transactions des réplicas → **les deux** rôles semi-synchrones peuvent rester actifs sans perte (topologies failover).
- À réserver aux réplicas **proches** et aux données **critiques** ; pour la cohérence forte, voir **Galera** (chapitre 14).

---

## Pour aller plus loin

- **13.1** — [Concepts de réplication : Asynchrone vs Semi-synchrone](01-concepts-replication.md) : la comparaison conceptuelle et les points d'attente.
- **13.7** — [Monitoring et troubleshooting](07-monitoring-troubleshooting.md) : surveiller l'état semi-synchrone.
- **13.8** — [Failover et switchover](08-failover-switchover.md) : réduire la perte au *failover*.
- **13.2.1** — [Configuration du Primary](02.1-configuration-primary.md) : binlog InnoDB (semi-synchrone indisponible).
- **Chapitre 14** — [Haute Disponibilité](../14-haute-disponibilite/README.md) : Galera Cluster pour la cohérence forte.

⏭️ [Optimistic ALTER TABLE pour réduction du lag](/13-replication/10-optimistic-alter-table.md)
