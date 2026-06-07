🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.4 — MaxScale

> **Chapitre 14 — Haute Disponibilité** · Section 14.4  
> Version de référence : **MariaDB 12.3 LTS**

Les sections précédentes ont traité la **redondance des données** (Galera, 14.2) et la **prévention du split-brain** (quorum, 14.3) — soit les deux premières couches du modèle de HA présenté en 14.1. Place maintenant à la **troisième couche** : le **routage et le point d'accès stable**. **MaxScale** est le proxy de base de données de MariaDB ; il se place entre les applications et le cluster pour leur présenter un **point d'entrée unique** et router intelligemment les requêtes. Cette page présente MaxScale dans son ensemble ; ses fonctions précises sont détaillées dans les sous-sections, et ses nouveautés en 14.5.

---

## 1. Qu'est-ce que MaxScale ?

**MaxScale** est un **proxy de base de données avancé** développé par MariaDB. Sa particularité : il est **conscient du protocole et du SQL** (*database-aware*). Contrairement à un simple répartiteur de charge réseau (niveau TCP), MaxScale **comprend** les requêtes qui le traversent. Il peut donc prendre des décisions de routage fondées sur le **contenu** : distinguer une lecture d'une écriture, diriger vers le bon serveur, filtrer une requête dangereuse, etc.

Il se place **entre les applications et les serveurs MariaDB** (cluster Galera ou topologie primaire/réplicas).

---

## 2. Le problème : un point d'accès stable et intelligent

Rappel de 14.1 : pour être réellement disponible, un service doit offrir aux applications un **point d'accès qui ne change pas**, quelle que soit la topologie sous-jacente. Sans proxy, c'est à l'**application** d'assumer une logique complexe :

- savoir **quel nœud** accepte les écritures ;
- **répartir** les lectures sur les autres nœuds ;
- **se reconnecter** au bon serveur après une bascule.

MaxScale **absorbe cette complexité** : il expose un **point d'entrée unique** derrière lequel il gère le routage, la santé des serveurs et la bascule, de façon **transparente** pour l'application.

---

## 3. Position dans l'architecture

```
   ┌──────────────────── Applications / clients ─────────────────────┐
   └────────────────────────────┬────────────────────────────────────┘
                                │ point d'accès unique et stable
                    ┌───────────▼──────────────┐
                    │        MaxScale          │
                    │  Listeners → Services    │
                    │   (Routeurs + Filtres)   │
                    │  Monitors (santé/rôles)  │
                    └───────────┬──────────────┘
               ┌────────────────┼────────────────┐
         ┌─────▼─────┐    ┌─────▼─────┐    ┌─────▼──────┐
         │ Serveur 1 │    │ Serveur 2 │    │ Serveur 3  │
         │    cluster Galera ou primaire/réplicas       │
         └───────────┘    └───────────┘    └────────────┘
```

> ⚠️ **MaxScale ne doit pas devenir un SPOF.** En s'intercalant devant le cluster, un MaxScale **unique** redeviendrait le point unique de défaillance que la HA cherche à éliminer (voir 14.1). En production, on déploie donc **plusieurs instances** de MaxScale, généralement derrière une **IP virtuelle / keepalived** (voir 14.7).

---

## 4. Les briques de MaxScale

MaxScale s'articule autour de quelques composants qui correspondent aux sections de son fichier de configuration (`maxscale.cnf`) :

| Brique | Rôle |
|--------|------|
| **Servers** | Les serveurs MariaDB du *backend* que MaxScale connaît. |
| **Monitors** | Surveillent l'**état**, la **santé** et le **rôle** des serveurs (`galeramon` pour Galera, `mariadbmon` pour la réplication) ; ils sont la clé du **failover automatique** (voir 14.6). |
| **Services** | Entité **logique** reliant un **routeur**, des serveurs et des *listeners* ; définit comment les requêtes sont traitées. |
| **Routers** | Le module qui **décide du routage** : `readwritesplit` (R/W split), `readconnroute` (par connexion), `schemarouter` (sharding par schéma), `binlogrouter` (relais de binlog)… |
| **Listeners** | Le **point d'écoute réseau** (port) où les clients se connectent à un service. |
| **Filters** | Modules **optionnels** insérés dans le pipeline de requêtes : pare-feu, journalisation, réécriture, cache… |

L'administration se fait via la CLI **`maxctrl`**, une **API REST** et une interface graphique.

---

## 5. Ce que MaxScale apporte (panorama)

MaxScale couvre un large éventail de fonctions, dont les principales sont détaillées dans ce chapitre :

- la **répartition de charge** (*load balancing*) — voir 14.4.1 ;
- le **partage lecture/écriture** (*read/write split*) — voir 14.4.2 ;
- le **routage de requêtes** (*query routing*) — voir 14.4.3 ;
- le **pare-feu de base de données** (*database firewall*) — voir 14.4.4 ;
- le **failover et le switchover automatiques** via les *monitors* — voir 14.6 ;
- la **migration de connexion** et le **rejeu de transaction** (*transaction replay*) pour rendre la bascule transparente — voir 14.10 ;
- la **capture/rejeu de charge** et le **Diff Router** (nouveautés) — voir 14.5 ;
- le **multiplexage / pooling de connexions**.

---

## 6. Bénéfices

- **Point d'accès unique et stable** : l'application est découplée de la topologie.
- **Bascule transparente** : MaxScale réachemine le trafic après un *failover* sans réécriture applicative.
- **Montée en charge des lectures** : via le partage R/W (14.4.2).
- **Sécurité** : filtrage et pare-feu (14.4.4).
- **Observabilité** : visibilité sur l'état des serveurs et le trafic.

---

## 7. Licence et alternatives

MaxScale n'est pas distribué sous la même licence que le serveur MariaDB : il relève de la **Business Source License (BSL)**, un modèle *source-available* dont les conditions d'usage en production diffèrent d'une licence open source classique. Avant un déploiement, il convient de **vérifier les conditions de licence en vigueur** pour son cas d'usage.

Il existe par ailleurs des **alternatives** (ProxySQL, HAProxy) répondant à des besoins voisins de répartition et de routage ; elles sont comparées en **14.9**.

---

## 8. Plan de la section

Les sous-sections détaillent les fonctions cœur de MaxScale :

1. **[14.4.1 — Load Balancing](04.1-load-balancing.md)** : répartir la charge entre les serveurs du backend.
2. **[14.4.2 — Read/Write Split](04.2-read-write-split.md)** : diriger automatiquement écritures et lectures vers les bons nœuds.
3. **[14.4.3 — Query Routing](04.3-query-routing.md)** : router les requêtes selon des règles fines.
4. **[14.4.4 — Database Firewall](04.4-database-firewall.md)** : filtrer et bloquer les requêtes indésirables.

La section **14.5** présentera ensuite les **fonctionnalités récentes** (Workload Capture/Replay, Diff Router).

---

## À retenir

- **MaxScale** est le **proxy de base de données** de MariaDB ; il constitue la **couche de routage et de point d'accès stable** (3ᵉ couche de HA, voir 14.1).
- Il est **conscient du SQL** : il route selon le **contenu** des requêtes (lecture/écriture, règles, filtrage), pas seulement au niveau TCP.
- Il **absorbe la complexité** côté application : découverte du primaire, répartition des lectures, reconnexion après bascule.
- Ses briques sont les **Servers**, **Monitors**, **Services**, **Routers**, **Listeners** et **Filters** ; les *monitors* sont la clé du **failover** (14.6).
- ⚠️ Un MaxScale **unique** serait un **SPOF** : on en déploie **plusieurs** derrière une **VIP/keepalived** (14.7).
- MaxScale relève de la **BSL** (vérifier les conditions de licence) ; alternatives en **14.9**.

La sous-section suivante ouvre le détail des fonctions cœur, en commençant par la **répartition de charge**.

---

⬅️ [14.3 — Split-brain et quorum](03-split-brain-quorum.md) · [📑 Sommaire](../SOMMAIRE.md) · [14.4.1 — Load Balancing ➡️](04.1-load-balancing.md)

⏭️ [Load Balancing](/14-haute-disponibilite/04.1-load-balancing.md)
