🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.1 — Architectures haute disponibilité : Concepts

> **Chapitre 14 — Haute Disponibilité** · Section 14.1  
> Version de référence : **MariaDB 12.3 LTS**

Avant de déployer un cluster Galera, de configurer MaxScale ou de mettre en place un *failover* automatique, il faut maîtriser le vocabulaire et les principes qui sous-tendent **toutes** les architectures de haute disponibilité. Cette section pose ces fondations : que signifie « hautement disponible », comment mesurer la disponibilité, et quels grands compromis structurent la conception d'un système résilient. Ces concepts sont indépendants des outils ; ils éclairent les choix techniques des sections suivantes (Galera en 14.2, split-brain et quorum en 14.3, MaxScale en 14.4).

---

## 1. Qu'est-ce que la haute disponibilité ?

La **haute disponibilité** (*High Availability*, HA) est la propriété d'un système à **rester opérationnel et accessible pendant une proportion très élevée du temps**, malgré les pannes de ses composants. Il ne s'agit pas d'empêcher toute panne — c'est impossible — mais de faire en sorte que la panne d'un élément n'entraîne pas l'indisponibilité du service.

Plusieurs notions voisines sont souvent confondues. Les distinguer est essentiel pour concevoir correctement :

| Notion | Question à laquelle elle répond | Exemple sous MariaDB |
|--------|-------------------------------|----------------------|
| **Haute disponibilité (HA)** | Le service reste-t-il *accessible* malgré une panne ? | Un nœud Galera tombe, les autres continuent de servir. |
| **Tolérance aux pannes** | Le service continue-t-il *sans interruption ni dégradation* perceptible ? | Forme la plus exigeante de HA (RTO proche de 0). |
| **Durabilité** | Une donnée *validée* survit-elle à une panne ? | Propriété **D** d'ACID, garantie par le redo log InnoDB. |
| **Scalabilité (montée en charge)** | Le système absorbe-t-il *plus de charge* ? | Répartir les lectures sur des réplicas. |
| **Reprise après sinistre (DR)** | Comment redémarrer après un *désastre majeur* (perte d'un datacenter) ? | Restaurer sur un site distant. |

> ⚠️ **Disponibilité ≠ durabilité ≠ scalabilité.** Un système peut être très disponible mais peu durable (s'il sert des données non encore persistées), ou très scalable mais vulnérable à une panne. La HA traite spécifiquement de l'**accessibilité du service dans le temps**.

---

## 2. Mesurer la disponibilité

On ne peut pas améliorer ce que l'on ne mesure pas. Trois familles d'indicateurs décrivent la disponibilité.

### 2.1 Le pourcentage de disponibilité (« les neuf »)

La disponibilité s'exprime en pourcentage de temps de fonctionnement sur une période. On parle du nombre de « neuf » :

| Disponibilité | Surnom | Indisponibilité / an | Indisponibilité / mois |
|--------------|--------|----------------------|------------------------|
| 99 % | *two nines* | ≈ 3,65 jours | ≈ 7,3 heures |
| 99,9 % | *three nines* | ≈ 8,77 heures | ≈ 43,8 minutes |
| 99,99 % | *four nines* | ≈ 52,6 minutes | ≈ 4,38 minutes |
| 99,999 % | *five nines* | ≈ 5,26 minutes | ≈ 26,3 secondes |
| 99,9999 % | *six nines* | ≈ 31,5 secondes | ≈ 2,6 secondes |

Chaque « neuf » supplémentaire est **exponentiellement plus coûteux** à atteindre. Passer de 99,9 % à 99,99 % peut nécessiter de doubler les nœuds, d'automatiser le *failover* et de répartir l'infrastructure sur plusieurs zones.

### 2.2 La formule : MTBF, MTTR et disponibilité

La disponibilité se calcule à partir de deux durées moyennes :

- **MTBF — *Mean Time Between Failures*** : temps moyen *entre deux pannes* (mesure la fiabilité).
- **MTTR — *Mean Time To Repair/Recovery*** : temps moyen pour *rétablir* le service après une panne.
- (**MTTF — *Mean Time To Failure*** : variante pour les composants non réparables.)

$$
\text{Disponibilité} = \frac{\text{MTBF}}{\text{MTBF} + \text{MTTR}}
$$

**Exemple.** Un serveur avec un MTBF de 1000 h et un MTTR de 1 h :
`A = 1000 / (1000 + 1) = 0,999` → **99,9 %**.

Si l'on réduit le MTTR à 6 minutes (0,1 h) grâce à un *failover* automatique :
`A = 1000 / 1000,1 = 0,9999` → **99,99 %**.

> 💡 **Insight clé.** Améliorer la disponibilité en réduisant le **MTTR** (automatisation, bascule rapide) est souvent **moins coûteux** qu'en augmentant le **MTBF** (matériel plus fiable). C'est précisément ce que vise la HA : on accepte que les pannes surviennent, mais on **récupère vite**.

### 2.3 Disponibilité des systèmes composés : série vs parallèle

Un service de base de données est une **chaîne de composants** (application → réseau → proxy → serveur → stockage). La disponibilité globale dépend de la façon dont ils sont assemblés.

**Composants en série** (tous doivent fonctionner) — la disponibilité **diminue** :

$$
A_{total} = A_1 \times A_2 \times \dots \times A_n
$$

Deux composants à 99 % en série → `0,99 × 0,99 = 0,9801` → **98,01 %**. Chaque dépendance ajoutée *abaisse* la disponibilité.

**Composants en parallèle** (au moins un doit fonctionner — c'est la redondance) — la disponibilité **augmente** :

$$
A_{total} = 1 - (1 - A_1)(1 - A_2) \dots (1 - A_n)
$$

Deux composants à 99 % en parallèle → `1 - (0,01 × 0,01) = 0,9999` → **99,99 %**. La redondance transforme deux « neuf » en quatre.

> ⚠️ **Idéalisation.** Ces formules supposent des pannes **indépendantes** et une bascule **instantanée et parfaite**. En réalité, les pannes corrélées (même rack, même alimentation, même bug logiciel), le temps de bascule et la couche de routage abaissent la disponibilité réelle. C'est pourquoi on répartit les nœuds sur des **domaines de défaillance** distincts (§8).

---

## 3. RTO et RPO : les objectifs de continuité

Deux objectifs métier guident toute architecture de HA et de reprise. Ils se lisent autour du moment de l'incident :

```
        ← RPO →                          ← RTO →
──────────●────────────────⚡──────────────────●─────────►  temps
   dernier point         incident          service
   de cohérence                            rétabli
   (données sûres)
```

- **RPO — *Recovery Point Objective*** : **quantité maximale de données que l'on accepte de perdre.** Mesuré *en arrière* depuis l'incident. Un RPO de 0 signifie « aucune perte tolérée » ; un RPO de 5 minutes signifie « on accepte de perdre jusqu'à 5 minutes de transactions ».

- **RTO — *Recovery Time Objective*** : **durée maximale d'interruption tolérée.** Mesuré *en avant* depuis l'incident. Un RTO de 30 secondes impose une bascule automatique ; un RTO de plusieurs heures autorise une restauration manuelle.

Le RPO atteignable dépend directement de la **stratégie de réplication** :

| Stratégie | RPO typique | RTO typique |
|-----------|-------------|-------------|
| Réplication **asynchrone** | > 0 (perte du *lag* en cours) | Faible si *failover* automatisé |
| Réplication **semi-synchrone** | Très faible | Faible |
| Réplication **synchrone** (Galera) | ≈ 0 | Très faible (nœuds déjà à jour) |
| Sauvegarde + journaux binaires (PITR) | Selon fréquence | Élevé (restauration) |

### SLA, SLO et SLI

Ces objectifs s'inscrivent dans une hiérarchie contractuelle :

- **SLI — *Service Level Indicator*** : la **métrique mesurée** (ex. % de requêtes réussies, disponibilité observée).
- **SLO — *Service Level Objective*** : la **cible interne** à atteindre (ex. « 99,95 % de disponibilité par trimestre »).
- **SLA — *Service Level Agreement*** : l'**engagement contractuel** envers le client, souvent assorti de pénalités. Le SLA est généralement *moins strict* que le SLO interne, pour conserver une marge.

---

## 4. Le point unique de défaillance (SPOF)

Un **SPOF (*Single Point of Failure*)** est un composant dont la panne, à lui seul, entraîne l'indisponibilité de tout le service. **Identifier et éliminer les SPOF est l'objectif central de la conception HA.**

Dans une pile de base de données typique, les SPOF candidats sont nombreux :

- le **serveur de base de données** unique ;
- l'unique **commutateur réseau** ou lien réseau ;
- l'**alimentation électrique** ou le **rack** unique ;
- le **proxy** ou le **load balancer** unique ;
- l'enregistrement **DNS** ou l'**adresse IP** d'accès ;
- voire le **datacenter** entier.

> ⚠️ **Le paradoxe de la HA : les remèdes peuvent devenir des SPOF.** Ajouter un proxy (MaxScale, ProxySQL) ou une IP virtuelle pour faciliter la bascule introduit un **nouveau** composant unique. Ces éléments doivent **eux-mêmes** être rendus redondants (plusieurs instances de proxy, *keepalived* sur plusieurs hôtes — voir 14.7 et 14.9), sous peine de simplement déplacer le SPOF.

---

## 5. La redondance et ses formes

La redondance — disposer de plus de ressources que le strict nécessaire — est le moyen fondamental d'éliminer les SPOF. Elle prend plusieurs formes.

### 5.1 Actif/Passif vs Actif/Actif

| Critère | **Actif/Passif** | **Actif/Actif** |
|---------|------------------|-----------------|
| Principe | Un nœud sert, le(s) autre(s) en attente | Plusieurs nœuds servent simultanément |
| Utilisation des ressources | Capacité passive inexploitée | Toutes les ressources utilisées |
| Conflits d'écriture | Aucun (un seul écrivain) | À gérer (certification Galera, ou *sharding*) |
| Complexité | Plus simple | Plus complexe |
| RTO | Délai de promotion du passif | Potentiellement quasi nul |
| Exemple MariaDB | Primaire + réplica avec bascule | Galera Cluster multi-maître |

### 5.2 Niveaux de préparation du standby (cold / warm / hot)

Pour une architecture actif/passif, l'état de préparation du nœud de secours détermine le RTO :

| Type de standby | État | RTO | Coût |
|-----------------|------|-----|------|
| **Cold** (froid) | Serveur arrêté ou non synchronisé ; à provisionner et restaurer | Élevé | Faible |
| **Warm** (tiède) | Serveur démarré, reçoit la réplication, mais ne sert pas ; promotion nécessaire | Moyen | Moyen |
| **Hot** (chaud) | Serveur synchronisé et prêt à servir immédiatement | Faible | Élevé |

Un cluster **actif/actif** revient à n'avoir que des nœuds « chauds ».

---

## 6. Synchrone vs asynchrone : le compromis fondamental

C'est le choix structurant de toute réplication de données. Il oppose **cohérence/durabilité** d'un côté et **performance/latence** de l'autre.

| Aspect | **Réplication synchrone** | **Réplication asynchrone** |
|--------|---------------------------|----------------------------|
| Validation d'une écriture | Attend la confirmation des autres nœuds | Locale, puis propagation différée |
| RPO | ≈ 0 (pas de perte sur panne d'un nœud) | > 0 (perte possible du *lag*) |
| Latence en écriture | Plus élevée (dépend du réseau et du nœud le plus lent) | Faible |
| Sensibilité à la latence réseau | Forte (problématique en WAN) | Faible |
| Exemple | Galera (virtuellement synchrone) | Réplication classique source-réplica |

Le **semi-synchrone** (chapitre 13) est un compromis intermédiaire : la source attend qu'**au moins un** réplica ait *reçu* (mais pas forcément appliqué) la transaction avant de la valider.

> Ce compromis n'est pas qu'une affaire de panne : il a un coût **permanent** sur la latence d'écriture, même en fonctionnement normal — point que le théorème PACELC formalise (§7).

---

## 7. Le théorème CAP (et son extension PACELC)

Le **théorème CAP** est un modèle mental incontournable pour raisonner sur les systèmes de données distribués. Il énonce qu'en présence d'une **partition réseau**, un système distribué ne peut garantir simultanément que **deux** des trois propriétés suivantes :

- **C — Consistency (cohérence)** : toute lecture renvoie l'écriture la plus récente (ou une erreur).
- **A — Availability (disponibilité)** : toute requête reçoit une réponse non erronée, sans garantie qu'elle soit la plus récente.
- **P — Partition tolerance (tolérance au partitionnement)** : le système continue de fonctionner malgré la perte de messages entre nœuds.

Comme une partition réseau **finit toujours par survenir** dans un système distribué, **P n'est pas optionnel**. Le vrai choix, *lors d'une partition*, est donc entre **C et A** :

| Choix lors d'une partition | Comportement | Approche MariaDB correspondante |
|----------------------------|--------------|----------------------------------|
| **Privilégier C** (CP) | La partition minoritaire **refuse de servir** pour ne pas diverger | Galera : un sous-groupe sans quorum devient *non-Primary* et cesse d'accepter les requêtes (voir 14.3) |
| **Privilégier A** (AP) | Tous les nœuds restent disponibles, au risque de servir des **données obsolètes** | Réplicas asynchrones qui continuent de répondre malgré le *lag* |

### PACELC : le compromis même hors panne

L'extension **PACELC** complète le modèle : *« en cas de **P**artition, choisir entre **A** et **C** ; **E**lse (sinon, en fonctionnement normal), choisir entre **L**atency (latence) et **C**onsistency (cohérence). »*

Cela explique pourquoi la réplication synchrone (forte cohérence) impose une latence accrue **en permanence**, et pas seulement lors des pannes — exactement le compromis vu en §6.

> ℹ️ **Mise en garde.** CAP est un modèle volontairement simplifié (les propriétés sont en réalité des spectres, pas des absolus). Il reste un excellent outil pour **catégoriser les compromis**, mais ne dispense pas d'analyser le comportement précis de chaque système.

---

## 8. Domaines de défaillance et distribution géographique

Un **domaine de défaillance** (*fault/failure domain*) est un périmètre dont les composants partagent un risque commun et peuvent tomber **ensemble**. Du plus petit au plus grand :

```
serveur  ⊂  rack  ⊂  salle  ⊂  datacenter  ⊂  zone de disponibilité (AZ)  ⊂  région
```

**Principe directeur :** répartir les nœuds redondants sur des domaines **distincts**, afin que la défaillance d'un domaine entier (un rack qui perd son alimentation, une AZ qui tombe) ne mette pas hors service tout le cluster. Placer trois réplicas dans le même rack ne protège que des pannes d'un seul serveur, pas d'une coupure du rack.

> ⚠️ **Tension géo-distribution / synchrone.** Répartir un cluster **synchrone** (Galera) sur des sites éloignés améliore la résilience géographique mais **dégrade les écritures** : chaque validation attend l'aller-retour réseau (RTT) entre sites. C'est pourquoi le multi-région combine souvent **Galera local** (synchrone, faible latence) et **réplication asynchrone entre sites** — un schéma facilité en 12.3 par la réplication parallèle entre clusters Galera (voir 13.11).

---

## 9. Détecter la panne : *health checks*, faux positifs et *fencing*

Une bascule ne vaut que par la **fiabilité de la détection** qui la déclenche.

- **Health checks / *heartbeats*** : les nœuds et le superviseur s'échangent des signaux périodiques. L'absence de réponse au-delà d'un seuil est interprétée comme une panne.
- **Le danger des faux positifs** : une *latence réseau passagère* peut être prise pour une panne et déclencher un *failover* inutile — voire amener deux nœuds à se croire simultanément primaires, c'est-à-dire un **split-brain** (traité en détail en 14.3).
- **Quorum** : exiger qu'une **majorité** de nœuds s'accorde avant toute décision empêche un sous-groupe minoritaire d'agir seul. C'est la défense principale contre le split-brain.
- ***Fencing* / STONITH** (*Shoot The Other Node In The Head*) : isoler de force un nœud suspect (couper son accès au stockage ou au réseau) pour l'empêcher de corrompre les données après une bascule erronée.

Bien régler les **seuils de détection** est un art : trop sensibles, ils provoquent des bascules intempestives ; trop laxistes, ils allongent le RTO.

---

## 10. Les trois couches d'une architecture HA

Une architecture de haute disponibilité complète s'organise en trois couches complémentaires. Cette grille de lecture structure le reste du chapitre :

| Couche | Rôle | Sections du chapitre |
|--------|------|----------------------|
| **1. Redondance des données** | Disposer de plusieurs copies à jour des données | Réplication (ch. 13), **Galera (14.2)** |
| **2. Détection et orchestration** | Repérer la panne et basculer | Split-brain & quorum (14.3), **failover automatique (14.6)** |
| **3. Routage et point d'accès stable** | Présenter à l'application un point d'entrée constant | VIP/keepalived (14.7), **MaxScale (14.4–14.5)**, ProxySQL/HAProxy (14.9), *Transaction Replay* (14.10) |

À ces trois couches s'ajoute le rôle de l'**application elle-même** : un client robuste gère la reconnexion, le *retry* idempotent et le *connection pooling*, pour absorber les microcoupures sans erreur visible pour l'utilisateur.

---

## 11. HA, reprise après sinistre et sauvegarde : ne pas confondre

Ces trois mécanismes répondent à des risques **différents** et sont **complémentaires**, jamais substituables :

| Mécanisme | Protège contre | Portée | RTO/RPO visés |
|-----------|----------------|--------|---------------|
| **Haute disponibilité** | Panne d'un composant (nœud, réseau) | Locale, automatique | Très faibles |
| **Reprise après sinistre (DR)** | Catastrophe majeure (perte d'un site) | Distante (autre région) | Plus élevés, acceptables |
| **Sauvegarde** | Corruption, suppression, *ransomware*, erreur humaine | Restauration à un instant T | Selon la stratégie |

> ⚠️ **Point capital : la HA ne remplace pas les sauvegardes.** Un cluster réplique fidèlement les **erreurs** : un `DROP TABLE` ou un `UPDATE` sans `WHERE` se propage instantanément à tous les nœuds. Seule une **sauvegarde** (chapitre 12) permet de revenir en arrière. HA et sauvegarde traitent des risques orthogonaux et doivent **coexister**.

---

## À retenir

- La HA vise à **maintenir le service accessible** malgré les pannes ; elle se distingue de la durabilité, de la scalabilité et de la reprise après sinistre.
- La disponibilité se mesure en **« neuf »** et se calcule via `MTBF / (MTBF + MTTR)` ; **réduire le MTTR** (automatisation) est souvent le levier le plus rentable.
- En **série**, la disponibilité diminue ; en **parallèle** (redondance), elle augmente.
- **RTO** (durée d'interruption tolérée) et **RPO** (perte de données tolérée) sont les objectifs métier qui dictent la stratégie de réplication.
- Concevoir la HA, c'est **éliminer les SPOF** — sans en créer de nouveaux avec les composants ajoutés.
- Le compromis **synchrone/asynchrone**, formalisé par **CAP/PACELC**, oppose cohérence et latence/disponibilité.
- Répartir les nœuds sur des **domaines de défaillance distincts** et sécuriser la **détection de panne** (quorum, *fencing*) sont indispensables.
- HA, DR et **sauvegarde** sont **complémentaires** : un cluster réplique aussi les erreurs.

Ces concepts en main, la section suivante les met en œuvre concrètement avec la technologie phare de la HA synchrone sous MariaDB : **Galera Cluster**.

---

⬅️ [14 — Haute Disponibilité (introduction)](README.md) · [📑 Sommaire](../SOMMAIRE.md) · [14.2 — MariaDB Galera Cluster ➡️](02-galera-cluster.md)

⏭️ [MariaDB Galera Cluster](/14-haute-disponibilite/02-galera-cluster.md)
