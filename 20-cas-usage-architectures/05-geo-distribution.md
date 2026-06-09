🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.5 Géo-distribution

> **Chapitre 20 — Cas d'Usage et Architectures**  
> Niveau : Avancé  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

Les sections précédentes ont traité du cloisonnement des données — par service (§20.2), puis par client (§20.4). La **géo-distribution** change d'axe : la question n'est plus *qui* accède aux données, mais *où* elles résident physiquement. Il s'agit de répartir l'infrastructure de base de données sur **plusieurs régions géographiques**, pour rapprocher les données des utilisateurs, satisfaire des exigences réglementaires, ou survivre à la perte d'une région entière.

Cette répartition se heurte à une contrainte que l'architecture ne peut contourner : la **vitesse de la lumière**. La latence réseau entre régions éloignées se compte en dizaines, voire centaines de millisecondes, et cette latence entre en conflit direct avec la cohérence forte. Comprendre ce compromis est la clé de toute conception géo-distribuée.

---

## Pourquoi géo-distribuer

Quatre motivations principales conduisent à distribuer une base sur plusieurs régions :

- **Latence** : placer les données près des utilisateurs réduit les temps d'aller-retour. Un utilisateur en Asie servi depuis une base européenne subit une latence bien supérieure à celui servi par une copie locale.
- **Souveraineté et résidence des données** : certaines réglementations imposent que des données restent dans une juridiction précise. La géo-distribution permet de placer les données là où la loi l'exige.
- **Reprise après sinistre** : disposer d'une copie dans une autre région permet de survivre à la perte complète d'un centre de données.
- **Disponibilité** : servir depuis plusieurs régions évite qu'une panne régionale n'interrompe le service mondial.

---

## La contrainte fondamentale : latence vs cohérence

Toute architecture géo-distribuée arbitre entre deux objectifs antagonistes :

- **La réplication synchrone** garantit une **cohérence forte** : chaque écriture n'est validée qu'une fois confirmée par les autres sites. Mais sur un lien longue distance (WAN), cela ajoute la latence inter-région à **chaque** validation, ce qui dégrade fortement les performances d'écriture.
- **La réplication asynchrone** offre une **faible latence** d'écriture (la validation est locale), mais une **cohérence à terme** : les copies distantes accusent un retard, d'où un risque de lectures obsolètes et de perte de données en cas de bascule.

| Approche inter-régions | Cohérence | Latence d'écriture | Perte potentielle (RPO) |
|------------------------|-----------|--------------------|--------------------------|
| **Synchrone (sur WAN)** | Forte | Élevée (attente inter-région à chaque *commit*) | Quasi nulle |
| **Asynchrone** | À terme | Faible (*commit* local) | Non nulle (retard de réplication) |

C'est l'expression, à l'échelle géographique, du **théorème CAP** : en cas de partition réseau entre régions, on doit choisir entre privilégier la cohérence ou la disponibilité. Ce compromis structure les patterns qui suivent.

---

## Le pattern « lecture locale, écriture distante »

L'approche la plus répandue pour la géo-distribution est fondée sur la **réplication asynchrone** (chapitre 13) : une instance primaire dans une région, et des **réplicas asynchrones** dans les autres régions.

- Les **lectures** sont servies par le réplica local, avec une faible latence.
- Les **écritures** sont dirigées vers le primaire, ce qui implique une latence inter-région pour les utilisateurs distants.

Les **GTID** (§13.4) facilitent le repositionnement et la bascule des réplicas, tandis qu'un proxy comme **MaxScale** (§14.4) route automatiquement les lectures vers le réplica local et les écritures vers le primaire (séparation lecture/écriture).

Ce modèle convient particulièrement aux applications **mondiales à dominante de lecture** et à la reprise après sinistre. Ses limites tiennent à la cohérence à terme : un réplica en retard peut renvoyer des données légèrement obsolètes (`Seconds_Behind_Master`, §13.7.2), et une panne du primaire peut entraîner la perte des transactions non encore répliquées.

---

## Galera et le multi-datacenter

Le cluster **Galera** (§14.2) est, par nature, **synchrone** : sa réplication par certification valide chaque transaction à l'échelle de tout le cluster. Cette conception est idéale sur un réseau local à faible latence, mais devient problématique sur un lien longue distance, où la latence inter-région s'ajoute à chaque validation.

Galera peut néanmoins s'étendre sur plusieurs centres de données grâce à la notion de **segments** : on attribue un même numéro de segment aux nœuds d'une même région, ce qui limite le trafic de réplication entre régions (les messages ne traversent le lien WAN qu'une fois par segment plutôt que pour chaque nœud).

```ini
# Les nœuds d'une même région partagent un même numéro de segment
wsrep_provider_options="gmcast.segment=1"
```

Cela rend le multi-datacenter possible, mais ne supprime pas la latence de validation : **un Galera étendu sur de longues distances reste pénalisé par sa nature synchrone.** Il convient donc à des régions relativement proches ou à des charges tolérant une latence d'écriture accrue, mais rarement à une distribution réellement mondiale.

---

## Clusters régionaux + réplication asynchrone entre eux

Le meilleur compromis pour une distribution à grande échelle combine les deux approches : **un cluster Galera par région** (cohérence forte et faible latence *à l'intérieur* de chaque région), reliés entre eux par une **réplication asynchrone** (cohérence à terme *entre* régions).

On obtient ainsi le meilleur des deux mondes : les utilisateurs d'une région bénéficient de la robustesse synchrone locale, tandis que les régions se synchronisent en asynchrone sans pénaliser les écritures. C'est précisément le terrain de la **réplication parallèle entre clusters Galera** (§13.11), où le parallélisme de la réplication (`slave_parallel_threads`) aide à absorber le volume de changements échangés entre clusters distants.

---

## Écritures multi-régions et gestion des conflits

Dès lors que l'on autorise des **écritures dans plusieurs régions** simultanément (modèle actif-actif géographique), un nouveau risque apparaît : les **conflits d'écriture**, lorsque deux régions modifient la même donnée. Galera tranche ces conflits *au sein d'un cluster* par sa certification (la transaction la plus tardive échoue), mais entre clusters reliés en asynchrone, aucun arbitrage automatique n'existe.

La parade la plus sûre consiste à **partitionner les écritures par région** : chaque région est désignée **propriétaire** d'un sous-ensemble des données (par exemple selon la géographie des clients), de sorte que deux régions ne modifient jamais les mêmes lignes. On obtient alors un actif-actif sans conflit, chaque région n'écrivant que sa propre partition. À défaut d'un tel partitionnement, la résolution des conflits doit être prise en charge par l'application, ce qui est complexe et fragile.

---

## Résidence des données et souveraineté

La géo-distribution est aussi un levier de **conformité**. Lorsqu'une réglementation impose que les données d'une catégorie d'utilisateurs restent dans une région donnée, on **place** ces données dans la région concernée.

Cette logique rejoint le multi-tenant (§20.4) : un modèle « base par locataire » (§20.4.1) en variante « instance dédiée » permet d'héberger chaque locataire dans la région exigée par sa réglementation. Plus largement, un **sharding par géographie** — répartir les données selon la localisation des clients — combine résidence des données et proximité, le catalogue de routage assurant le suivi du placement.

---

## Reprise après sinistre : RPO et RTO

La géo-distribution est l'un des piliers d'un plan de reprise (PRA, §12.7). Deux indicateurs en mesurent les objectifs :

- **RPO** (*Recovery Point Objective*) : la quantité de données que l'on accepte de perdre. La réplication asynchrone implique un RPO **non nul** (le retard de réplication est autant de données potentiellement perdues) ; une approche synchrone le ramène quasiment à zéro.
- **RTO** (*Recovery Time Objective*) : le délai de remise en service. Une bascule inter-région suppose de promouvoir un réplica d'une autre région, opération que les GTID et MaxScale facilitent.

Le choix de l'approche de réplication découle donc directement des objectifs de RPO/RTO et des stratégies de récupération (§14.8) : plus on veut limiter la perte de données, plus on tend vers le synchrone — au prix de la latence.

---

## Synthèse

La géo-distribution se ramène, au fond, à un arbitrage permanent entre **latence** et **cohérence**, doublé d'enjeux de souveraineté et de reprise. Les patterns MariaDB s'échelonnent selon ce compromis :

- réplication **asynchrone** inter-régions pour la faible latence et la reprise, au prix d'une cohérence à terme ;
- **Galera** pour la cohérence forte, idéale en intra-région ou multi-datacenter rapproché via les segments ;
- **clusters régionaux reliés en asynchrone** (§13.11) pour conjuguer robustesse locale et portée mondiale ;
- **partitionnement des écritures par région** pour un actif-actif sans conflit ;
- **placement géographique** des données pour la résidence et la souveraineté.

Le bon choix dépend des priorités : exigences de cohérence, sensibilité à la latence, contraintes réglementaires et objectifs de reprise. Cette répartition géographique soulève naturellement la question de l'infrastructure sous-jacente — un ou plusieurs fournisseurs cloud —, qu'aborde la section suivante sur le **cloud hybride et multi-cloud** (§20.6).

---

## Navigation

⬅️ Section précédente : [20.4.3 Shared schema avec discriminateur](04.3-shared-schema.md)  
➡️ Section suivante : [20.6 Hybrid cloud et multi-cloud](06-hybrid-multi-cloud.md)  
⬆️ Retour au [sommaire](../SOMMAIRE.md)

⏭️ [Hybrid cloud et multi-cloud](/20-cas-usage-architectures/06-hybrid-multi-cloud.md)
