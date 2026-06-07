🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.7 — Virtual IP et keepalived

> **Chapitre 14 — Haute Disponibilité** · Section 14.7  
> Version de référence : **MariaDB 12.3 LTS**

Une bascule (14.6) n'a de valeur que si le **trafic suit**. L'**IP virtuelle** (*Virtual IP*, VIP) est l'un des moyens les plus répandus d'assurer cette **redirection** en offrant aux clients une **adresse stable** qui se **déplace** vers un hôte sain en cas de panne. Sous Linux, c'est le démon **keepalived** (via le protocole VRRP) qui gère cette IP flottante. Cette section explique le mécanisme, ses cas d'usage en HA MariaDB et ses limites — notamment en environnement *cloud*.

---

## 1. Qu'est-ce qu'une IP virtuelle (VIP) ?

Une **IP virtuelle** est une adresse IP **non liée en permanence** à un hôte physique : elle peut **migrer** d'une machine à une autre. Les clients se connectent **à la VIP**, et c'est l'hôte qui la **détient** à un instant donné qui reçoit le trafic. En cas de panne, la VIP **bascule** vers un hôte sain — **de façon transparente** pour les clients, qui continuent d'utiliser la **même adresse**.

```
              VIP : 10.0.0.100   (adresse stable, vue par les clients)
                         │
        ┌────────────────┴──────────────────┐
   ┌────▼─────┐                       ┌─────▼────┐
   │ Nœud A   │  ◄──── VRRP ────►     │ Nœud B   │
   │ MASTER   │     (heartbeats)      │ BACKUP   │
   │ détient  │                       │ en       │
   │ la VIP   │                       │ attente  │
   └──────────┘                       └──────────┘

   Panne du nœud A → la VIP migre vers B, qui devient MASTER
```

C'est la concrétisation du **point d'accès stable** évoqué dès 14.1.

---

## 2. Pourquoi une VIP ?

L'intérêt est de **découpler les clients des hôtes physiques** : ils ne visent jamais une machine précise, mais toujours la VIP. On l'utilise typiquement pour **mettre en façade** :

- une **paire de MaxScale** — afin que MaxScale lui-même **ne soit pas un SPOF** (rappel de 14.4) : si l'instance active tombe, la VIP passe à l'instance de secours ;
- un couple **primaire/secours** sans proxy ;
- une paire de proxys **HAProxy/ProxySQL** (voir 14.9).

---

## 3. VRRP et keepalived

- **VRRP** (*Virtual Router Redundancy Protocol*) est le **protocole** qui gère l'IP flottante. Les nœuds forment un **groupe VRRP** : l'un est **MASTER** (il détient la VIP), les autres sont **BACKUP**. Ils échangent des messages d'**annonce** (*heartbeats*) périodiques. Si le MASTER cesse d'émettre, un BACKUP **prend la main** selon sa **priorité**.
- **keepalived** est le **démon** Linux qui implémente VRRP (et propose aussi des *health checks*). C'est l'outil de référence pour gérer une VIP.

Les notions clés de configuration : `vrrp_instance`, `virtual_router_id` (identique sur tout le groupe), `priority` (la plus élevée devient MASTER), `state` (MASTER/BACKUP), `advert_int` (intervalle d'annonce), `virtual_ipaddress` et `track_script` (les vérifications de santé).

---

## 4. Comment se fait la bascule

1. Le **MASTER** émet régulièrement des **annonces VRRP**.
2. Si le **MASTER disparaît** (ou si sa vérification de santé échoue, §5), les BACKUP **cessent de recevoir** les annonces.
3. Le BACKUP de **plus haute priorité** se promeut **MASTER** et **prend la VIP**.
4. Il émet un **ARP gratuit** (*gratuitous ARP*) pour **mettre à jour** les caches ARP du réseau (commutateurs, hôtes).
5. Le trafic destiné à la VIP arrive désormais sur le **nouveau MASTER**.

> ℹ️ **Préemption.** Par défaut, un nœud de plus haute priorité **reprend** le rôle de MASTER à son retour (préemption). Ce comportement est **configurable** (`nopreempt`) pour éviter des bascules en va-et-vient.

Exemple illustratif de configuration keepalived (référence, la syntaxe exacte pouvant varier) :

```
vrrp_script chk_maxscale {
    script "/usr/bin/killall -0 maxscale"   # le service est-il vivant ?
    interval 2
    weight -20                              # abaisse la priorité en cas d'échec
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51        # identique sur tous les nœuds du groupe
    priority 110                # la plus élevée devient MASTER
    advert_int 1
    virtual_ipaddress {
        10.0.0.100/24
    }
    track_script {
        chk_maxscale
    }
}
```

---

## 5. Les vérifications de santé (`track_script`) sont essentielles

Point souvent négligé : **sans vérification du service**, la VIP ne bascule **que** si l'hôte ou keepalived **meurt** — **pas** si le **service** (MaxScale, MariaDB) tombe alors que la machine reste allumée.

La parade est le **`track_script`** : keepalived exécute périodiquement un script qui **teste le vrai service** (par exemple : MaxScale écoute-t-il ? MariaDB répond-il ?). En cas d'échec, la **priorité du nœud baisse** (ou il abandonne le rôle MASTER), ce qui **déclenche la migration** de la VIP. **Vérifier le service, pas seulement l'hôte**, est donc indispensable.

---

## 6. Cas d'usage en HA MariaDB

- **Façade pour MaxScale** : on déploie **deux instances ou plus** de MaxScale et keepalived leur attribue une VIP. Si l'instance active tombe, la VIP passe à l'autre — MaxScale devient ainsi **hautement disponible**. (MaxScale dispose par ailleurs d'une **surveillance coopérative** pour qu'une seule instance agisse activement sur le cluster.)
- **Façade pour un couple primaire/secours** dans les déploiements **sans proxy**.
- **Façade pour des proxys** HAProxy/ProxySQL (voir 14.9).

---

## 7. Limites et points d'attention

- ⚠️ **Même réseau de niveau 2.** VRRP et l'ARP gratuit fonctionnent au sein d'un **même segment réseau (couche 2)**. **À travers des sous-réseaux ou des datacenters différents**, le VRRP « classique » **ne fonctionne pas** ; il faut d'autres mécanismes.
- ⚠️ **Cloud.** La plupart des fournisseurs *cloud* **interdisent** la manipulation arbitraire d'IP/ARP : on utilise alors le **répartiteur de charge** du fournisseur ou son **API d'IP flottante** plutôt que keepalived.
- **Split-brain au niveau de la VIP** : si une partition réseau **isole les nœuds entre eux**, plusieurs pourraient se croire MASTER et **revendiquer la VIP**. Une bonne configuration de priorités atténue le risque, mais cela **ne remplace pas** le quorum et le *fencing* **au niveau des données** (voir 14.3).
- **Bascule de la VIP ≠ bascule des données.** Déplacer la VIP **ne promeut pas** un réplica : la VIP **redirige** seulement les connexions. La **promotion** (14.6) et le **quorum** (14.3) sont des problèmes **distincts**, à traiter par ailleurs.

---

## 8. VIP, failover et proxy : qui fait quoi ?

Il est essentiel de ne pas confondre les rôles des différents composants de HA :

| Question | Composant | Section |
|----------|-----------|---------|
| Qui détecte la panne et **promeut** un nouveau primaire ? | Solution de failover (`mariadbmon`, Orchestrator…) | 14.6 |
| Qui garantit qu'un **seul** côté écrit (anti-split-brain) ? | **Quorum / fencing** | 14.3 |
| Qui **route** les requêtes (R/W split…) ? | Proxy (MaxScale) | 14.4 |
| Qui offre une **adresse stable** et la **déplace** ? | **VIP / keepalived** | **14.7** |

La VIP est donc la brique de **redirection** (l'étape 5 du failover, voir 14.6). Le couple **MaxScale + keepalived** est une combinaison courante : keepalived rend les **proxys** hautement disponibles, et MaxScale gère le **routage** et la **détection** côté base de données.

---

## À retenir

- Une **IP virtuelle (VIP)** est une adresse **flottante** que les clients ciblent en permanence ; elle **migre** vers un hôte sain en cas de panne, offrant un **point d'accès stable**.
- **keepalived** gère la VIP via **VRRP** : un nœud **MASTER** détient l'adresse, les **BACKUP** prennent le relais selon leur **priorité**, avec mise à jour réseau par **ARP gratuit**.
- Les **`track_script`** sont **essentiels** : sans vérification du **service**, la VIP ne bascule que sur mort de l'hôte/keepalived, pas sur panne applicative.
- Usage phare : **rendre MaxScale hautement disponible** (façade d'une paire de proxys), éliminant le SPOF de 14.4.
- ⚠️ Limites : **même segment L2** requis (problématique en multi-DC), **interdit/à remplacer en cloud** (répartiteur ou IP flottante du fournisseur), et risque de **split-brain au niveau de la VIP** — à combiner avec le **quorum/fencing** des données (14.3).
- **Déplacer la VIP ≠ promouvoir un réplica** : la VIP **redirige** (étape 5 du failover) ; la **promotion** (14.6) et le **quorum** (14.3) restent des problèmes distincts.

La section suivante traite la remise en service après incident : les **stratégies de récupération**.

---

⬅️ [14.6 — Solutions de failover automatique](06-failover-automatique.md) · [📑 Sommaire](../SOMMAIRE.md) · [14.8 — Stratégies de récupération après incident ➡️](08-strategies-recuperation.md)

⏭️ [Stratégies de récupération après incident](/14-haute-disponibilite/08-strategies-recuperation.md)
