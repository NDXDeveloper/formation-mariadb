🔝 Retour au [Sommaire](/SOMMAIRE.md)

[← Retour au sommaire](../SOMMAIRE.md)

# 18. Fonctionnalités Avancées

> **Partie 9 — Intégration, Développement et Fonctionnalités Avancées**  
> Niveau : Avancé · Version de référence : **MariaDB 12.3 LTS**

## Introduction

Maîtriser le SQL, comprendre l'indexation, savoir gérer des transactions et programmer côté serveur : à ce stade de la formation, vous disposez de tout le nécessaire pour concevoir et exploiter une base relationnelle solide. Ce chapitre franchit une étape supplémentaire. Il rassemble les fonctionnalités qui distinguent MariaDB d'un simple moteur SQL et en font une véritable plateforme de données, capable de répondre à des besoins que l'on rencontre rarement dans les premiers projets mais qui deviennent incontournables en production : tracer l'historique complet d'une donnée, gérer des périodes de validité métier, sécuriser le stockage physique, accompagner les charges fortement concurrentes, faire évoluer un schéma sans interruption de service, ou encore alimenter des applications d'intelligence artificielle par la recherche vectorielle.

Beaucoup de ces fonctionnalités sont des implémentations de standards SQL — notamment SQL:2011 pour les aspects temporels — tandis que d'autres sont des innovations propres à MariaDB. Toutes ont en commun de résoudre des problèmes concrets sans recourir à une brique externe : ce qui était autrefois délégué à du code applicatif, à des tables d'audit maintenues à la main ou à un moteur spécialisé peut désormais être pris en charge directement par le serveur.

## Objectifs pédagogiques

À l'issue de ce chapitre, vous serez capable de :

- choisir entre une séquence et un `AUTO_INCREMENT` pour générer des identifiants, et savoir quand chaque approche est pertinente ;
- mettre en place l'historisation automatique des données avec les tables temporelles (*system-versioned*) et interroger l'état d'une table à une date passée ;
- modéliser des périodes de validité applicatives et garantir leur cohérence ;
- exploiter les colonnes générées, invisibles et dynamiques pour rendre un schéma plus expressif et plus performant ;
- réduire l'empreinte disque et protéger les données au repos via la compression et le chiffrement ;
- comprendre l'apport du Thread Pool dans les environnements à forte concurrence ;
- intégrer la recherche vectorielle de MariaDB dans une architecture d'IA / RAG ;
- faire évoluer un schéma de production avec un minimum de blocage grâce aux opérations en ligne.

## Vue d'ensemble du chapitre

Les douze sections de ce chapitre couvrent des domaines volontairement variés. On peut les regrouper en quelques grandes familles afin de mieux s'y orienter.

**Génération de valeurs.** La section consacrée aux *séquences* (18.1) présente une alternative souple à l'`AUTO_INCREMENT`, héritée de la culture Oracle et PostgreSQL, qui offre un contrôle fin sur le pas, les bornes, la mise en cache et le cyclage des valeurs générées.

**La dimension temporelle.** Deux mécanismes complémentaires permettent d'inscrire le temps au cœur du modèle de données. Les *tables temporelles à versionnement système* (18.2) historisent automatiquement chaque modification, ouvrant la voie à l'audit, à la conformité réglementaire et aux requêtes « telles que les données étaient à telle date ». Les *Application Time Period Tables* (18.3) traitent quant à elles du temps métier — durée de validité d'un contrat, d'un tarif ou d'une affectation — avec garantie de non-chevauchement.

**Des colonnes plus intelligentes.** MariaDB enrichit la notion classique de colonne. Les *colonnes générées* (18.4), virtuelles ou stockées, calculent leur valeur à partir d'autres colonnes et peuvent être indexées pour accélérer tris et regroupements. Les *colonnes invisibles* (18.5) restent absentes d'un `SELECT *` tout en demeurant accessibles explicitement, ce qui facilite les évolutions de schéma. Les *colonnes dynamiques* (18.9) offrent enfin un stockage clé-valeur au sein d'une colonne, antérieur au support natif du JSON.

**Optimisation du stockage et protection des données.** La *compression de tables* (18.6) réduit l'espace occupé et la pression sur les entrées-sorties. Le *chiffrement au repos* (18.7) protège les fichiers de données, avec une gestion des clés qui s'appuie sur des plugins comme `file_key_management` ou sur l'intégration avec un coffre externe de type HashiCorp Vault.

**Concurrence.** Le *Thread Pool avancé* (18.8) mutualise les threads du serveur pour soutenir un grand nombre de connexions simultanées sans s'effondrer sous le coût des changements de contexte.

**L'ouverture vers l'intelligence artificielle.** La section consacrée à *MariaDB Vector* (18.10) constitue l'un des temps forts du chapitre. Désormais pleinement intégrée au moteur, elle introduit un type de données `VECTOR`, un index HNSW, des fonctions de distance et de conversion, ainsi que des optimisations SIMD, et s'inscrit naturellement dans les architectures de *Retrieval-Augmented Generation* aux côtés des grands modèles de langage. La version 12.3 y apporte des optimisations supplémentaires — calcul de distance par extrapolation, exécution au plus près du moteur de stockage. 🆕

**Exploitation et évolution du schéma.** Enfin, l'*Online Schema Change* (18.11) permet de modifier la structure d'une table volumineuse en limitant le verrouillage, condition essentielle pour les bases à forte disponibilité ; et une évolution récente concernant le *nommage des contraintes de clés étrangères* (18.12) — désormais unique par table et non plus par base — rapproche MariaDB du comportement standard. 🆕

## Plan détaillé

1. [Sequences (`CREATE SEQUENCE`)](01-sequences.md)
2. [Tables temporelles (System-Versioned Tables)](02-system-versioned-tables.md)
3. [Application Time Period Tables](03-application-time-period-tables.md)
4. [Colonnes virtuelles et générées](04-virtual-generated-columns.md)
5. [Invisible columns](05-invisible-columns.md)
6. [Compression de tables](06-compression-tables.md)
7. [Encryption at rest](07-encryption-at-rest.md)
8. [Thread Pool avancé](08-thread-pool-avance.md)
9. [Dynamic columns](09-dynamic-columns.md)
10. [MariaDB Vector : Recherche vectorielle pour l'IA/RAG](10-mariadb-vector.md)
11. [Online Schema Change (ALTER TABLE non-bloquant)](11-online-schema-change.md)
12. [Contraintes FK : noms uniques par table seulement](12-fk-constraint-names.md) 🆕

## Prérequis

Ce chapitre suppose une bonne maîtrise des fondamentaux SQL (Parties 1 et 2), des notions d'indexation et de transactions (Partie 3), ainsi qu'une familiarité avec les moteurs de stockage et la programmation serveur (Partie 4). Pour la partie consacrée à la recherche vectorielle, des notions générales sur les *embeddings* et les modèles de langage faciliteront la compréhension, sans être indispensables.

## À propos des versions

Cette formation prend pour référence **MariaDB 12.3 LTS** (disponible depuis fin mai 2026, supportée jusqu'en juin 2029). Dans ce chapitre, les éléments signalés par 🆕 correspondent aux nouveautés introduites dans la série 12.x. Plusieurs fonctionnalités présentées ici — MariaDB Vector, les Application Time Period Tables ou l'Online Schema Change notamment — sont issues de la lignée 11.8 et constituent aujourd'hui un socle mature et stabilisé.

⏭️ [Sequences (CREATE SEQUENCE)](/18-fonctionnalites-avancees/01-sequences.md)
