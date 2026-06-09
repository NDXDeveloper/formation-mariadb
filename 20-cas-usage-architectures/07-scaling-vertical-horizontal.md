🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.7 Scaling vertical vs horizontal

> **Chapitre 20 — Cas d'Usage et Architectures**  
> Niveau : Avancé  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

Les sections précédentes ont décidé *où* placer les données (§20.5) et *sur quelle infrastructure* (§20.6). Reste une question pratique récurrente : **comment faire croître la capacité de la base** lorsque la charge augmente ? Deux stratégies fondamentales s'opposent — rendre une machine plus puissante, ou multiplier les machines — et leur arbitrage est l'un des plus structurants de toute l'ingénierie des bases de données.

La difficulté propre aux bases de données tient à leur nature : ce sont des composants **avec état**. Distribuer des serveurs applicatifs sans état est relativement simple ; distribuer des **données** l'est beaucoup moins. C'est le fil rouge de tout ce chapitre, et il atteint ici son expression la plus directe.

---

## Deux stratégies fondamentales

- **Scaling vertical** (*scale up*) : augmenter la capacité d'**un seul serveur** — plus de cœurs, plus de mémoire, des disques plus rapides. On rend la machine plus grosse.
- **Scaling horizontal** (*scale out*) : ajouter **davantage de serveurs** et répartir la charge ou les données entre eux. On multiplie les machines.

| Critère | Vertical (*scale up*) | Horizontal (*scale out*) |
|---------|------------------------|---------------------------|
| **Principe** | Une machine plus puissante | Plusieurs machines |
| **Complexité** | Faible | Élevée |
| **Modifications applicatives** | Souvent aucune | Possibles (sharding) |
| **Plafond** | Physique (taille maximale) | Quasi illimité |
| **Disponibilité** | Inchangée (point de défaillance unique) | Améliorée (redondance) |
| **Cohérence** | Forte (nœud unique) | Compromis distribué (CAP) |
| **Coût** | Non linéaire en haut de gamme | Matériel banalisé |
| **Montée des écritures** | Limitée | Possible (sharding) |

---

## Le scaling vertical (scale up)

Monter en puissance consiste à offrir davantage de ressources à une instance unique : dans le cloud, en changeant de type d'instance ; sur site, en remplaçant ou en complétant le matériel.

**Avantages :**

- **Simplicité maximale** : aucune modification de l'application, aucune logique de distribution. La base reste une **unité logique unique**, où toutes les jointures fonctionnent et où l'ACID complet est préservé.
- **Matériel moderne très capable** : avec des serveurs offrant des téraoctets de mémoire et de nombreux cœurs, le scaling vertical mène souvent bien plus loin qu'on ne l'imagine.
- **MariaDB exploite bien les grosses configurations** : dimensionner le *buffer pool* à la mémoire (§15.2), le *Thread Pool* aux cœurs (§11.10) et la configuration I/O aux disques rapides (§15.4) permet de tirer pleinement parti d'une machine puissante (chapitre 15).

**Inconvénients :**

- **Plafond physique** : il existe une taille de machine maximale ; on ne monte pas indéfiniment.
- **Coût non linéaire** : le matériel haut de gamme coûte proportionnellement bien plus cher.
- **Point de défaillance unique** : une machine plus grosse reste une seule machine ; le scaling vertical n'améliore **pas** la disponibilité.
- **Interruption au redimensionnement** : changer de taille impose souvent un redémarrage ou une bascule.
- **Rendements décroissants** : au-delà d'un certain point, la contention et les limites du modèle à écrivain unique réduisent le bénéfice de chaque ressource ajoutée.

---

## Le scaling horizontal (scale out)

Répartir la charge sur plusieurs serveurs lève le plafond du nœud unique, mais introduit la complexité du distribué.

**Avantages :**

- **Pas de plafond strict** : on peut, en principe, continuer d'ajouter des nœuds.
- **Disponibilité accrue** : la redondance de plusieurs nœuds protège contre la panne d'un serveur (notamment avec Galera et les réplicas).
- **Matériel banalisé** : plusieurs machines modestes peuvent revenir moins cher qu'une seule très haut de gamme.
- **Montée en charge des écritures** possible — mais uniquement par le *sharding* (voir plus loin).

**Inconvénients :**

- **Complexité** : routage, rééquilibrage, supervision d'un parc — la charge d'exploitation croît.
- **Cohérence distribuée** : transactions inter-nœuds, cohérence à terme, compromis CAP (déjà rencontrés en §20.5).
- **Requêtes et jointures inter-nœuds** : difficiles, voire impossibles, lorsque les données sont réparties.
- **Modifications applicatives** : le sharding, en particulier, exige que l'application connaisse la clé de répartition.

---

## La distinction clé : lectures vs écritures

C'est le point le plus important — et le plus souvent mal compris — du passage à l'échelle d'une base. **Monter en charge les lectures et monter en charge les écritures sont deux problèmes de difficulté très différente.**

### Monter en charge les lectures : relativement aisé

Pour une charge à dominante de lecture, on ajoute des **réplicas** (réplication asynchrone ou semi-synchrone, chapitre 13) ou des **nœuds Galera** (chapitre 14), et l'on **route les lectures vers eux** à l'aide de **MaxScale** (§14.4) ou de ProxySQL/HAProxy (§14.9), tandis que les écritures vont au primaire. C'est une solution éprouvée et efficace : multiplier les réplicas multiplie la capacité de lecture.

### Monter en charge les écritures : difficile

Les écritures, elles, se heurtent à un obstacle de fond : **un primaire unique les traite toutes.**

- Les **réplicas** n'aident pas : ils ne font que **rejouer** les écritures du primaire ; ils n'en absorbent pas la charge.
- **Galera** n'aide pas davantage pour le débit d'écriture : **chaque nœud applique chaque écriture** et participe à la certification. Ajouter des nœuds Galera améliore la disponibilité et la capacité de **lecture**, mais **pas** le débit d'écriture global.

La **seule** façon de monter en charge les écritures horizontalement est le **sharding** : partitionner les **données** elles-mêmes entre plusieurs serveurs, chacun n'hébergeant qu'un sous-ensemble, de sorte que des écritures différentes touchent des serveurs différents. MariaDB le permet via le moteur **Spider** (§7.10.3, §15.11) ou un sharding piloté par l'application.

Mais le sharding a un coût élevé : choix d'une **clé de répartition** pertinente, **routage** des requêtes, difficulté des **requêtes et transactions inter-shards**, et **rééquilibrage** des données lors de l'ajout d'un shard. C'est l'option la plus puissante, mais aussi la plus complexe.

> **À retenir.** Lectures saturées → ajouter des réplicas. Écritures saturées → *sharder*. Confondre les deux conduit à empiler des réplicas pour un problème d'écriture qu'ils ne résoudront jamais.

---

## Vertical et horizontal sont complémentaires

Ces deux stratégies ne s'excluent pas : les systèmes réels les **combinent**. On déploie volontiers un cluster Galera de **nœuds puissants** (vertical *et* horizontal), ou une architecture *shardée* où **chaque shard est lui-même répliqué** pour la disponibilité. Le scaling vertical optimise chaque nœud ; le scaling horizontal multiplie les nœuds et apporte la redondance.

---

## Une progression pragmatique

Plutôt qu'un choix binaire d'emblée, le passage à l'échelle suit en pratique une progression, du moins coûteux au plus complexe :

1. **Optimiser d'abord.** Avant tout ajout de ressources, on révise les requêtes, les index, le schéma et la configuration (chapitres 5 et 15). Un système lent est souvent **sous-optimisé** plutôt que **sous-dimensionné** — et c'est la « mise à l'échelle » la moins chère.
2. **Monter en puissance (vertical).** L'étape la plus simple, sans modification applicative ; le matériel moderne porte loin.
3. **Répartir les lectures (horizontal).** Pour une charge de lecture importante, ajouter des réplicas et router les lectures.
4. **Sharder (horizontal, écritures).** En dernier recours, lorsque la capacité d'écriture d'un primaire unique est réellement épuisée — l'option la plus puissante, mais à n'engager qu'à bon escient.

Cette progression évite l'écueil du **sharding prématuré**, qui impose la complexité du distribué à des systèmes que l'optimisation ou le scaling vertical auraient suffi à satisfaire.

---

## Synthèse

Le passage à l'échelle d'une base se ramène à un arbitrage entre **simplicité** (vertical) et **capacité/disponibilité au prix de la complexité** (horizontal), avec une asymétrie fondamentale entre lectures (faciles à répartir) et écritures (qui n'évoluent que par le sharding). La boîte à outils MariaDB couvre tout l'éventail : tuning d'un nœud (chapitre 15), réplicas et Galera pour les lectures et la disponibilité (chapitres 13 et 14), Spider et sharding pour les écritures (§15.11).

Le bon choix dépend du **profil de charge** (lecture ou écriture dominante), des **exigences de disponibilité**, et du **plafond et coût** acceptables. Et parce que la difficulté réside toujours dans la distribution des **données**, les architectures à grande échelle s'appuient de plus en plus sur la propagation **par événements** pour décorréler les composants et diffuser les changements. C'est l'objet de la section suivante : **MariaDB dans les architectures orientées événements** (§20.8).

---

## Navigation

⬅️ Section précédente : [20.6 Hybrid cloud et multi-cloud](06-hybrid-multi-cloud.md)  
➡️ Section suivante : [20.8 MariaDB dans les architectures Event-Driven](08-architectures-event-driven.md)  
⬆️ Retour au [sommaire](../SOMMAIRE.md)

⏭️ [MariaDB dans les architectures Event-Driven](/20-cas-usage-architectures/08-architectures-event-driven.md)
