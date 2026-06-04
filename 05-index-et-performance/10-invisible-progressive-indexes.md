🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.10 — Invisible indexes et Progressive indexes

> **Chapitre 5 — Index et Performance** · Section 5.10  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

Ce chapitre se clôt sur la dimension **opérationnelle** des index : comment les **tester** et les **déployer** sans risque pour la production. Le cœur du sujet est une fonctionnalité simple mais précieuse — l'index que l'on peut rendre **invisible à l'optimiseur sans le supprimer**. On en profitera aussi pour clarifier ce que recouvre l'expression « progressive indexes ».

---

## Les « invisible indexes » en MariaDB : les index ignorés

> **Une précision de nommage.** L'expression « *invisible index* » est le terme employé par **MySQL**. En **MariaDB**, la fonctionnalité — disponible depuis la version **10.6** — porte le nom d'**index ignoré** (*ignored index*) et utilise une syntaxe propre. Le principe est le même ; seul le vocabulaire diffère.

Un **index ignoré** est un index **toujours présent et maintenu**, mais que **l'optimiseur n'utilise pas** pour bâtir ses plans d'exécution. Il reste à jour à chaque écriture, mais devient *invisible* du point de vue de la planification des requêtes — comme s'il n'existait pas, sans pour autant être supprimé.

### Syntaxe

Rendre un index ignoré (ou le réactiver) est un simple **changement de métadonnées** : l'opération est **instantanée**, sans reconstruction.

```sql
-- Rendre un index ignoré
ALTER TABLE commandes ALTER INDEX idx_statut IGNORED;

-- Le réactiver, tout aussi instantanément
ALTER TABLE commandes ALTER INDEX idx_statut NOT IGNORED;
```

On peut aussi créer un index **d'emblée ignoré** — il sera construit et maintenu, mais pas encore exploité :

```sql
CREATE INDEX idx_date ON commandes (date_commande) IGNORED;
```

L'état d'un index se lit dans la colonne **`Ignored`** (`YES`/`NO`) de `SHOW INDEX` ([section 5.4](04-creation-gestion-index.md)) :

```sql
SHOW INDEX FROM commandes;
```

> **Exception :** la **clé primaire ne peut pas** être rendue ignorée.

### Une nuance essentielle

Un index ignoré n'est **pas** un index « désactivé » au sens d'inerte. Deux conséquences à garder à l'esprit :

- il continue d'être **maintenu** à chaque `INSERT`/`UPDATE`/`DELETE` — son **coût en écriture demeure** ;
- s'il s'agit d'un index **`UNIQUE`**, il **continue de faire respecter sa contrainte** d'unicité.

« Ignoré » signifie donc uniquement : *non considéré par l'optimiseur pour les plans de requête*. Rien de plus.

---

## Cas d'usage des index ignorés

### Tester sans risque la suppression d'un index

C'est l'usage phare, déjà évoqué aux [sections 5.4](04-creation-gestion-index.md) et [5.5](05-strategies-indexation.md). Avant de supprimer un index suspecté inutile, on le rend **ignoré** et l'on observe la production :

```sql
-- 1) Index suspecté inutile (repéré via le Performance Schema, cf. 5.5 et 15.8)
-- 2) Le rendre ignoré et surveiller
ALTER TABLE commandes ALTER INDEX idx_suspect IGNORED;

-- 3a) Aucune régression → suppression définitive
ALTER TABLE commandes DROP INDEX idx_suspect;

-- 3b) Régression constatée → réactivation IMMÉDIATE, sans coût de reconstruction
ALTER TABLE commandes ALTER INDEX idx_suspect NOT IGNORED;
```

L'intérêt est décisif : en cas de problème, on **revient en arrière instantanément**, là où recréer un index supprimé sur une grosse table prendrait du temps.

### Déployer progressivement un nouvel index

Symétriquement, on peut créer un nouvel index en l'état **ignoré**, le laisser se construire et se maintenir sans qu'il n'influence encore les plans, puis le **passer en `NOT IGNORED`** au moment choisi — par exemple lors d'une fenêtre de faible activité. On contrôle ainsi **précisément l'instant** où l'index commence à modifier le comportement de l'optimiseur.

### Diagnostiquer un plan d'exécution

Rendre un index temporairement ignoré permet aussi de **comparer les plans** avec et sans cet index (via `EXPLAIN`, [section 5.7](07-analyse-plans-execution.md)), sans risquer de le perdre.

---

## « Progressive indexes » : mise au point

L'expression « *progressive index* » mérite une clarification, car **MariaDB ne propose aucune fonctionnalité portant ce nom**. Elle renvoie plutôt à un **concept** général, qui recouvre deux idées distinctes :

1. **L'indexation incrémentale / adaptative** — l'idée qu'une base construise ou affine ses index **progressivement**, au fil des requêtes (un sujet largement exploré dans la recherche, sous des noms comme *adaptive indexing* ou *database cracking*). **Ce n'est pas un mécanisme du cœur de MariaDB.** La seule adaptation automatique qu'opère le moteur est l'**Adaptive Hash Index** ([section 5.2.2](02.2-hash.md)) — un cache mémoire interne, et non une gestion automatique des index de l'utilisateur.

2. **Le déploiement progressif** d'un index en production — et celui-là, MariaDB le permet pleinement, à partir des outils déjà vus : la **création d'index en ligne** non bloquante (`ALGORITHM = INPLACE, LOCK = NONE`, [section 5.4](04-creation-gestion-index.md) et [section 18.11](../18-fonctionnalites-avancees/11-online-schema-change.md)) combinée aux **index ignorés** décrits ci-dessus.

En pratique, c'est donc cette **seconde acception** — le déploiement maîtrisé et progressif — qui est opérationnelle dans MariaDB.

---

## Une démarche de déploiement progressif et sûr

En combinant les techniques du chapitre, on obtient un **cycle de vie d'index** robuste :

1. **Identifier** le besoin via `EXPLAIN`/`ANALYZE` ([5.7](07-analyse-plans-execution.md)) et le *slow query log*.
2. **Créer** l'index **sans bloquer** la production (`ALGORITHM = INPLACE, LOCK = NONE`, [5.4](04-creation-gestion-index.md)), éventuellement en l'état **`IGNORED`**.
3. **Activer** l'index (`NOT IGNORED`) au moment opportun et **mesurer** son effet.
4. Périodiquement, **repérer** les index inutiles ([5.5](05-strategies-indexation.md), [15.8](../15-performance-tuning/08-performance-schema-sys.md)), les **rendre ignorés** pour valider sans risque, puis les **supprimer**.

Ce cycle referme la boucle ouverte par la stratégie d'indexation de la [section 5.5](05-strategies-indexation.md) : indexer **juste**, et faire **évoluer** les index en toute sécurité.

---

> ### 📝 À retenir  
>  
> - En MariaDB (depuis 10.6), un « invisible index » s'appelle un **index ignoré** : il est **maintenu** mais **non utilisé par l'optimiseur** ; `ALTER TABLE … ALTER INDEX … IGNORED | NOT IGNORED` est un changement **instantané** (la clé primaire ne peut être ignorée).  
> - Un index ignoré **coûte toujours en écriture** et, s'il est `UNIQUE`, **fait toujours respecter** son unicité : seule son utilisation par l'optimiseur est suspendue.  
> - Usage phare : **tester la suppression** d'un index sans risque (revenir en arrière est instantané), et **déployer progressivement** un nouvel index.  
> - **« Progressive index » n'est pas une fonctionnalité MariaDB** : le concept d'indexation adaptative automatique n'existe pas dans le cœur du serveur ; en revanche, le **déploiement progressif** d'index est réalisable via la **création en ligne** + les **index ignorés**.  
> - Ces outils dessinent un **cycle de vie d'index** sûr, complément opérationnel de la stratégie d'indexation (5.5).

---

## 🧭 Navigation

- ⬅️ Section précédente : [5.9 Index covering et index-only scans](09-index-covering.md)
- ➡️ Chapitre suivant : [6. Transactions et Concurrence](../06-transactions-et-concurrence/README.md)
- 📂 Chapitre : [5. Index et Performance](README.md)
- 🏠 [Retour au sommaire](../SOMMAIRE.md)

⏭️ [Transactions et Concurrence](/06-transactions-et-concurrence/README.md)
