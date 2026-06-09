🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.4 Architecture multi-tenant

> **Chapitre 20 — Cas d'Usage et Architectures**  
> Niveau : Avancé  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

La section §20.2 a exploré une première forme de cloisonnement : celui des **services** d'une application. L'architecture **multi-tenant** pose une question voisine, mais sous un autre angle : comment cloisonner non plus des services, mais des **clients** ? Une même application doit servir de nombreux clients tout en garantissant que les données de l'un ne soient jamais visibles par un autre.

Cette problématique est au cœur du modèle **SaaS** (*Software as a Service*) : une seule application, hébergée par le fournisseur, sert simultanément des dizaines, des centaines ou des milliers d'organisations clientes. La conception de la base de données y est déterminante, car c'est elle qui porte l'essentiel de l'isolation entre clients.

Cette section pose le cadre général : elle définit le multi-tenant, met en évidence la tension fondamentale entre isolation et mutualisation, énumère les critères de décision, et présente les trois grands patterns qui seront détaillés dans les sous-sections suivantes.

---

## Qu'est-ce que le multi-tenant ?

Le **multi-tenant** (multi-locataire) désigne une architecture où une **même instance** d'application et d'infrastructure sert plusieurs **locataires** (*tenants*) — chaque locataire étant typiquement une organisation cliente, avec ses propres utilisateurs et ses propres données.

L'exigence cardinale est l'**isolation** : un locataire ne doit en aucun cas accéder aux données d'un autre. Cette isolation doit être garantie à la fois sur le plan de la sécurité (étanchéité), de la confidentialité (conformité réglementaire) et, idéalement, des performances (les traitements d'un locataire ne devant pas dégrader ceux des autres).

Le modèle opposé est le **mono-tenant** (*single-tenant*), où chaque client dispose d'un déploiement entièrement dédié et isolé. Le mono-tenant offre l'isolation maximale, mais au prix d'un coût et d'une charge d'exploitation qui croissent linéairement avec le nombre de clients. Le multi-tenant cherche à mutualiser l'infrastructure pour réduire ce coût — d'où une tension permanente entre mutualisation et isolation.

---

## La tension fondamentale : isolation vs densité

Toute la conception d'une architecture multi-tenant se ramène à un arbitrage entre deux objectifs en partie contradictoires :

- **L'isolation** : plus les données des locataires sont séparées, meilleures sont la sécurité, la confidentialité, la maîtrise des performances et la facilité des opérations par locataire (sauvegarde, restauration, migration ciblées).
- **La densité** (et donc l'efficience économique) : plus les locataires partagent les mêmes ressources, plus le coût par locataire diminue et plus l'exploitation se simplifie à grande échelle.

Renforcer l'isolation tend à réduire la densité, et inversement. C'est cet arbitrage — le même esprit de compromis qui traverse tout le chapitre — qui distingue les trois patterns présentés plus loin : chacun privilégie un point différent sur l'axe **isolation ↔ densité**.

---

## Les critères de choix

Le bon pattern dépend du contexte. Plusieurs critères entrent en jeu :

- **Isolation et conformité** : certains secteurs (santé, finance) ou certaines réglementations (protection des données personnelles, résidence géographique des données) imposent une séparation forte, voire un placement précis des données par locataire.
- **Densité et coût** : combien de locataires peut-on héberger par serveur ? À grande échelle, la densité conditionne directement la rentabilité.
- **Voisin bruyant** (*noisy neighbor*) : dans une infrastructure mutualisée, un locataire très actif peut consommer une part disproportionnée des ressources et pénaliser les autres. Une isolation plus forte atténue ce risque.
- **Personnalisation** : si certains locataires nécessitent des variations de schéma qui leur sont propres, une séparation par base ou par schéma facilite ces adaptations, là où un schéma entièrement partagé les rend difficiles.
- **Passage à l'échelle** : le nombre de locataires attendu, et son rythme de croissance, oriente le choix — gérer des milliers de bases distinctes n'a pas les mêmes implications que gérer une base unique très peuplée.
- **Complexité opérationnelle** : sauvegardes, restaurations, migrations de schéma et supervision se font soit par locataire, soit globalement, avec des coûts très différents selon le pattern.
- **Intégration et suppression** (*onboarding / offboarding*) : créer ou supprimer un locataire est trivial dans certains modèles, plus lourd dans d'autres.

Aucun pattern n'optimise tous ces critères à la fois : il s'agit de pondérer ceux qui comptent le plus pour le projet.

---

## Trois patterns, un continuum

Les sous-sections suivantes détaillent les trois grandes approches du multi-tenant, qui jalonnent le continuum isolation ↔ densité :

| Pattern | Section | Principe | Isolation | Densité |
|---------|---------|----------|-----------|---------|
| **Base par locataire** | §20.4.1 | Une base de données dédiée à chaque locataire | Forte | Faible |
| **Schéma par locataire** | §20.4.2 | Un schéma (jeu de tables) par locataire, sur une infrastructure mutualisée | Moyenne | Moyenne |
| **Schéma partagé** | §20.4.3 | Tous les locataires dans les mêmes tables, distingués par une colonne discriminante | Faible | Forte |

À mesure que l'on descend dans ce tableau, l'isolation décroît et la densité augmente. Le premier modèle se rapproche du mono-tenant (chaque locataire isolé), tandis que le dernier mutualise tout, au prix d'une isolation reposant entièrement sur l'application.

> **Une particularité MariaDB à garder à l'esprit.** Dans MariaDB (comme dans MySQL), `CREATE SCHEMA` est un **synonyme** de `CREATE DATABASE` : un « schéma » y est équivalent à une base, sans niveau intermédiaire de « schéma à l'intérieur d'une base » comme on en trouve dans PostgreSQL ou SQL Server. La distinction entre « base par locataire » et « schéma par locataire » tient donc, sous MariaDB, davantage à la **topologie de déploiement** (instances dédiées ou mutualisées, regroupement des bases) qu'à deux mécanismes fondamentalement différents. Les sous-sections §20.4.1 et §20.4.2 précisent cette nuance.

---

## Approches hybrides

En pratique, beaucoup de plateformes SaaS adoptent un modèle **hybride** plutôt qu'un pattern unique. Une approche **par paliers** (*tiered*) est courante : les petits locataires, nombreux, sont regroupés dans un schéma partagé pour maximiser la densité, tandis que les locataires importants ou « premium » — qui exigent une isolation forte, des performances garanties ou une conformité particulière — se voient attribuer une base dédiée.

Cette combinaison permet d'optimiser le coût global tout en répondant aux exigences spécifiques de chaque catégorie de clients. Le choix d'un pattern n'est donc pas forcément exclusif : il peut varier selon le segment de clientèle.

---

## MariaDB et le multi-tenant

Les trois patterns sont réalisables avec MariaDB, mais chacun mobilise des mécanismes différents pour garantir l'isolation :

- **Isolation par les privilèges** : pour les modèles « base par locataire » et « schéma par locataire », l'étanchéité s'appuie sur des utilisateurs dédiés dont les droits sont strictement limités à la base du locataire (`GRANT ... ON locataire_db.*`), exactement comme pour le cloisonnement par service vu en §20.2.1.
- **Isolation applicative pour le schéma partagé** : contrairement à PostgreSQL, MariaDB **ne propose pas de sécurité au niveau ligne** (*row-level security*) native. Dans le modèle à schéma partagé, l'isolation entre locataires repose donc essentiellement sur la **logique applicative** — un filtrage systématique sur la colonne discriminante (`WHERE tenant_id = ?`) — éventuellement renforcé par des vues. Ce point, détaillé en §20.4.3, est un facteur de risque à ne pas négliger.
- **Mise à l'échelle** : pour répartir un grand nombre de locataires sur plusieurs serveurs, le moteur **Spider** (§7.10.3, §15.11) ou un placement par locataire permettent une distribution horizontale.
- **Opérations par locataire** : la sauvegarde et la restauration ciblées d'un locataire sont triviales avec une base dédiée (`mariadb-dump` par base, chapitre 12), mais nettement plus complexes dans un schéma partagé, où les données d'un locataire sont entremêlées à celles des autres.
- **Gestion des connexions** : un modèle « base par locataire » très peuplé peut multiplier les connexions et les pools ; le *connection pooling* (chapitre 17) et le Thread Pool (chapitre 11) deviennent alors des leviers importants.

---

## Ce que couvrent les sections suivantes

Les trois sous-sections approfondissent chacun des patterns :

- **§20.4.1 — Database per tenant** : l'isolation maximale, une base par locataire.
- **§20.4.2 — Schema per tenant** : un schéma par locataire sur une infrastructure mutualisée, et ce que cela recouvre concrètement sous MariaDB.
- **§20.4.3 — Shared schema avec discriminateur** : la densité maximale, tous les locataires dans les mêmes tables, et les enjeux d'isolation applicative qui en découlent.

À l'issue de ces sections, le lecteur disposera d'une grille claire pour arbitrer entre isolation et densité selon son contexte SaaS, avant d'aborder les questions de répartition géographique des données (§20.5) et de déploiement multi-cloud (§20.6).

---

## Navigation

⬅️ Section précédente : [20.3 Data warehousing avec ColumnStore](03-data-warehousing-columnstore.md)  
➡️ Section suivante : [20.4.1 Database per tenant](04.1-database-per-tenant.md)  
⬆️ Retour au [sommaire](../SOMMAIRE.md)

⏭️ [Database per tenant](/20-cas-usage-architectures/04.1-database-per-tenant.md)
