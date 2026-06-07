🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.9 — Alternatives : ProxySQL et HAProxy

> **Chapitre 14 — Haute Disponibilité** · Section 14.9  
> Version de référence : **MariaDB 12.3 LTS**

MaxScale (14.4–14.5) est le proxy **natif** de MariaDB, mais ce n'est pas la seule option pour la **couche de routage et de répartition**. Deux alternatives sont très répandues, à des niveaux différents : **HAProxy**, répartiteur de charge **généraliste** (niveau réseau), et **ProxySQL**, proxy **conscient du SQL** spécifique à MySQL/MariaDB. Cette section les présente et les compare à MaxScale.

---

## 1. Pourquoi des alternatives ?

MaxScale est puissant et intégré, mais il relève de la **licence BSL** (*source-available*, voir 14.4). Certaines équipes préfèrent une solution **open source** (GPL), ou réutilisent simplement un outil qu'elles **exploitent déjà** ailleurs. Les deux alternatives se situent à des **niveaux distincts** : HAProxy au **niveau connexion (TCP)**, ProxySQL au **niveau SQL**.

---

## 2. HAProxy — répartiteur de charge généraliste (niveau 4)

**HAProxy** est un répartiteur de charge / proxy inverse **rapide, fiable et omniprésent**. Pour les bases de données, il opère au **niveau 4 (TCP)** : il distribue des **connexions**, sans comprendre le protocole MySQL ni le SQL.

**Usage avec MariaDB :**
- répartir les **connexions** sur plusieurs serveurs (nœuds Galera `Synced`, réplicas) ;
- diriger lectures et écritures via des **écouteurs/backends séparés** (puisqu'il ne peut pas le faire par instruction).

**Vérifications de santé :** HAProxy peut effectuer un test TCP, un test MySQL, ou un test HTTP via le script **`clustercheck`** pour Galera — un script qui expose l'état `wsrep` du nœud en HTTP, afin que HAProxy ne route **que vers les nœuds `Synced`**.

> ⚠️ **HAProxy n'est pas conscient du SQL.** Il ne peut **pas** faire de **partage lecture/écriture par instruction** : il route des connexions entières au niveau TCP. Le R/W split doit alors être assuré par l'**application** (ou via des ports/backends distincts).

**Atouts :** **extrêmement rapide**, **simple**, éprouvé, faible surcoût, **généraliste** (pas réservé aux bases) et **open source** (GPL).

---

## 3. ProxySQL — proxy conscient du SQL

**ProxySQL** est un proxy **haute performance conscient du SQL**, dédié à MySQL/MariaDB et **open source** (GPL). Il **comprend** le protocole et les requêtes.

**Capacités :**
- **règles de requêtes** (expressions régulières → *hostgroups*), **partage lecture/écriture**, **cache de requêtes**, **réécriture**, **multiplexage de connexions**, limitation de débit ;
- organisation des serveurs en **hostgroups** (écrivains / lecteurs) ; **prise en charge native de Galera** (suivi des états `wsrep` pour aiguiller vers les nœuds sains) ;
- **configuration à chaud** via une **interface SQL d'administration** (on configure ProxySQL en exécutant du SQL sur ses propres tables de configuration), **sans redémarrage**.

**Modèle de déploiement notable — le *sidecar* :** ProxySQL peut tourner **sur chaque serveur applicatif** (en local) ; cela **élimine le SPOF** d'un proxy central et **réduit la latence**.

> ProxySQL **réachemine** le trafic selon l'**état** des serveurs (par exemple le drapeau `read_only` lorsqu'un réplica est promu), mais il **ne promeut pas** lui-même un réplica : la **promotion** reste l'affaire d'un orchestrateur (voir 14.6).

---

## 4. Comparaison avec MaxScale

| Critère | **MaxScale** | **ProxySQL** | **HAProxy** |
|---------|--------------|--------------|-------------|
| Niveau | Conscient du SQL | Conscient du SQL | **TCP (niveau 4)** |
| Partage lecture/écriture | Oui (`readwritesplit`) | Oui (règles + hostgroups) | **Non** (par connexion) |
| Orchestration du failover | **Oui** (`mariadbmon`) | Non (réachemine selon l'état, sans promotion) | Non |
| Cache de requêtes | Oui | Oui | Non |
| Configuration | `maxscale.cnf` / `maxctrl` / API | **Interface SQL d'admin (à chaud)** | Fichier `haproxy.cfg` |
| Licence | **BSL** (*source-available*) | **GPL** (open source) | **GPL** (open source) |
| Spécificité | Natif MariaDB, outillage récent (14.5) | Riche pour MySQL/MariaDB, *sidecar* | Généraliste, ultra-rapide |

---

## 5. Quand utiliser quoi ?

| Choisir… | Quand… |
|----------|--------|
| **HAProxy** | On veut une **répartition de connexions simple** (par ex. sur les nœuds Galera `Synced` via `clustercheck`), le R/W split est géré par l'application (ou inutile), et l'on privilégie **vitesse, simplicité** et un outil déjà exploité. |
| **ProxySQL** | On veut un **partage lecture/écriture conscient du SQL**, des **règles de requêtes**, du **cache**, dans une solution **open source** — souvent en environnement d'héritage MySQL, ou en *sidecar*. |
| **MaxScale** | On veut la solution **native MariaDB intégrée** : routage **+ orchestration du failover** (`mariadbmon`) **+** outillage récent (Capture/Replay, Diff, 14.5). |

Ces choix ne sont pas exclusifs : ils peuvent **coexister** selon les besoins (par exemple ProxySQL pour le R/W split et HAProxy en façade, ou simplement keepalived/VIP pour la HA du proxy, voir 14.7).

---

## 6. La haute disponibilité du proxy lui-même

Rappel de 14.4 et 14.7 : **tout proxy est un SPOF potentiel**. Comme MaxScale, **HAProxy et ProxySQL doivent être rendus hautement disponibles** :

- en déployant **plusieurs instances** derrière une **VIP/keepalived** (voir 14.7) ;
- pour **ProxySQL**, le modèle **sidecar** (une instance par serveur applicatif) **évite d'emblée** un proxy central et donc ce SPOF.

---

## 7. Considérations Galera (transversales aux trois proxys)

Quel que soit le proxy, deux exigences reviennent avec Galera (rappels de 14.2.2 et 14.4) :

- **ne router que vers les nœuds `Synced`** : HAProxy via **`clustercheck`** (HTTP), ProxySQL via sa **prise en charge native de Galera**, MaxScale via **`galeramon`** ;
- **diriger les écritures vers un seul nœud** pour **limiter les conflits de certification** (14.2.2) : backend écrivain unique côté HAProxy, *hostgroup* écrivain côté ProxySQL, `readwritesplit`/`galeramon` côté MaxScale.

---

## À retenir

- Au-delà de MaxScale (natif, **BSL**), deux alternatives **open source** (GPL) répandues : **HAProxy** (niveau **TCP**) et **ProxySQL** (conscient du **SQL**).
- **HAProxy** est **ultra-rapide et simple** mais **non conscient du SQL** : pas de R/W split par instruction (à gérer côté application) ; pour Galera, il route vers les nœuds `Synced` via **`clustercheck`**.
- **ProxySQL** est **conscient du SQL** : **R/W split**, **règles de requêtes**, **cache**, **hostgroups**, **support natif Galera**, **configuration à chaud** par interface SQL, et déploiement en **sidecar** (sans SPOF central). Il **réachemine** selon l'état mais **ne promeut pas**.
- **MaxScale** se distingue par l'**orchestration du failover** (`mariadbmon`) et son **outillage récent** (14.5).
- ⚠️ Comme tout proxy, HAProxy et ProxySQL **doivent être rendus HA** (multi-instances + VIP/keepalived, 14.7 ; ou ProxySQL en *sidecar*).
- Transversal Galera : **router vers les `Synced`** et **écrire sur un seul nœud** (limiter les conflits de certification, 14.2.2) — chaque proxy à sa manière.

La section suivante revient à MaxScale pour une fonction qui rend la bascule transparente : le **Transaction Replay et la Connection Migration**.

---

⬅️ [14.8 — Stratégies de récupération après incident](08-strategies-recuperation.md) · [📑 Sommaire](../SOMMAIRE.md) · [14.10 — Transaction Replay et Connection Migration ➡️](10-transaction-replay-connection-migration.md)

⏭️ [Transaction Replay et Connection Migration](/14-haute-disponibilite/10-transaction-replay-connection-migration.md)
