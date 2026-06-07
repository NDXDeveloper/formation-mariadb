🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.2 — MariaDB Galera Cluster

> **Chapitre 14 — Haute Disponibilité** · Section 14.2  
> Version de référence : **MariaDB 12.3 LTS**

La section 14.1 a posé les concepts. Place maintenant à la technologie **phare de la haute disponibilité synchrone** sous MariaDB : **Galera Cluster**. C'est la solution de référence lorsqu'on cherche une **redondance des données sans perte** (RPO ≈ 0) et la possibilité de **lire et écrire sur n'importe quel nœud**. Cette page présente Galera dans son ensemble — son principe, son vocabulaire, ses prérequis, ses forces et ses limites — avant que les sous-sections n'en détaillent chaque rouage.

---

## 1. Qu'est-ce que Galera Cluster ?

**Galera Cluster** est une solution de **réplication virtuellement synchrone, multi-maître**, intégrée à MariaDB. Tous les nœuds sont **égaux** : chacun accepte aussi bien les **lectures que les écritures**, et tous détiennent en permanence le même jeu de données.

```
        ┌──────────────── Galera Cluster (gcomm://) ────────────────┐
        │                  communication de groupe (full mesh)      │
   ┌────┴────┐            ┌─────────┐            ┌─────────┐
   │ Nœud 1  │◄──────────►│ Nœud 2  │◄──────────►│ Nœud 3  │
   │  R / W  │◄──────────►│  R / W  │◄──────────►│  R / W  │
   └─────────┘            └─────────┘            └─────────┘
        ▲                      ▲                      ▲
        └─── lectures ET écritures possibles sur chaque nœud ────┘
```

Techniquement, Galera repose sur :

- la **bibliothèque de réplication Galera** (développée par Codership), un *provider* externe chargé par le serveur (`libgalera_smm.so`) ;
- l'**API wsrep** (*Write-Set Replication*), l'interface standardisée par laquelle MariaDB délègue la réplication à ce provider ;
- un **système de communication de groupe** (GCS) qui diffuse les messages entre nœuds et maintient la composition du cluster.

> 🆕 **Nouveauté 12.3 — packaging séparé.** Le support Galera est désormais fourni par un **paquet dédié, `mariadb-server-galera`**, et n'est plus une dépendance du paquet serveur. L'installation de Galera devient donc un choix explicite. Cette évolution est détaillée en **14.2.5** et a des conséquences concrètes sur les images Docker et l'operator Kubernetes (voir 16.3.1 et 16.5.2).

---

## 2. Pourquoi « virtuellement synchrone » ?

C'est la clé de voûte de Galera, à mi-chemin entre synchrone et asynchrone :

- **Au COMMIT**, la transaction produit un *write-set* (l'ensemble de ses modifications + des clés de certification). Ce write-set est **diffusé à tous les nœuds** et reçoit un **numéro d'ordre global**.
- Chaque nœud exécute alors une **certification** : un contrôle de conflit **déterministe** sur ce flux totalement ordonné. Comme tous les nœuds appliquent la même logique au même ordre, ils prennent **tous la même décision**.
- Si la certification réussit, le nœud d'origine valide et **rend la main au client immédiatement** ; les autres nœuds **appliquent le write-set en différé** (de façon asynchrone), mais l'issue est déjà garantie identique partout.

Ce mécanisme évite d'attendre l'application physique sur chaque nœud (d'où « virtuellement » synchrone) tout en garantissant la **cohérence** sans risque de divergence. La certification proprement dite fait l'objet de la section **14.2.2**, et l'architecture d'ensemble de la section **14.2.1**.

---

## 3. Galera face à la réplication classique

Galera ne remplace pas la réplication du chapitre 13 ; il répond à un besoin différent.

| Critère | **Galera Cluster** | **Réplication classique (ch. 13)** |
|---------|--------------------|-----------------------------------|
| Synchronisme | Virtuellement synchrone | Asynchrone (ou semi-synchrone) |
| Topologie | Multi-maître (tous R/W) | Généralement un seul écrivain (source) |
| RPO | ≈ 0 | > 0 (perte du *lag* possible) |
| Bascule (*failover*) | Implicite : aucun nœud n'est « le » primaire | À orchestrer (promotion d'un réplica) |
| Adhésion des nœuds | Automatique (provisioning SST/IST) | Manuelle |
| Sensibilité à la latence réseau | Forte | Faible |
| Cas d'usage type | HA sans perte, montée en charge des lectures | Réplicas de lecture, distribution géographique souple |

Les deux approches se **combinent** fréquemment : par exemple une réplication asynchrone entre deux clusters Galera géo-distribués (facilitée en 12.3 par la réplication parallèle inter-clusters, voir 13.11).

---

## 4. Vocabulaire Galera essentiel

Ces termes reviennent dans toutes les sous-sections ; mieux vaut les avoir en tête dès maintenant :

| Terme | Signification |
|-------|---------------|
| **Write-set** | Ensemble des modifications d'une transaction (lignes + clés), unité de réplication. |
| **Certification** | Contrôle de conflit déterministe appliqué à chaque write-set (voir 14.2.2). |
| **wsrep** | *Write-Set Replication* : l'API et le protocole de réplication synchrone. |
| **Provider (wsrep_provider)** | La bibliothèque Galera chargée par MariaDB (`libgalera_smm.so`). |
| **GCS** | *Group Communication System* : couche de messagerie entre nœuds. |
| **GCache** | Tampon circulaire stockant les write-sets récents, utilisé pour l'IST. |
| **SST / IST** | *State Snapshot Transfer* (copie complète) / *Incremental State Transfer* (différentiel) — voir 14.2.4. |
| **Flow control** | Régulation : un nœud en retard peut **mettre le cluster en pause** le temps de rattraper. |
| **Primary Component** | Sous-groupe disposant du quorum, seul autorisé à servir (voir 14.3). |
| **BF-abort** (*brute force abort*) | Annulation forcée d'une transaction locale en conflit avec un write-set entrant. |

---

## 5. Prérequis et contraintes

Galera impose un certain nombre de conditions techniques, valables pour **toute** la section. Les ignorer conduit à des comportements imprévisibles.

| Prérequis | Détail |
|-----------|--------|
| **Moteur InnoDB** | Seules les tables InnoDB sont répliquées par Galera. Les autres moteurs ne le sont pas. |
| **Clé primaire sur chaque table** | Indispensable à la certification et à l'application des lignes. Une table sans `PRIMARY KEY` peut diverger entre nœuds. |
| **`binlog_format = ROW`** | Le format ligne est requis pour la réplication des write-sets. |
| **Nombre de nœuds impair** | Recommandé (3, 5…) pour préserver le quorum et éviter le split-brain (voir 14.3). |
| **Réseau à faible latence** | Le synchronisme rend Galera sensible au RTT ; les déploiements WAN demandent une attention particulière. |
| **Paquet `mariadb-server-galera`** 🆕 | À installer explicitement en 12.3 (voir 14.2.5). |

### Limitations connues

Quelques comportements diffèrent d'un serveur MariaDB autonome :

- les **verrous de table** (`LOCK TABLES`) et les **verrous utilisateur** (`GET_LOCK()`, `RELEASE_LOCK()`) **ne s'appliquent pas à l'échelle du cluster** ;
- les **très grosses transactions** sont déconseillées (un write-set volumineux pèse sur tout le cluster) ;
- les **points chauds en écriture** (plusieurs nœuds modifiant les mêmes lignes) génèrent des **échecs de certification**, signalés au client comme des erreurs de type *deadlock* (à rejouer côté application) ;
- le **query cache** (de toute façon déprécié, voir 15.3) n'est pas compatible.

---

## 6. Quand utiliser Galera (et quand l'éviter)

| ✅ Galera est bien adapté quand… | ❌ Galera est moins adapté quand… |
|----------------------------------|-----------------------------------|
| On vise une HA **sans perte de données** (RPO ≈ 0). | La charge est **très intensive en écriture** sur les **mêmes lignes** (conflits de certification). |
| On veut **répartir les lectures** sur plusieurs nœuds. | Les transactions sont **très volumineuses** ou très longues. |
| Les nœuds sont **proches** (même datacenter / zone). | Le réseau entre nœuds est **à forte latence** (WAN). |
| On souhaite une **bascule implicite**, sans promotion. | Le schéma comporte des **tables sans clé primaire** difficiles à corriger. |
| La charge est de type **OLTP** à contention modérée. | On a besoin de verrous de table cluster-wide ou de transactions XA étendues. |

---

## 7. Nouveautés 12.3 pour Galera

La 12.3 LTS apporte deux évolutions notables à l'écosystème Galera :

| Nouveauté | Apport | Détail |
|-----------|--------|--------|
| **Packaging séparé** (`mariadb-server-galera`) 🆕 | Galera n'est plus une dépendance du serveur ; installation explicite, image serveur allégée | **14.2.5** |
| **Retry des write-sets** (`wsrep_applier_retry_count`) 🆕 | L'*applier* peut **rejouer** un write-set en cas de conflit transitoire, au lieu d'abandonner aussitôt — robustesse accrue | **14.2.6** |

---

## 8. Plan de la section

La section progresse du **principe** vers la **mise en œuvre**, puis vers les **nouveautés** :

1. **[14.2.1 — Architecture synchrone multi-master](02.1-architecture-synchrone.md)**
   Le fonctionnement interne : flux d'une transaction, ordre global, application des write-sets, *flow control*.

2. **[14.2.2 — Certification-based replication](02.2-certification-based.md)**
   Le cœur de la cohérence : comment les conflits sont détectés de façon déterministe (« le premier validé l'emporte »).

3. **[14.2.3 — Configuration et déploiement](02.3-configuration-deploiement.md)**
   Les variables `wsrep_*`, l'amorçage du cluster (*bootstrap*), l'adresse `gcomm://`, le premier nœud.

4. **[14.2.4 — State transfers (SST, IST)](02.4-state-transfers.md)**
   Comment un nœud se synchronise en rejoignant le cluster : transfert complet (SST) ou incrémental (IST) via le GCache.

5. **[14.2.5 — Packaging Galera en 12.3 : paquet `mariadb-server-galera` séparé](02.5-packaging-galera.md)** 🆕
   La dépendance retirée du serveur et ses conséquences sur l'installation et les conteneurs.

6. **[14.2.6 — Retry des write-sets (`wsrep_applier_retry_count`)](02.6-retry-write-sets.md)** 🆕
   Le nouveau mécanisme de réessai côté *applier* pour absorber les conflits transitoires.

---

## À retenir

- **Galera Cluster** est une solution de réplication **virtuellement synchrone, multi-maître** : on lit et on écrit sur **n'importe quel nœud**, sans perte de données (RPO ≈ 0).
- Le « virtuellement synchrone » repose sur la **certification déterministe** des *write-sets* au COMMIT : la cohérence est garantie sans attendre l'application physique sur chaque nœud.
- Galera **complète** la réplication classique du chapitre 13 plutôt qu'il ne la remplace ; les deux se combinent souvent (clusters géo-distribués).
- Il impose des **prérequis stricts** : InnoDB, clé primaire sur chaque table, `binlog_format=ROW`, nombre de nœuds impair, réseau à faible latence.
- Il est idéal pour l'**OLTP à contention modérée** mais souffre des **points chauds en écriture**, des **grosses transactions** et de la **latence WAN**.
- En 12.3, Galera est **packagé séparément** (`mariadb-server-galera`) et gagne le **retry des write-sets** (`wsrep_applier_retry_count`).

La sous-section suivante ouvre le capot pour détailler ce fonctionnement multi-maître synchrone.

---

⬅️ [14.1 — Concepts](01-architectures-ha-concepts.md) · [📑 Sommaire](../SOMMAIRE.md) · [14.2.1 — Architecture synchrone multi-master ➡️](02.1-architecture-synchrone.md)

⏭️ [Architecture synchrone multi-master](/14-haute-disponibilite/02.1-architecture-synchrone.md)
