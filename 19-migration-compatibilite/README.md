🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19. Migration et Compatibilité

> **Partie 10 — Migration, Compatibilité et Architectures**  
> Niveau : DBA / Architecte · Parcours concernés : Administrateur/DBA

---

## Introduction

La migration figure parmi les opérations les plus sensibles du cycle de vie d'une base de données. Contrairement à l'ajout d'un index ou à l'ajustement d'un paramètre, elle touche directement aux données de production, impose souvent une fenêtre d'indisponibilité et mobilise plusieurs équipes (DBA, développeurs, exploitation, métier). Un échec ne se traduit pas seulement par une dégradation de performance : il peut entraîner une perte de données, une interruption de service prolongée ou des incompatibilités applicatives bloquantes. C'est pourquoi une migration réussie repose autant sur la préparation, les tests et le plan de repli que sur la maîtrise technique des outils.

Ce chapitre aborde la migration sous ses deux grandes formes, qui posent des problématiques distinctes :

La **migration hétérogène** consiste à faire venir vers MariaDB des données et des applications qui reposaient sur un autre SGBD : MySQL, Oracle, SQL Server ou PostgreSQL. Ici, la difficulté principale est la **compatibilité** : différences de types de données, dialectes SQL divergents, procédures stockées écrites dans des langages propriétaires (PL/SQL, T-SQL), comportements implicites variables d'un moteur à l'autre. MariaDB facilite ce travail grâce à une forte compatibilité native avec MySQL — dont elle est issue par un fork en 2009, avant une divergence progressive — et grâce à des couches de compatibilité dédiées, notamment le mode Oracle (`SQL_MODE=ORACLE`) et ses constructions de type PL/SQL.

La **migration homogène** consiste à passer d'une version de MariaDB à une autre, par exemple de la **11.8 LTS** vers la **12.3 LTS**. La compatibilité des données y est garantie, mais l'attention se déplace vers les **changements de comportement** introduits entre versions, le choix du chemin de mise à jour (*upgrade path*) et la réduction de la durée d'interruption. La 12.3 consolide la série rolling 12.0 → 12.2 et introduit quelques modifications qui ont un impact sur ce type de migration : suppression de certaines variables système, évolution de la portée des noms de contraintes de clés étrangères, et changement de packaging de Galera. (L'isolation par *snapshot*, fréquemment citée comme le changement phare de la 12.3, est en réalité déjà active par défaut dès la 11.8 : ce n'est donc pas un changement propre au passage 11.8 → 12.3.) Une section entière (19.10) est consacrée à ce passage.

La notion de **compatibilité** traverse donc tout le chapitre : compatibilité entre SGBD différents, compatibilité applicative à préserver pendant et après la bascule, et compatibilité ascendante entre versions successives de MariaDB.

---

## Objectifs pédagogiques

À l'issue de ce chapitre, vous serez capable de :

- **Comprendre les enjeux et les risques** propres à toute opération de migration, et structurer une démarche méthodique (audit préalable, tests, repli).
- **Migrer une base depuis MySQL** vers MariaDB en identifiant les points de compatibilité et les différences notables entre les deux produits.
- **Aborder une migration depuis Oracle, SQL Server ou PostgreSQL**, en vous appuyant sur les couches de compatibilité de MariaDB et sur les outils adaptés.
- **Choisir une stratégie de versions cohérente** en distinguant les éditions LTS des éditions rolling, selon le profil de l'application et les contraintes d'exploitation.
- **Maîtriser les chemins et outils de mise à jour** (`mariadb-upgrade`, mise à jour *in-place* versus logique) et leurs implications respectives.
- **Sécuriser la bascule** par des tests de compatibilité, une stratégie de rollback et un plan de contingence.
- **Mettre en œuvre des migrations sans interruption** (*zero-downtime*) lorsque la continuité de service l'exige.
- **Conduire spécifiquement une migration 11.8 → 12.3** en anticipant les changements de comportement de la version cible.

---

## Plan du chapitre

Le chapitre est organisé en cinq grands ensembles, du plus général au plus spécifique :

**Migrer vers MariaDB depuis un autre SGBD (19.1 – 19.2).**
On traite d'abord la migration depuis MySQL, cas le plus courant et le plus simple en raison de la proximité des deux moteurs (19.1), puis la migration depuis d'autres systèmes — Oracle, SQL Server, PostgreSQL — où les écarts de dialecte et de fonctionnalités sont plus marqués (19.2).

**Stratégie de versions et mises à jour (19.3 – 19.4).**
On clarifie le choix entre éditions LTS et rolling (19.3), avant d'examiner les stratégies de mise à jour et les chemins d'*upgrade* possibles, ainsi que l'outil `mariadb-upgrade` et le choix entre mise à jour en place ou logique (19.4).

**Sécuriser la migration (19.5 – 19.7).**
Cette partie couvre la préservation de la compatibilité applicative (19.5), la mise en place de tests de compatibilité (19.6), puis les mécanismes de rollback et de contingence indispensables pour pouvoir revenir en arrière sans dommage (19.7).

**Migrations avancées (19.8 – 19.9).**
On y aborde les migrations sans interruption de service (19.8) et le cas particulier des *System-Versioned Tables*, dont le format de versionnement temporel mérite une attention spécifique lors d'une bascule (19.9).

**Le cas concret 11.8 → 12.3 (19.10).**
Le chapitre se conclut par une section dédiée au passage de la LTS précédente vers la LTS actuelle, recensant les changements de comportement à anticiper : variables système retirées, portée des noms de contraintes de clés étrangères et packaging de Galera (l'isolation par snapshot, elle, étant déjà active par défaut dès la 11.8).

---

## Prérequis

Ce chapitre suppose une bonne maîtrise des fondamentaux abordés dans les parties précédentes. Une migration ne s'improvise pas : elle s'appuie sur la connaissance des moteurs de stockage (chapitre 7), des mécanismes de réplication (chapitre 13) et de sauvegarde/restauration (chapitre 12), ainsi que des principes d'administration et de configuration (chapitre 11). Les questions de sécurité et de gestion des utilisateurs (chapitre 10) sont également mobilisées, notamment lors de la migration des comptes et de leurs privilèges entre systèmes ou entre versions.

---

## Repères de version

Cette formation prend pour socle **MariaDB 12.3 LTS** (GA = 12.3.2, fin mai 2026, supportée jusqu'en juin 2029). La **11.8 LTS** (juin 2025, support jusqu'en 2028), largement déployée, sert de point de référence pour les scénarios de migration homogène. La série **13.x** ouvre le cycle suivant : sa première version, **13.0**, est en phase de préversion puis de version candidate (RC publiée fin mai 2026), sa GA n'étant pas encore parue à la date de rédaction (juin 2026). Pour le positionnement détaillé des versions et la politique de support — LTS annuelle supportée trois ans depuis la 11.8, éditions rolling trimestrielles supportées jusqu'à la version suivante — se reporter à l'Annexe G.

⏭️ [Migration depuis MySQL](/19-migration-compatibilite/01-migration-depuis-mysql.md)
