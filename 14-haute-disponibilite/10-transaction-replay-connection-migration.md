🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.10 — Transaction Replay et Connection Migration

> **Chapitre 14 — Haute Disponibilité** · Section 14.10  
> Version de référence : **MariaDB 12.3 LTS** · MaxScale

Dernière pièce du puzzle de la haute disponibilité : rendre la bascule **réellement transparente** pour le client. Un failover (14.6) a beau promouvoir un réplica et rediriger le trafic (14.7), que deviennent les **connexions** et les **transactions en cours** au moment de la panne ? La **migration de connexion** (*Connection Migration / Session Restore*) et le **rejeu de transaction** (*Transaction Replay*) de MaxScale répondent précisément à cette question. Cette section, qui clôt le chapitre, montre comment elles s'intègrent à tout ce qui précède.

---

## 1. Le problème : connexions et transactions en cours

Le failover (14.6) et la redirection (14.7) gèrent bien les **nouvelles** connexions. Mais au moment précis de la panne du primaire :

- les **connexions existantes** vers le serveur défaillant sont **rompues** ;
- une **transaction en cours** est **interrompue** : sans aide, le client reçoit une **erreur** et **perd** son travail.

Autrement dit, sans mécanisme dédié, un failover « transparent » ne l'est **pas** pour les sessions **en vol**. Les deux fonctions de cette section comblent exactement ce manque.

---

## 2. Connection Migration / Session Restore

La première brique consiste à **faire survivre la connexion** à la perte de son serveur. MaxScale y parvient en **mémorisant l'état de la session** — l'historique des **commandes de session** (`SET`, `USE`…, déjà évoqué en 14.4.2). Lorsqu'un *backend* est perdu (panne) ou retiré (drain), MaxScale peut **reconnecter la session** à un autre serveur et **y restaurer son état**, sans que le client ne s'en aperçoive.

Deux paramètres y concourent :

- **`master_reconnection`** : autorise la **reconnexion** à un (nouveau) primaire **en cours de session** ;
- **`delayed_retry`** : **réessaie** une requête après un court délai si elle a échoué pendant la transition.

---

## 3. Transaction Replay

Le rejeu de transaction va **plus loin** : si une **transaction était ouverte** au moment de la panne, MaxScale **migre la transaction vers un serveur de remplacement** et **rejoue** ses instructions — ce qui peut **masquer complètement la panne** du primaire, **sans effet visible** pour le client. Si **aucun** serveur de remplacement ne devient disponible, la **connexion est fermée**.

Le paramètre **`transaction_replay`** (introduit dans MaxScale 2.3.0, **désactivé par défaut**) orchestre cela. Fait important : **l'activer active aussi `delayed_retry` et `master_reconnection`, et force `master_failure_mode` à `fail_on_write`**, en **surchargeant** toute valeur configurée pour ces paramètres. Le rejeu est par ailleurs **borné** par `transaction_replay_timeout` (durée maximale) et `transaction_replay_max_size` (taille maximale de la transaction).

---

## 4. Les garde-fous (limitations importantes)

Le rejeu n'est **pas magique** : il s'accompagne de protections **indispensables**.

- ⚠️ **Cohérence des résultats** : c'est le garde-fou central. Si, lors du rejeu, les **résultats diffèrent** de ceux d'origine, MaxScale **ferme la connexion** plutôt que de renvoyer des données **incohérentes**. (Cette vérification s'appuie sur un **checksum** des résultats.) C'est aussi ce qui protège contre les **fonctions non déterministes** (`NOW()`, etc.) dont le rejeu donnerait un résultat différent.
- ⚠️ **Validations implicites** : une instruction provoquant une **validation implicite** (un DDL, par exemple) pourrait, au rejeu, être **validée deux fois**. Les instructions de **gestion de transaction** (`BEGIN`, `START TRANSACTION`) font exception : elles sont **détectées** et **réinitialisent correctement** la transaction.
- **Indications de routage** : lorsque `transaction_replay` est activé, les **hints à l'intérieur d'une transaction rejouable sont ignorés** (depuis MaxScale 6.1.4), pour éviter qu'une modification interne n'affecte des serveurs **hors** de la transaction.
- **Évolution récente** : dans les versions plus anciennes, si une connexion était perdue **pendant l'exécution** d'une instruction et que le résultat avait été **partiellement** livré au client, MaxScale fermait immédiatement la session sans tenter de rejouer. **Depuis MaxScale 23.08**, cette limite ne s'applique plus si l'instruction était **dans une transaction** avec `transaction_replay` activé.

---

## 5. Transactions optimistes : une optimisation voisine

La **transaction optimiste** route **optimistiquement** une transaction vers un **réplica** tant qu'elle n'écrit pas : si la transaction ne contient **aucune** instruction modifiant les données, elle s'achève sur le réplica ; dès qu'une instruction **modifiante** apparaît, elle est **annulée sur le réplica et reprise sur le primaire**. Toutes les **limitations** du rejeu de transaction s'y appliquent également. C'est une **optimisation de montée en charge** : décharger les lectures même au sein de transactions.

> 🆕 **Changement en MaxScale 25.01.** Cette fonctionnalité, autrefois pilotée par le paramètre `optimistic_trx` de `readwritesplit`, a été **déplacée dans un filtre dédié, `OptimisticTrx`**, en MaxScale 25.01 (le paramètre a été **retiré** de `readwritesplit`). Sur les versions antérieures, on l'activait via `optimistic_trx=true` sur le service ; en 25.01, on insère désormais le **filtre `OptimisticTrx`** dans la chaîne du service.

---

## 6. Failover transparent de bout en bout

Ces fonctions sont la **dernière étape** d'une chaîne qui mobilise tout le chapitre. Vu du client, une panne du primaire se déroule idéalement ainsi :

```
Incident sur le primaire
        │
1. Détection                  (monitor — 14.6)
2. Promotion d'un réplica      (mariadbmon — 14.6)
3. Quorum / fencing            (anti-split-brain — 14.3)
4. Redirection du trafic       (routage MaxScale 14.4 / VIP 14.7)
5. Migration de connexion      (session restore — 14.10)
6. Rejeu de la transaction     (transaction replay — 14.10)
        │
   ► Le client ne voit (presque) rien
```

Sans les étapes 5 et 6, le client aurait subi une **erreur** et une **transaction perdue**, même avec un failover par ailleurs réussi. C'est ce qui distingue une HA « le service est revenu » d'une HA « l'utilisateur n'a rien remarqué ».

---

## 7. Considérations

- **Ce n'est pas illimité** : le rejeu est borné par la **taille** (`transaction_replay_max_size`) et le **délai** (`transaction_replay_timeout`), et soumis à la **cohérence des résultats** et au **non-déterminisme**.
- **Surcoût** : suivre l'état de session et tamponner la transaction a un **coût** mémoire/traitement.
- **À combiner** avec le **GTID** et la réplication **semi-synchrone** pour minimiser la perte de données lors de la promotion (voir 14.6).
- **Dépendance** : ces fonctions sont celles du routeur **`readwritesplit`** de MaxScale (14.4.2) — elles supposent donc MaxScale **et** le partage lecture/écriture.

---

## À retenir

- Un failover, même réussi, **rompt les connexions** et **interrompt les transactions en cours** ; **Connection Migration** et **Transaction Replay** comblent ce manque pour une bascule **réellement transparente**.
- La **migration de connexion** s'appuie sur l'**historique des commandes de session** (14.4.2) et sur **`master_reconnection`**/**`delayed_retry`** pour reconnecter et restaurer la session sur un autre serveur.
- **`transaction_replay`** (désactivé par défaut) **migre et rejoue** une transaction interrompue ; l'activer force aussi **`delayed_retry`**, **`master_reconnection`** et **`master_failure_mode=fail_on_write`**.
- Garde-fous : **cohérence des résultats** (fermeture de la connexion en cas d'écart, via **checksum** — protège du **non-déterminisme**), gestion des **validations implicites**, **hints ignorés** dans une transaction rejouable, bornes de **taille/délai**.
- La **transaction optimiste** route les transactions sur un réplica tant qu'elles ne modifient rien (montée en charge), avec les mêmes limitations ; en **MaxScale 25.01**, elle est passée du paramètre `optimistic_trx` de `readwritesplit` à un **filtre dédié `OptimisticTrx`**.
- Ces fonctions sont la **dernière étape** du failover transparent : détection → promotion → quorum → redirection → **migration + rejeu** ; sans elles, le client subirait une erreur.

---

## 🎓 Fin du chapitre 14

Cette section **clôt le chapitre 14 — Haute Disponibilité**. Le parcours est allé des **concepts** (14.1), au cœur synchrone de **Galera** (14.2) et au **quorum** (14.3), puis au routage avec **MaxScale** (14.4–14.5), au **failover** (14.6), à la **redirection** (14.7), à la **récupération** (14.8), aux **alternatives** (14.9) et enfin à la **transparence de bout en bout** (14.10). On retiendra le fil rouge : **HA ≠ sauvegarde**, **prévenir le split-brain par le quorum**, et viser non seulement le retour du service mais l'**absence d'impact perçu** par l'utilisateur.

---

⬅️ [14.9 — Alternatives : ProxySQL et HAProxy](09-proxysql-haproxy.md) · [📑 Sommaire](../SOMMAIRE.md) · [⬆️ Chapitre 14](README.md) · [Chapitre 15 — Performance et Tuning ➡️](../15-performance-tuning/README.md)

⏭️ [Partie 7 : Performance et Tuning (DBA Avancé)](/partie-07-performance-tuning.md)
