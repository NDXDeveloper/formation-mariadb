🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.8 — Stratégies de récupération après incident

> **Chapitre 14 — Haute Disponibilité** · Section 14.8  
> Version de référence : **MariaDB 12.3 LTS**

Quand un incident survient, deux objectifs **distincts** se succèdent : **maintenir le service** (c'est le rôle du failover, 14.6) puis **rétablir un état sain et redondant** (c'est la **récupération**, objet de cette section). Cette section propose un **« playbook »** : les principes de récupération, les procédures par **scénario** de panne, la boîte à outils correspondante et le rôle du **plan de reprise (PRA)**. Elle réutilise abondamment les mécanismes vus précédemment (14.2.3, 14.2.4, 14.3, 14.6) et le chapitre 12.

---

## 1. Récupération, failover et sauvegarde : trois rôles

Il est essentiel de ne pas confondre ces trois mécanismes complémentaires :

| Mécanisme | Rôle | Quand | Section |
|-----------|------|-------|---------|
| **Failover** | Maintenir le **service** | **Pendant** l'incident | 14.6 |
| **Récupération** | Rétablir l'**état sain et redondant** | **Après** l'incident | **14.8** |
| **Sauvegarde** | Filet de sécurité des **données** | Catastrophe (corruption, perte) | Chapitre 12 |

> ⚠️ Rappel capital (14.1) : la HA **réplique les erreurs**. Elle **réduit** le recours à la sauvegarde, mais ne l'**élimine pas** : certains incidents ne se récupèrent **que** par restauration (§3.5).

---

## 2. Principes de la récupération

Avant toute action, quelques règles évitent d'**aggraver** la situation :

- **Service d'abord, redondance ensuite** : le failover a (idéalement) maintenu le service ; la récupération vise à **rétablir la pleine redondance**, sans urgence destructrice.
- **Évaluer la portée avant d'agir** : un seul nœud ? une perte de quorum ? une corruption de données ? le cluster entier ? Le diagnostic conditionne la procédure.
- **Préférer le chemin le moins destructeur** : réintégration **IST** > **SST** > restauration complète (du plus léger au plus lourd).
- **Ne pas empirer** : pas de second amorçage, pas de `pc.bootstrap` sur deux côtés, **isoler (fencing)** avant de réintégrer un ancien primaire (voir 14.3, 14.6).
- **Préserver une copie saine intacte** : ne **jamais expérimenter** sur son **dernier survivant** ni sur son unique sauvegarde.
- **Disposer d'un mode opératoire testé** : une procédure non documentée et non répétée n'existe pas (§5).

---

## 3. Le playbook par scénario

### 3.1 Panne d'un seul nœud

- **Galera** : le cluster a continué via les survivants (quorum maintenu). Le nœud **réintègre automatiquement** au redémarrage — par **IST** si le GCache du donneur couvre l'écart, sinon par **SST** (voir 14.2.4).
- **Réplica (réplication)** : on le **redémarre** ; il **reprend** la réplication à sa dernière position — le **GTID** (13.4) rend cela trivial. S'il est **trop en retard** ou corrompu, on le **re-provisionne** depuis une sauvegarde/clone.

### 3.2 Panne du primaire (réplication)

Le failover (14.6) a **promu un réplica**. La récupération consiste à **réintégrer l'ancien primaire en réplica** (auto-rejoin de `mariadbmon`, ou manuellement). ⚠️ Il faut l'**isoler d'abord** pour éviter un second primaire (split-brain). Si l'ancien primaire détenait des transactions **non encore répliquées** (réplication asynchrone), une **divergence** peut subsister, à **réconcilier ou écarter**.

### 3.3 Perte de quorum (Galera)

Le cluster est passé **`non-Primary`** et a **cessé de servir** (14.3) — les **données restent saines**. On **rétablit le quorum** : si une partie est **réellement hors service**, on **reconstitue la composante primaire** sur le côté survivant via `SET GLOBAL wsrep_provider_pc_bootstrap = 1` (en MariaDB 12.3 ; l'ancienne forme `wsrep_provider_options='pc.bootstrap=YES'` est read-only et renvoie ERROR 1238 — voir 14.3). ⚠️ **À ne faire qu'une fois certain** que l'autre côté est éteint.

### 3.4 Arrêt complet du cluster (Galera)

Tous les nœuds sont arrêtés. On **amorce depuis le nœud le plus avancé** — celui au **seqno le plus élevé** (`safe_to_bootstrap: 1` dans `grastate.dat`, voir 14.2.3). Pour un nœud au **seqno indéterminé** (`-1`, arrêt brutal), on le récupère via `galera_recovery` / `--wsrep-recover` avant de décider. Les autres nœuds **rejoignent** ensuite.

### 3.5 Corruption de données ou erreur humaine

C'est **le** cas où la HA ne suffit pas : un `DROP TABLE` ou un `UPDATE` sans `WHERE` s'est **propagé à tout le cluster**. Aucun nœud n'a la « bonne » version. La seule issue est la **restauration depuis une sauvegarde** (chapitre 12), complétée par une **récupération à un instant précis (PITR)** rejouant les binlogs **jusqu'à juste avant** l'instruction fautive. C'est l'illustration directe de « **HA ≠ sauvegarde** ».

### 3.6 Perte d'un site (sinistre / DR)

Si l'architecture est **géo-distribuée**, on **bascule vers le site de secours**. La récupération consiste ensuite à **reconstruire le site perdu**, le **re-synchroniser**, puis **revenir** (*failback*) quand c'est sûr. ⚠️ Rappel du **piège des deux datacenters** (14.3) : sans **3ᵉ site/arbitre**, on ne survit pas proprement à la perte d'un site (voir aussi 20.5).

### 3.7 Split-brain avéré (le pire cas)

Si un **split-brain** s'est réellement produit (deux côtés ont divergé), la récupération impose de **choisir une source de vérité**, d'**écarter** les écritures divergentes de l'autre côté, puis de **reconstruire**. C'est **coûteux et avec perte** — ce qui rappelle que la **prévention** (quorum, fencing, 14.3) prime toujours sur la guérison.

---

## 4. La boîte à outils de récupération

Chaque scénario mobilise des techniques déjà décrites dans le chapitre ou au chapitre 12 :

| Technique | Quand | Section |
|-----------|-------|---------|
| Réintégration par **IST** (incrémental) | Nœud Galera brièvement absent, écart couvert par le GCache | 14.2.4 |
| Réintégration par **SST** (complet) | Nœud trop en retard, neuf, ou seqno indéterminé | 14.2.4 |
| **Auto-rejoin** | Réintégrer l'ancien primaire en réplica | 14.6 |
| **Amorçage** du nœud le plus avancé | Redémarrage **complet** du cluster Galera | 14.2.3 |
| **`wsrep_provider_pc_bootstrap=1`** | Rétablir la composante primaire après perte de quorum (12.3 ; ≠ ancienne forme `wsrep_provider_options`) | 14.3 |
| **Restauration + PITR** | Corruption, erreur humaine, sinistre | Chapitre 12 |
| **Re-provisionnement** depuis une sauvegarde | Reconstruire un nœud (parfois plus rapide qu'un SST) | Chapitre 12 |

---

## 5. Le plan de reprise (PRA) et son test

Une stratégie de récupération **ne vaut que par sa documentation et sa répétition** :

- **Cibles RTO/RPO par scénario** (14.1) : tous les incidents n'exigent pas le même délai ni la même tolérance à la perte.
- **Modes opératoires (runbooks)** clairs pour chaque scénario du §3, accessibles **même sous stress**.
- **Exercices réguliers** : tests de restauration, simulations de perte de nœud, exercices de failover **et** de failback. Ce sont les **tests de restauration et le PRA** du chapitre **12.7**.

> 💡 **La règle d'or** : un plan de reprise **non testé** doit être considéré comme **inexistant**. Une sauvegarde qu'on n'a jamais su restaurer, une procédure de bootstrap jamais répétée, échouent précisément le jour de l'incident.

---

## 6. Ordre des opérations (synthèse)

Face à un incident, une trame stable aide à ne pas se précipiter :

1. **Évaluer** la portée (un nœud ? quorum ? données ? site ?).
2. **Stabiliser** le service (le failover a-t-il joué ? sinon, rétablir le service).
3. **Rétablir la redondance** par le chemin **le moins destructeur** (§2, §4).
4. **Analyser la cause racine** une fois le calme revenu.
5. **Documenter** l'incident et **améliorer** les runbooks.

Et toujours : **ne pas expérimenter sur le dernier survivant**, **isoler avant de réintégrer**, **préserver une copie saine**.

---

## À retenir

- **Failover** (14.6), **récupération** (14.8) et **sauvegarde** (chapitre 12) sont **trois rôles distincts** : maintenir le service, rétablir la redondance, protéger les données.
- Principes : **service d'abord puis redondance**, **évaluer avant d'agir**, **chemin le moins destructeur** (IST > SST > restauration), **ne pas empirer**, **préserver une copie saine**.
- Playbook par scénario : **panne d'un nœud** (auto-rejoin/IST/SST), **panne du primaire** (rejoin + fencing), **perte de quorum** (`pc.bootstrap`), **arrêt complet** (amorçage du nœud le plus avancé), **corruption/erreur humaine** (restauration + PITR), **perte de site** (DR + failback), **split-brain avéré** (reconstruction avec perte).
- ⚠️ **HA ≠ sauvegarde** : la corruption et l'erreur humaine se récupèrent **uniquement** par restauration et PITR (chapitre 12).
- Un **plan de reprise (PRA)** n'a de valeur que **testé et répété** (voir 12.7) ; un plan non testé est réputé **inexistant**.

La section suivante présente des **alternatives** à MaxScale pour la couche de routage : **ProxySQL et HAProxy**.

---

⬅️ [14.7 — Virtual IP et keepalived](07-virtual-ip-keepalived.md) · [📑 Sommaire](../SOMMAIRE.md) · [14.9 — Alternatives : ProxySQL et HAProxy ➡️](09-proxysql-haproxy.md)

⏭️ [Alternatives : ProxySQL et HAProxy](/14-haute-disponibilite/09-proxysql-haproxy.md)
