🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.3 — Split-brain et quorum

> **Chapitre 14 — Haute Disponibilité** · Section 14.3  
> Version de référence : **MariaDB 12.3 LTS**

Toute architecture distribuée affronte un même danger fondamental : le **split-brain** (« cerveau divisé »). La parade s'appelle le **quorum**. Ces deux notions, évoquées en 14.1 (quorum, *fencing*) et en 14.2.1 (composante primaire), méritent une section dédiée car elles conditionnent la **sûreté** d'un cluster. On les illustre ici principalement avec Galera, mais le principe s'applique à toutes les solutions de HA.

---

## 1. Qu'est-ce que le split-brain ?

Le **split-brain** survient lorsqu'une **partition réseau** coupe un cluster en plusieurs sous-groupes, et que **chaque sous-groupe se croit seul légitime** et continue d'accepter des écritures **indépendamment**. Les données des différentes parties **divergent** alors de façon **irréconciliable**.

```
Avant la partition :                Après une partition réseau :

   ┌────┐ ┌────┐ ┌────┐                ┌────┐ ┌────┐  ╳  ┌────┐
   │ N1 │─│ N2 │─│ N3 │     ──►        │ N1 │─│ N2 │  ╳  │ N3 │
   └────┘ └────┘ └────┘                └────┘ └────┘  ╳  └────┘
     cluster cohérent                   2 nœuds (2/3)      1 nœud (1/3)
                                        majorité            minorité
```

Le scénario classique se décline selon la topologie :

- **Galera (multi-maître)** : deux groupes de nœuds qui continueraient tous deux à servir finiraient par contenir des données contradictoires.
- **Actif/passif (réplication + VIP)** : deux nœuds se croient tous deux *primaires* (« dual primary »), détiennent l'IP virtuelle et écrivent chacun de leur côté.

---

## 2. Pourquoi c'est catastrophique

Le split-brain est **pire qu'une indisponibilité**. Une panne franche arrête le service, mais ne corrompt pas les données ; un split-brain, lui, produit **deux jeux de données divergents** qu'on ne peut généralement **pas fusionner** :

- quelle écriture l'emporte lorsque les deux côtés ont modifié la même ligne différemment ?
- la « réconciliation » impose presque toujours de **sacrifier les écritures d'un côté**, donc de **perdre des données**.

C'est une **corruption silencieuse** : le service paraît fonctionner des deux côtés, mais l'intégrité est déjà rompue. D'où la priorité absolue : **empêcher** le split-brain plutôt que d'avoir à le réparer.

---

## 3. Le quorum : la parade

Le **quorum** est le **nombre minimal de voix** (de membres) qu'une partition doit réunir pour être **autorisée à continuer** de fonctionner.

Le principe est simple et puissant :

> Seule la partition détenant une **majorité** (strictement plus de 50 % des voix) peut continuer ; **toute partition minoritaire doit s'arrêter** (cesser de servir, au moins en écriture).

Comme **deux majorités ne peuvent pas coexister**, ce mécanisme garantit qu'**au plus une seule partition** reste active à un instant donné — le split-brain devient **impossible**. C'est exactement le choix **cohérence (C) plutôt que disponibilité (A)** du théorème CAP (voir 14.1) : la minorité **renonce à servir** pour préserver l'intégrité.

---

## 4. Pourquoi un nombre impair de nœuds

Le quorum explique l'insistance, depuis le début de ce chapitre, sur un **nombre impair de nœuds** :

- avec un nombre **impair** (3, 5, 7), toute partition désigne **toujours** un côté majoritaire ;
- avec un nombre **pair**, une coupure 50/50 ne donne **aucune majorité** → **les deux côtés s'arrêtent** (pas de split-brain, mais plus de service).

La tolérance aux pannes d'un cluster de *N* nœuds votants suit la règle : il survit à la perte de **⌊(N−1)/2⌋** nœuds.

| Nœuds (votants) | Majorité requise | Pannes tolérées | Remarque |
|-----------------|------------------|-----------------|----------|
| 1 | 1 | 0 | Aucune redondance |
| 2 | 2 | 0 | ⚠️ Le piège des 2 nœuds (voir ci-dessous) |
| **3** | 2 | **1** | ✅ Minimum recommandé |
| 4 | 3 | 1 | Pas mieux que 3, mais risque d'égalité accru |
| **5** | 3 | **2** | ✅ Pour une tolérance supérieure |
| 7 | 4 | 3 | Rarement nécessaire |

> ⚠️ **Le piège des 2 nœuds.** Un cluster à 2 nœuds **ne tolère aucune panne brutale** : si un nœud disparaît sans préavis (crash, partition), le survivant ne détient plus que 50 % des voix — **pas une majorité** — et s'arrête donc lui aussi. Nuance importante : un **arrêt propre** (*graceful*) est géré différemment (le cluster réduit sa composition attendue, et le survivant reste primaire) ; c'est la **disparition non annoncée** qui fait perdre le quorum. D'où la nécessité d'un **3ᵉ vote** (un nœud supplémentaire ou un arbitre, §6).

---

## 5. Galera : la composante primaire (*Primary Component*)

Chez Galera, le quorum se matérialise par la **composante primaire**, déjà introduite en 14.2.1. Lors d'une partition :

1. Galera évalue le quorum sur la base de la **dernière composition connue** du cluster.
2. Le groupe **majoritaire** (> 50 %) devient/reste la **composante primaire** : `wsrep_cluster_status = Primary`. Il **continue** de servir.
3. Le groupe **minoritaire** passe **`non-Primary`** : il **refuse les requêtes** (il renvoie une erreur tant qu'il n'a pas rejoint une composante primaire) et **cesse de servir**.

C'est la traduction concrète du choix **cohérence avant disponibilité** : un nœud isolé préfère **refuser de répondre** plutôt que de risquer de diverger. On surveille cet état via `wsrep_cluster_status` (doit valoir `Primary`) et `wsrep_cluster_size` (voir 14.2.3).

---

## 6. L'arbitre Galera (`garbd`)

Pour les configurations à **nombre pair** de nœuds de données — typiquement **2 nœuds** — Galera fournit un **arbitre** : le démon **`garbd`** (*Galera Arbitrator*). C'est un **membre votant** de la communication de groupe qui :

- **participe au quorum** (il compte comme une voix) ;
- ne **stocke aucune donnée** ;
- ne **reçoit aucun trafic SQL**.

Il rétablit ainsi l'**imparité** nécessaire à un quorum toujours tranchable, à moindre coût (un petit processus, pas un serveur complet).

> ⚠️ **Placement de l'arbitre.** `garbd` doit tourner sur un **hôte tiers, indépendant** des deux nœuds de données — idéalement dans un **domaine de défaillance distinct** (voir 14.1). L'installer sur la machine de l'un des deux nœuds ne servirait à rien : la panne de cette machine ferait perdre **deux voix** d'un coup.

---

## 7. Poids de quorum et multi-datacenter

### Poids (`pc.weight`)

Par défaut, chaque nœud pèse **une voix**. Galera permet d'attribuer des **poids** différents via l'option `pc.weight` (dans `wsrep_provider_options` au démarrage, ou la variable dynamique `wsrep_provider_pc_weight` à chaud — voir la note du §8) ; le quorum se calcule alors sur la **somme des poids**. On peut ainsi donner **plus de poids au datacenter principal**, pour qu'il conserve le quorum si le lien vers un site secondaire tombe.

### Le piège des deux datacenters

C'est une erreur de conception fréquente : **avec seulement deux sites, on ne peut survivre à la perte d'aucun des deux** tout en gardant le quorum.

- Mettre la majorité dans le DC1 → la perte du DC1 tue le quorum.
- Répartir à égalité → une coupure du lien inter-DC laisse **deux moitiés sans majorité**, donc **tout s'arrête**.

La seule solution robuste est d'introduire un **3ᵉ emplacement** — ne serait-ce qu'un **arbitre `garbd`** sur un site tiers (ou un cloud) — pour **départager** en cas de coupure entre les deux DC principaux.

> ℹ️ Sur un déploiement multi-DC, on regroupe par ailleurs les nœuds d'un même site dans un **segment** (`gmcast.segment`) pour limiter le trafic de réplication traversant le WAN : un seul nœud par segment relaie vers les autres segments.

---

## 8. Récupérer après une perte de quorum (`pc.bootstrap`)

Si le cluster **perd entièrement le quorum** (mauvaise partition, trop de nœuds tombés), **tous** les nœuds survivants passent `non-Primary` et **cessent de servir** — le service est interrompu, mais les données restent saines.

Pour rétablir le service, un administrateur peut **forcer manuellement** la (re)constitution d'une composante primaire sur un nœud choisi. En **MariaDB 12.3**, le plugin **`wsrep_provider`** (intégré depuis la 11.4, actif avec Galera) expose chaque option du provider Galera sous forme de **variable système individuelle** ; on déclenche donc l'amorçage par :

```sql
SET GLOBAL wsrep_provider_pc_bootstrap = 1;   -- MariaDB 12.3 (vérifié sur 12.3.2)
```

> ⚠️ **Forme historique read-only en 12.3.** L'ancienne syntaxe `SET GLOBAL wsrep_provider_options = 'pc.bootstrap=YES'` (encore courante dans la documentation Galera/Codership) **échoue en MariaDB 12.3** avec `ERROR 1238 : Variable 'wsrep_provider_options' is a read only variable` : lorsque le plugin `wsrep_provider` est actif, `wsrep_provider_options` n'est modifiable **qu'au démarrage** (fichier d'options), et les changements **à chaud** passent par les variables `wsrep_provider_<option>` (ici `wsrep_provider_pc_bootstrap`).

> ⚠️ **Manœuvre dangereuse.** Forcer `pc.bootstrap` revient à dire à un nœud « tu es désormais la composante primaire ». Le faire sur le **mauvais** nœud, ou sur **deux côtés** d'une partition, **recrée précisément un split-brain**. On ne l'utilise qu'**après s'être assuré** que l'autre partie est réellement hors service. Cette opération se rattache à l'amorçage vu en 14.2.3 et aux procédures de reprise du 14.8.

---

## 9. Split-brain hors Galera

Le risque ne concerne pas que les clusters synchrones. Dans une architecture **réplication asynchrone + VIP** (actif/passif), le split-brain prend la forme de **deux primaires** simultanés. Les parades sont alors :

- le **fencing / STONITH** (voir 14.1) : isoler de force le nœud suspect pour l'empêcher d'écrire ;
- un **gestionnaire de cluster à quorum** (par exemple Pacemaker/Corosync) qui applique la même logique de majorité aux ressources et à la VIP ;
- une **arbitration de l'IP virtuelle** (voir 14.7) pour qu'un seul nœud puisse la détenir.

La leçon est générale : **tout mécanisme de bascule a besoin d'un moyen d'empêcher l'existence de deux primaires**.

---

## 10. Principes de conception anti-split-brain

| Principe | Pourquoi |
|----------|----------|
| **Nombre impair de membres votants** | Garantit toujours une majorité tranchable. |
| **Arbitre `garbd` pour 2 nœuds** | Rétablit l'imparité sans serveur complet. |
| **Au moins 3 domaines de défaillance / 3ᵉ site** | Indispensable pour survivre à la perte d'un DC en multi-site. |
| **Poids (`pc.weight`) pour le multi-DC asymétrique** | Conserve le quorum au site prioritaire. |
| **Fencing pour les topologies non-Galera** | Empêche le double primaire en réplication + VIP. |
| **`pc.bootstrap` avec prudence** | À n'utiliser qu'une fois l'autre côté certainement éteint. |

---

## À retenir

- Le **split-brain** survient quand une **partition** laisse plusieurs sous-groupes se croire légitimes et écrire chacun de leur côté → **divergence irréconciliable** des données ; c'est **pire qu'une panne** (corruption silencieuse).
- Le **quorum** est la parade : seule la **majorité** (> 50 %) continue, la **minorité s'arrête**. Comme deux majorités ne coexistent pas, le split-brain devient impossible (choix **C** plutôt que **A** du CAP).
- Un **nombre impair** de nœuds garantit une majorité ; un cluster de *N* nœuds tolère **⌊(N−1)/2⌋** pannes. **2 nœuds ne tolèrent aucune panne brutale**.
- Chez Galera, la **composante primaire** (majoritaire) reste `Primary` et sert ; la minorité passe `non-Primary` et **refuse les requêtes**.
- L'**arbitre `garbd`** (votant, sans données) sécurise les configurations à 2 nœuds, à placer sur un **hôte tiers indépendant**.
- En **multi-DC**, deux sites ne suffisent pas : il faut un **3ᵉ emplacement** (au moins un arbitre) ; les **poids** (`pc.weight`) gèrent l'asymétrie.
- Après une perte totale de quorum, on rétablit le service avec **`SET GLOBAL wsrep_provider_pc_bootstrap = 1`** (en 12.3 ; l'ancienne forme `wsrep_provider_options='pc.bootstrap=YES'` est read-only → ERROR 1238) — **manœuvre à risque** réservée aux cas où l'autre côté est sûrement éteint.

La section suivante introduit l'outil qui présente aux applications un point d'accès stable et route intelligemment les requêtes : **MaxScale**.

---

⬅️ [14.2.6 — Retry des write-sets](02.6-retry-write-sets.md) · [📑 Sommaire](../SOMMAIRE.md) · [14.4 — MaxScale ➡️](04-maxscale.md)

⏭️ [MaxScale](/14-haute-disponibilite/04-maxscale.md)
