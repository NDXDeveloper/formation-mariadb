🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.6 — Solutions de failover automatique

> **Chapitre 14 — Haute Disponibilité** · Section 14.6  
> Version de référence : **MariaDB 12.3 LTS**

Le **failover automatique** est le mécanisme qui, après la **panne détectée** d'un serveur, bascule **sans intervention humaine** vers un serveur sain. Il est au cœur de la **deuxième couche** de HA (détection et orchestration, voir 14.1). Cette section en détaille l'anatomie, les solutions disponibles et les risques — en s'appuyant sur la détection vue en 14.1, le quorum de 14.3 et le routage de MaxScale (14.4).

---

## 1. Failover vs switchover (rappel)

Deux notions à ne pas confondre (voir 14.1) :

| | **Failover** | **Switchover** |
|---|--------------|----------------|
| Déclenchement | **Automatique**, réactif | **Planifié**, contrôlé |
| Contexte | Après une **panne** | Pour une **maintenance** |
| Perte de données possible | Oui (selon la réplication) | Non (bascule propre) |
| Enjeu | **Rapidité** (RTO) et sûreté | Zéro interruption perçue |

---

## 2. Où le failover automatique est crucial

Le besoin dépend fortement de la topologie :

- **Réplication primaire/réplica** : il n'y a **qu'un seul écrivain**. Si le primaire tombe, il faut **promouvoir un réplica**, ce qui exige une véritable **orchestration**. C'est ici que le failover automatique prend tout son sens.
- **Galera (multi-maître)** : le failover est **largement implicite** — aucun nœud n'est « le » primaire, et le cluster survit à la perte d'un nœud tant que le **quorum** tient (voir 14.3). La « bascule » se résume alors à **rediriger les écritures** vers un nœud survivant (voir 14.4 et §6).

Cette section concerne donc **principalement les topologies de réplication**.

---

## 3. Anatomie d'un failover automatique

Une solution de failover doit enchaîner plusieurs étapes, chacune source de risque si elle est mal gérée :

```
1. Détecter la panne du primaire          (health checks — gare aux faux positifs, 14.1)
2. Élire un nouveau primaire              (le réplica le plus à jour — GTID le plus avancé)
3. Promouvoir le réplica choisi
4. Reconfigurer les autres réplicas        (répliquer depuis le nouveau primaire)
5. Rediriger le trafic client             (proxy / VIP)
6. Isoler l'ancien primaire (fencing)      (empêcher un second primaire → split-brain)
7. Réintégrer l'ancien primaire            (en tant que réplica, à son retour)
```

La qualité d'une solution se juge à sa capacité à **toutes** les automatiser de façon **fiable**.

---

## 4. Le rôle clé du GTID

Le **GTID** (*Global Transaction Identifier*, voir 13.4) **simplifie radicalement** le failover. Grâce à lui, on peut **réorienter les réplicas** vers un nouveau primaire **sans calculer manuellement** des positions de binlog : l'ensemble de GTID du nouveau primaire permet aux autres de **reprendre au bon endroit**. Sans GTID, la promotion implique des calculs de coordonnées de binlog **fastidieux et sujets aux erreurs**. **Le GTID est donc un prérequis de fait** pour un failover automatique sûr.

---

## 5. Les solutions

### 5.1 MaxScale — MariaDB Monitor (`mariadbmon`)

C'est la solution **native** de l'écosystème MariaDB. Le *monitor* `mariadbmon` (voir 14.4) surveille une topologie primaire/réplicas et réalise **automatiquement** le **failover**, le **switchover** et la **réintégration** (*rejoin*) de l'ancien primaire. Son atout majeur : il est **intégré au routage** de MaxScale, si bien que la **redirection du trafic** (étape 5) est assurée **automatiquement** — détection, promotion et redirection sont unifiées dans un même outil.

> 🆕 MaxScale 25.01 a ajouté une **option « safe »** au failover automatique du MariaDB Monitor : le mode « safe » n'effectue pas de failover si une perte de données est certaine (détectée en comparant les **positions GTID** des serveurs), et une commande manuelle équivalente a été ajoutée. C'est un garde-fou direct contre le risque de perte de données (§7). À noter toutefois : le monitor n'interroge les GTID qu'**à chaque intervalle de surveillance** ; une écriture survenue **juste avant** un crash, non encore vue par le monitor, peut donc échapper à ce contrôle — le mode « safe » protège surtout des cas où les **réplicas accusent un retard** persistant, sans constituer une garantie absolue. MaxScale 25.01 peut en outre déclencher un failover lorsqu'un **test d'écriture** sur le primaire échoue (utile face aux blocages disque/moteur).

### 5.2 Orchestrator

**Orchestrator** est un outil **open source** réputé de gestion de topologies de réplication MySQL/MariaDB : **visualisation** de la topologie, **remaniement** (déplacement de réplicas) et **failover automatisé** avec détection des pannes. Très utilisé dans de grands environnements.

### 5.3 replication-manager

**replication-manager** (écosystème MariaDB/Signal18) assure le **monitoring** et l'**automatisation du failover/switchover** pour la réplication MariaDB (et des topologies multi-sources/Galera), avec une interface de pilotage.

### 5.4 Pacemaker / Corosync

Pour une approche **généraliste**, la pile **Pacemaker + Corosync** gère MariaDB comme une **ressource de cluster** : **Corosync** fournit la messagerie, l'appartenance et le **quorum** ; **Pacemaker** orchestre les ressources, le **fencing (STONITH)** et la bascule, souvent **couplé à une IP virtuelle** (voir 14.7). Puissant mais plus complexe à exploiter.

### 5.5 Cloud et Kubernetes

- Les **services managés** (bases de données *cloud*) intègrent généralement un **failover automatique** (par exemple un déploiement multi-zones).
- En **Kubernetes**, le **`mariadb-operator`** prend en charge la bascule au sein du cluster (voir 16.5).

---

## 6. Galera et le failover

En Galera, **il n'y a pas de primaire à promouvoir**. La résilience repose sur le **quorum** (14.3) : tant que la composante primaire tient, le cluster continue. Le « failover » consiste alors essentiellement à **rediriger les écritures** vers un nœud `Synced` survivant — un rôle assuré par le **proxy** (MaxScale et son *monitor* `galeramon`, voir 14.4). On ne parle donc pas de promotion, mais de **routage** et de **maintien du quorum**.

---

## 7. Risques et considérations

Le failover automatique est puissant mais délicat. Les principaux écueils :

- **Faux positifs et *flapping*** : une détection trop sensible (latence réseau passagère) déclenche des bascules **inutiles**, voire en cascade. La robustesse de la détection (seuils, confirmations) est primordiale (voir 14.1).
- **Split-brain** : après bascule, l'**ancien primaire doit être isolé** (*fencing*) et le nouveau doit être le **seul écrivain**, faute de quoi deux primaires divergent (voir 14.1 et 14.3).
- **Perte de données (RPO)** : en réplication **asynchrone**, le primaire défaillant peut emporter des transactions **non encore répliquées** ; les promouvoir un réplica en retard **perd** ces transactions. La réplication **semi-synchrone** (13.9) réduit ce risque, et le **mode « safe »** (§5.1) évite la bascule quand la perte est certaine.
- **Automatique vs assisté** : la bascule automatique est **rapide** (RTO faible) mais **plus risquée** (faux positifs) ; certaines organisations préfèrent **automatiser le switchover** mais **valider manuellement** le failover du primaire. C'est un arbitrage **rapidité/sûreté** à assumer explicitement.
- **Redirection** : la bascule n'a de valeur que si le trafic **suit** — d'où l'importance du proxy/VIP et du **rejeu de transaction** (voir 14.10).

---

## 8. Le failover n'est complet qu'avec la redirection

Une erreur fréquente est de penser qu'un failover s'arrête à la **promotion** d'un réplica. En réalité, sans **redirection du trafic**, les applications continueraient de viser l'ancien primaire. Un failover réussi combine donc **détection + promotion + redirection** :

- **MaxScale** **unifie** ces trois aspects (monitor + routage) ;
- les outils tiers (Orchestrator, Pacemaker) doivent être **couplés** à une **IP virtuelle** (14.7) ou à un proxy pour la redirection.

---

## À retenir

- Le **failover automatique** bascule sans intervention après une panne ; le **switchover** est sa version **planifiée**. Le besoin est surtout marqué en **réplication primaire/réplica** (un seul écrivain).
- Son **anatomie** : détecter → élire → promouvoir → reconfigurer les réplicas → **rediriger le trafic** → **isoler l'ancien primaire** → le réintégrer.
- Le **GTID** (13.4) est un **prérequis de fait** : il permet de réorienter les réplicas sans calcul de positions de binlog.
- Solutions : **MaxScale/`mariadbmon`** (native, intégrée au routage, avec un mode **« safe »** en 25.01), **Orchestrator**, **replication-manager**, **Pacemaker/Corosync**, ainsi que le **cloud managé** et le **`mariadb-operator`** (16.5).
- En **Galera**, le failover est **implicite** : pas de promotion, mais **quorum** (14.3) et **redirection** des écritures par le proxy (14.4).
- ⚠️ Risques clés : **faux positifs**, **split-brain** (fencing obligatoire), **perte de données** en asynchrone (atténuée par le semi-synchrone et le mode « safe ») ; arbitrer **automatique vs assisté**.
- Un failover n'est **complet qu'avec la redirection** du trafic (proxy/VIP + rejeu de transaction, 14.10).

La section suivante traite l'un des moyens d'assurer cette redirection : l'**IP virtuelle et keepalived**.

---

⬅️ [14.5.3 — Diff Router](05.3-diff-router.md) · [📑 Sommaire](../SOMMAIRE.md) · [14.7 — Virtual IP et keepalived ➡️](07-virtual-ip-keepalived.md)

⏭️ [Virtual IP et keepalived](/14-haute-disponibilite/07-virtual-ip-keepalived.md)
