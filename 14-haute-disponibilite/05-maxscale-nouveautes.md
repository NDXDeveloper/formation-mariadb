🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.5 — MaxScale : fonctionnalités récentes (Workload Capture/Replay, Diff Router)

> **Chapitre 14 — Haute Disponibilité** · Section 14.5  
> Version de référence : **MariaDB 12.3 LTS** · MaxScale **25.01**

Au-delà de ses fonctions cœur (14.4), MaxScale s'est enrichi de fonctionnalités tournées vers un besoin opérationnel récurrent : **valider un changement** (montée de version, modification de configuration) **sans risquer la production**. La version **MaxScale 25.01** introduit deux outils complémentaires pour cela : la **capture et le rejeu de charge** (*Workload Capture/Replay*) et le **Diff Router**. Cette page les présente d'ensemble ; leurs mécanismes sont détaillés dans les sous-sections.

---

## 1. Le contexte : tester avec une charge réelle, sans risque

Comment être sûr qu'une **mise à jour majeure** (par exemple 11.8 → 12.3, voir 19.10) ou un changement de configuration ne va ni casser, ni ralentir une application en production ? Les *benchmarks synthétiques* ne reproduisent jamais fidèlement les **nuances d'une charge réelle**. La réponse apportée par MaxScale 25.01 consiste à **travailler à partir du trafic de production lui-même** : la capture et le rejeu de charge, ainsi que le Diff Router, permettent de réaliser des tests de performance sans affecter la production.

Ces outils s'inscrivent donc dans la logique de **dé-risquage des changements**, en lien direct avec les stratégies de mise à jour (chapitre 19).

---

## 2. MaxScale 25.01 : ce qui change

Le Diff Router permet de comparer le comportement de deux serveurs, et un ensemble de composants permet de capturer une charge réelle puis de la rejouer ultérieurement. MaxScale 25.01 ajoute ainsi des fonctionnalités avancées comme la capture et le rejeu de charge, qui rendent les montées de version plus sereines. Cette version fait partie de la plateforme MariaDB 2025.

> 🔑 **Licence et essai.** MaxScale 25.01 est diffusé sous une **nouvelle licence** ; un « MaxScale Trial » gratuit, en version limitée dans le temps et en capacité, donne accès à l'ensemble des fonctionnalités, dont le Diff Router et la capture/rejeu de charge. On retrouve ici l'importance de **vérifier les conditions de licence** avant un usage en production, déjà soulignée en 14.4.

---

## 3. Workload Capture et Replay (WCAR) — en bref

L'idée : **enregistrer** le trafic réel qui passe par MaxScale, puis le **rejouer** plus tard sur un autre système. Ce filtre capture les sessions clientes pour créer des bancs d'essai et des environnements de test réalistes fondés sur les charges de production ; il évite d'avoir à construire des générateurs de trafic explicites, souvent coûteux et complexes à maintenir.

- la **capture** (détaillée en **14.5.1**) enregistre le trafic — elle peut être activée dynamiquement sur un service de routage existant, sans redémarrage de MaxScale, et requiert que l'instance MariaDB ait la journalisation binaire activée (log-bin=1) ;
- le **rejeu** (détaillé en **14.5.2**) rejoue la charge enregistrée contre un système cible.

> ℹ️ Point d'attention important pour le rejeu : les serveurs de capture et de rejeu doivent utiliser la même distribution Linux et la même architecture CPU, les fichiers de capture étant liés à l'architecture d'origine.

---

## 4. Diff Router — en bref

Le **Diff Router** compare **en direct** le comportement de **deux serveurs**. Il envoie la charge à la fois au serveur en cours d'utilisation — appelé « main » — et à un autre serveur — appelé « other » — dont on veut évaluer le comportement. Comme les réponses de « main » sont renvoyées au client sans attendre celles de « other », l'impact sur les performances est minimisé. Pendant son fonctionnement, Diff collecte des données d'histogramme de latence permettant de comparer le comportement de « main » et de « other », par exemple avant une montée de version.

Ce routeur est détaillé en **14.5.3**.

---

## 5. Comment ces outils s'articulent

Capture/Replay et Diff Router visent le même objectif — **dé-risquer un changement** — mais selon deux approches :

| Approche | Capture & Replay (14.5.1–14.5.2) | Diff Router (14.5.3) |
|----------|----------------------------------|----------------------|
| **Principe** | Enregistrer le trafic réel, puis le **rejouer** sur un autre système | Envoyer le trafic **en parallèle** à deux serveurs et **comparer** |
| **Moment** | **Différé** (hors ligne) | **En direct** (en ligne) |
| **Impact production** | Capture passive ; rejeu sur un système de test | « main » répond au client sans attendre « other » → impact **minimal** |
| **Usage type** | Valider une nouvelle version avec une charge **réaliste** | **Comparer** le comportement de deux versions avant bascule |

Les deux sont **complémentaires** : on peut comparer en direct (Diff) **et** rejouer une charge capturée sur un banc de test (Capture/Replay).

---

## 6. Cas d'usage en HA et migration

- **Valider une mise à jour majeure** (par exemple 11.8 → 12.3, voir 19.10) avec une charge **issue de la production**.
- **Évaluer** un changement de **configuration** ou de **matériel** avant de le déployer.
- Identifier des **requêtes lentes** sous une charge réelle.
- **Détecter des régressions** de comportement ou de performance entre deux versions.

---

## 7. Plan de la section

1. **[14.5.1 — Workload Capture](05.1-workload-capture.md)** : enregistrer le trafic de production réel.
2. **[14.5.2 — Workload Replay](05.2-workload-replay.md)** : rejouer la charge capturée sur un système cible.
3. **[14.5.3 — Diff Router](05.3-diff-router.md)** : comparer en direct le comportement de deux serveurs.

---

## À retenir

- **MaxScale 25.01** introduit des outils pour **valider un changement sans risquer la production** : **Workload Capture/Replay** et **Diff Router**.
- La **capture/rejeu (WCAR)** enregistre le **trafic réel** puis le **rejoue** sur un autre système — sans générateur de trafic synthétique ; la capture s'active **dynamiquement** (sans redémarrage) et requiert le **binlog** (`log-bin=1`).
- Le **Diff Router** envoie la charge à **deux serveurs** (« main » et « other ») et **compare** leurs latences, en renvoyant les réponses de « main » **sans attendre** « other » (impact minimal).
- Les deux approches sont **complémentaires** : rejeu **différé** sur banc de test vs comparaison **en direct**.
- Usage phare : **dé-risquer une montée de version** (par exemple 11.8 → 12.3, voir 19.10) avec une charge réaliste.
- ⚠️ MaxScale 25.01 est sous **nouvelle licence** (un **Trial** gratuit existe) ; vérifier les conditions pour la production (voir 14.4).

La sous-section suivante détaille le premier de ces outils : la **capture de charge** (*Workload Capture*).

---

⬅️ [14.4.4 — Database Firewall](04.4-database-firewall.md) · [📑 Sommaire](../SOMMAIRE.md) · [14.5.1 — Workload Capture ➡️](05.1-workload-capture.md)

⏭️ [Workload Capture](/14-haute-disponibilite/05.1-workload-capture.md)
