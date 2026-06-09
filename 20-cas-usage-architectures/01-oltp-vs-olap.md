🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.1 OLTP vs OLAP

> **Chapitre 20 — Cas d'Usage et Architectures**  
> Niveau : Avancé  
> Version de référence : **MariaDB 12.3 LTS**

---

## Introduction

Toute réflexion architecturale autour d'une base de données commence par une question simple mais déterminante : **à quoi sert cette base ?** Sert-elle à enregistrer en continu les opérations courantes d'une application (passer une commande, débiter un compte, réserver une place), ou bien à analyser de grandes quantités de données accumulées pour en tirer des indicateurs et des tendances ?

Ces deux finalités correspondent à deux familles de charges radicalement différentes, désignées par deux acronymes hérités de l'histoire des bases de données : **OLTP** (*Online Transaction Processing*) et **OLAP** (*Online Analytical Processing*). Comprendre leur nature et leurs contraintes est la première étape de toute conception, car cette distinction conditionne ensuite le choix du moteur de stockage, la modélisation du schéma, la stratégie d'indexation et le dimensionnement matériel.

Cette section pose les fondations conceptuelles. Les sections suivantes du chapitre (entrepôts de données, architectures distribuées, cas d'usage IA) reposent toutes, directement ou indirectement, sur cette dichotomie de départ.

---

## OLTP : le traitement transactionnel

L'**OLTP** désigne les systèmes qui gèrent les transactions opérationnelles d'une organisation, en temps réel et en grand nombre. C'est le cœur des applications du quotidien : sites de commerce électronique, applications bancaires, systèmes de réservation, CRM, gestion de stock.

Une charge OLTP se caractérise par :

- **De nombreuses opérations courtes et concurrentes** : un système OLTP peut traiter des milliers de transactions par seconde, chacune touchant un faible nombre de lignes.
- **Un fort taux de concurrence** : de nombreux utilisateurs ou processus accèdent simultanément à la base.
- **Des accès ciblés** : les requêtes visent le plus souvent une ligne ou un petit ensemble de lignes, repérées par clé primaire ou via un index.
- **Un mélange de lectures et d'écritures** : `INSERT`, `UPDATE` et `DELETE` y sont fréquents, ce qui place les propriétés **ACID** et le contrôle de concurrence au centre des préoccupations.
- **Une exigence de latence faible** : on attend des temps de réponse de l'ordre de la milliseconde.
- **Des données « vivantes »** : la base reflète l'état courant du système (le stock actuel, le solde du compte à l'instant T).

Dans MariaDB, le moteur de référence pour l'OLTP est **InnoDB** (moteur par défaut). Son verrouillage au niveau ligne, son modèle MVCC, sa gestion transactionnelle ACID et ses index B-Tree sont précisément conçus pour ce type de charge.

```sql
-- OLTP : accès ciblé à une commande, par clé primaire
SELECT * FROM commandes WHERE commande_id = 84213;

-- OLTP : une transaction courte qui modifie peu de lignes
START TRANSACTION;
  INSERT INTO commandes (client_id, montant) VALUES (512, 149.90);
  UPDATE stock SET quantite = quantite - 1 WHERE produit_id = 77;
COMMIT;
```

---

## OLAP : le traitement analytique

L'**OLAP** désigne à l'inverse les systèmes destinés à l'analyse de grands volumes de données, généralement historiques : informatique décisionnelle (BI), reporting, tableaux de bord, analyse de tendances. L'objectif n'est plus d'enregistrer des opérations, mais d'**interroger massivement** des données déjà collectées.

Une charge OLAP se caractérise par :

- **Un faible nombre de requêtes, mais complexes** : agrégations, regroupements et jointures portant sur des millions, voire des milliards de lignes.
- **Une dominante en lecture** : les données sont alimentées par lots (chargements périodiques), puis interrogées intensivement ; les mises à jour ponctuelles sont rares.
- **Des balayages de grande ampleur** : une requête analytique lit fréquemment une part importante d'une table, mais sur un petit nombre de colonnes seulement.
- **Une concurrence plus modérée** : les utilisateurs sont des analystes, des outils de reporting ou des tableaux de bord, en nombre généralement limité.
- **Une tolérance à la latence** : un temps de réponse de quelques secondes à quelques minutes est souvent acceptable.
- **Des données historiques et agrégées** : la base accumule l'historique pour permettre des comparaisons dans le temps.

Dans MariaDB, le moteur de référence pour l'OLAP est **ColumnStore**, un moteur de stockage **orienté colonnes** conçu pour l'analytique : il compresse fortement les données, n'en lit que les colonnes nécessaires et exécute les requêtes de manière parallèle.

```sql
-- OLAP : une agrégation sur un grand volume historique
SELECT
    YEAR(date_commande)        AS annee,
    region,
    SUM(montant)               AS chiffre_affaires,
    COUNT(*)                   AS nb_commandes
FROM commandes
GROUP BY annee, region
ORDER BY annee, chiffre_affaires DESC;
```

---

## Tableau comparatif

| Critère | OLTP | OLAP |
|---------|------|------|
| **Finalité** | Enregistrer les opérations courantes | Analyser de grands volumes de données |
| **Type de requêtes** | Courtes, ciblées | Longues, complexes (agrégations) |
| **Opérations dominantes** | Lectures **et** écritures | Lectures (chargements par lots) |
| **Lignes touchées par requête** | Peu (une ou quelques-unes) | Beaucoup (balayages massifs) |
| **Colonnes touchées par requête** | Souvent toute la ligne | Quelques colonnes seulement |
| **Concurrence** | Élevée (milliers d'utilisateurs) | Modérée (analystes, rapports) |
| **Latence attendue** | Milliseconde | Seconde à minute |
| **Schéma** | Normalisé (3NF) | Dénormalisé (étoile / flocon) |
| **Fraîcheur des données** | Temps réel, état courant | Historique, agrégé |
| **Moteur MariaDB typique** | InnoDB | ColumnStore |
| **Métrique clé** | Transactions par seconde (TPS) | Volume traité / temps |
| **Importance d'ACID** | Critique | Secondaire |

---

## La clé technique : stockage en lignes vs en colonnes

La différence la plus profonde entre ces deux mondes ne tient pas tant aux requêtes qu'à la **façon dont les données sont physiquement organisées sur le disque**. C'est ce qui explique pourquoi un seul moteur peut difficilement exceller dans les deux registres.

Un moteur **orienté lignes** comme InnoDB stocke les valeurs d'une même ligne de façon contiguë :

```
Ligne 1 → [ id=1 | nom=Alice | region=Nord | montant=100 ]
Ligne 2 → [ id=2 | nom=Bob   | region=Sud  | montant=200 ]
Ligne 3 → [ id=3 | nom=Chloé | region=Nord | montant=150 ]
```

Cette disposition est idéale pour l'OLTP : récupérer ou modifier une ligne entière (`SELECT *`, `INSERT`, `UPDATE`) ne demande qu'un accès localisé. En revanche, calculer `SUM(montant)` sur des millions de lignes oblige à parcourir l'intégralité des lignes — donc à lire aussi les colonnes `nom` et `region`, pourtant inutiles à ce calcul.

Un moteur **orienté colonnes** comme ColumnStore stocke au contraire les valeurs d'une même colonne ensemble :

```
Colonne id      → [ 1, 2, 3, ... ]
Colonne nom     → [ Alice, Bob, Chloé, ... ]
Colonne region  → [ Nord, Sud, Nord, ... ]
Colonne montant → [ 100, 200, 150, ... ]
```

Cette disposition apporte deux avantages décisifs pour l'OLAP :

1. **Moins d'entrées/sorties** : une agrégation sur `montant` ne lit que la colonne `montant`, en ignorant complètement les autres. Sur une table large, l'économie d'I/O est considérable.
2. **Une compression bien supérieure** : les valeurs d'une même colonne sont de même type et souvent répétitives (la colonne `region` ne contient que quelques valeurs distinctes), ce qui se compresse beaucoup mieux qu'un mélange de types au sein d'une ligne.

À l'inverse, ce modèle est mal adapté à l'écriture transactionnelle : insérer ou mettre à jour une seule ligne suppose de toucher tous les emplacements de colonnes correspondants, ce qui est coûteux. **Le choix de l'orientation est donc un compromis fondamental**, et c'est lui qui justifie l'existence de moteurs distincts pour l'OLTP et l'OLAP.

---

## Modélisation : schéma normalisé vs schéma en étoile

La nature de la charge influence aussi la **conception du schéma**.

En OLTP, on privilégie un **schéma normalisé** (typiquement en troisième forme normale). La normalisation élimine la redondance et garantit l'intégrité : une information n'est stockée qu'à un seul endroit, ce qui simplifie et sécurise les écritures. Le prix à payer est une multiplication des jointures à la lecture, acceptable puisque les requêtes OLTP restent ciblées.

En OLAP, on privilégie au contraire un **schéma dénormalisé**, le plus souvent un **schéma en étoile** : une table de **faits** centrale (les ventes, les événements mesurés), entourée de tables de **dimensions** (le temps, les produits, les clients, les régions). Cette dénormalisation réduit le nombre de jointures et accélère les requêtes analytiques, au prix d'une certaine redondance — un compromis cohérent avec un usage majoritairement en lecture.

```sql
-- OLAP : une table de faits, sur le moteur ColumnStore
CREATE TABLE faits_ventes (
    date_id      INT,
    produit_id   INT,
    client_id    INT,
    region_id    INT,
    quantite     INT,
    montant      DECIMAL(12,2)
) ENGINE=ColumnStore;
```

---

## Le choix du moteur dans MariaDB

L'un des atouts de MariaDB est son **architecture à moteurs de stockage interchangeables** (voir le chapitre 7) : on choisit le moteur table par table, en fonction de l'usage. Pour l'arbitrage OLTP/OLAP, deux moteurs se distinguent :

- **InnoDB** pour l'OLTP : transactionnel, ACID, verrouillage fin au niveau ligne, MVCC. C'est le moteur par défaut et le choix naturel pour les données opérationnelles.
- **ColumnStore** pour l'OLAP : stockage colonnaire, forte compression, exécution parallèle et capacité à traiter de très grands volumes analytiques.

Le **cost-based optimizer** de MariaDB 12.3, conscient des caractéristiques du stockage (y compris des disques SSD), adapte ses plans d'exécution à ces différents contextes.

---

## HTAP : quand les frontières s'estompent

Dans la pratique, la séparation entre OLTP et OLAP n'est pas toujours nette. De nombreuses organisations souhaitent analyser leurs données **au plus près du temps réel**, sans attendre un chargement nocturne. C'est l'ambition des architectures **HTAP** (*Hybrid Transactional/Analytical Processing*), qui cherchent à concilier les deux mondes.

L'écueil à éviter est de faire porter des requêtes analytiques lourdes directement sur la base transactionnelle de production : un balayage de plusieurs minutes peut saturer le *buffer pool*, allonger la durée des transactions et dégrader la latence perçue par les utilisateurs. La bonne pratique consiste plutôt à **séparer les charges** tout en synchronisant les données. Avec MariaDB, plusieurs approches sont possibles :

- **Réplication entre moteurs hétérogènes** : MariaDB autorise une réplication d'un primaire InnoDB vers un réplica ColumnStore, alimentant ainsi un entrepôt analytique de façon quasi continue à partir des données transactionnelles.
- **Pipelines de capture de changements (CDC)** : le journal binaire peut alimenter un système analytique via des outils comme Debezium et Kafka (voir §20.8).
- **Processus ETL classiques** : des chargements périodiques transforment et déversent les données opérationnelles vers l'entrepôt.

Ces mécanismes font le lien direct avec la suite du chapitre, notamment le **data warehousing avec ColumnStore** (§20.3) et les **architectures orientées événements** (§20.8).

---

## Implications architecturales

Identifier la nature d'une charge n'est pas un exercice académique : c'est la décision qui oriente toutes les suivantes.

- Le **moteur de stockage** (InnoDB pour l'OLTP, ColumnStore pour l'OLAP) découle directement de cette analyse.
- La **modélisation du schéma** (normalisé contre étoile) en dépend.
- La **stratégie d'indexation** diffère : index B-Tree sélectifs pour les accès OLTP, là où le stockage colonnaire de l'OLAP rend les index traditionnels largement superflus.
- Le **dimensionnement matériel** n'est pas le même : l'OLTP est sensible à la latence des écritures et à la mémoire dédiée au *buffer pool*, tandis que l'OLAP est avant tout gourmand en débit d'I/O et en puissance de calcul parallèle.
- Les **besoins en isolation transactionnelle** sont critiques en OLTP, beaucoup moins en OLAP.

En gardant cette dichotomie à l'esprit, on aborde les sections suivantes avec une grille de lecture claire : chaque cas d'usage présenté dans ce chapitre peut être situé sur l'axe transactionnel ↔ analytique, ce qui guide naturellement le choix des fonctionnalités MariaDB à mobiliser.

---

## Navigation

⬅️ Section précédente : [20. Cas d'Usage et Architectures](README.md)  
➡️ Section suivante : [20.2 Architecture microservices](02-architecture-microservices.md)  
⬆️ Retour au [sommaire](../SOMMAIRE.md)

⏭️ [Architecture microservices](/20-cas-usage-architectures/02-architecture-microservices.md)
