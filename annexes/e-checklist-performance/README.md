🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe E — Checklist de Performance

> ✅ Des listes de contrôle pour auditer méthodiquement la performance d'une installation MariaDB 12.3 LTS.

Cette annexe rassemble des checklists d'audit de performance : une façon structurée de passer en revue une installation MariaDB, couche par couche, pour repérer ce qui la freine. Elle prolonge les deux annexes précédentes — là où l'[Annexe C](../c-requetes-sql-reference/README.md) fournit les **requêtes de diagnostic** (les instruments) et l'[Annexe D](../d-configuration-reference/README.md) les **configurations de référence** (les cibles), l'Annexe E fournit les **listes de contrôle** qui les orchestrent : quoi vérifier, et dans quel ordre.

## Une démarche, pas une recette

Une checklist n'est utile que portée par une bonne méthode. L'optimisation est un processus **itératif et guidé par la mesure**, jamais une collection de valeurs magiques. La démarche tient en quelques principes : établir d'abord une mesure de référence, identifier le **goulot d'étranglement principal**, ne modifier **qu'une chose à la fois**, puis re-mesurer. Changer plusieurs paramètres simultanément rend impossible de savoir ce qui a aidé — ou nui. Et l'on commence par le levier le plus impactant (le plus souvent la mémoire ou un index manquant), pas par les réglages marginaux. *Voir Ch. 15.1 — Méthodologie d'optimisation.*

## Quatre couches d'audit

Les sous-sections suivent une progression, grossièrement par ordre d'impact : la **configuration** (mémoire, durabilité, E/S) constitue le socle ; l'**indexation** est de loin la cause la plus fréquente de lenteur ; les **requêtes** elles-mêmes viennent ensuite (lecture des plans, ordre des jointures, recours aux hints) ; et le **schéma** (types de données, structure) en constitue le fondement. Cet ordre est logique, mais en pratique on commence là où se trouve le goulot d'étranglement.

## Contenu de l'annexe

### E.1 — [Audit de configuration](01-audit-configuration.md)

Vérifier l'adéquation de la configuration serveur au profil de charge : dimensionnement du buffer pool, réglages de durabilité, E/S, et absence de variables dépréciées ou retirées. À confronter aux profils de l'Annexe D.

### E.2 — [Audit d'indexation](02-audit-indexation.md)

Vérifier les index : ceux qui manquent sur les colonnes filtrées, jointes ou triées ; ceux qui sont inutilisés ou redondants ; et ceux dont la sélectivité est trop faible pour servir. C'est l'audit au plus fort retour sur investissement.

### E.3 — [Audit de requêtes](03-audit-requetes.md)

Vérifier le SQL lui-même : lire les plans d'exécution (`EXPLAIN`), repérer parcours complets, mauvais ordres de jointure, tables temporaires et tris sur disque, et juger du **recours pertinent aux optimizer hints** introduits dans la série 12.x.

### E.4 — [Audit de schéma](04-audit-schema.md)

Vérifier la structure : justesse des types de données, équilibre normalisation/dénormalisation, contraintes, et partitionnement des grandes tables. Des fondations saines évitent bien des problèmes de performance en aval.

## Pour aller plus loin

Chaque audit renvoie aux chapitres qui le développent.

| Pour approfondir… | Voir |
|-------------------|------|
| Méthodologie d'optimisation | [Ch. 15.1 — Méthodologie d'optimisation](../../15-performance-tuning/README.md) |
| Configuration mémoire et tuning | [Ch. 15.2 — Configuration mémoire](../../15-performance-tuning/README.md) |
| Index et analyse des plans (`EXPLAIN`) | [Ch. 5 — Index et Performance](../../05-index-et-performance/README.md) |
| Analyse des requêtes lentes | [Ch. 15.7 — Analyse des requêtes lentes](../../15-performance-tuning/README.md) |
| Optimizer Hints | [Ch. 15.15 — Optimizer Hints](../../15-performance-tuning/README.md) |
| Types de données et conception | [Ch. 2 — Bases du SQL](../../02-bases-du-sql/README.md) |
| Requêtes de diagnostic prêtes à l'emploi | [Annexe C — Requêtes SQL de Référence](../c-requetes-sql-reference/README.md) |
| Configurations de référence | [Annexe D — Configuration de Référence](../d-configuration-reference/README.md) |

## Conventions

Chaque sous-section se présente comme une suite de points à vérifier. Les requêtes permettant de constater l'état du serveur figurent à l'Annexe C ; les valeurs et profils servant de cible, à l'Annexe D. Le fil conducteur reste le même partout : mesurer avant, modifier peu, mesurer après.

---

⬅️ [Annexe D — Configuration de Référence par Cas d'Usage](../d-configuration-reference/README.md) · 🏠 [Sommaire](../../SOMMAIRE.md) · ➡️ [E.1 — Audit de configuration](01-audit-configuration.md)

⏭️ [Audit de configuration](/annexes/e-checklist-performance/01-audit-configuration.md)
