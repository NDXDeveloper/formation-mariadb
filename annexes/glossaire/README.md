🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe A - Glossaire des Termes Techniques

> **Niveau** : Tous niveaux (Référence)  
> **Durée estimée** : Consultation à la demande  
> **Type** : Documentation de référence

---

## 📖 Introduction

Ce glossaire constitue une **référence complète** des termes techniques, concepts et acronymes utilisés tout au long de cette formation MariaDB 11.8 LTS. Il est conçu pour être accessible à tous les niveaux, du débutant à l'expert, et sert de point de référence rapide lors de votre apprentissage.

### 🎯 Objectifs du glossaire

- **Clarifier** les termes techniques spécifiques à MariaDB
- **Standardiser** la compréhension des concepts clés
- **Faciliter** la navigation dans la formation
- **Accélérer** la résolution de doutes terminologiques
- **Enrichir** votre vocabulaire technique MariaDB

---

## 📋 Organisation du Glossaire

Le glossaire est organisé en deux sections principales :

### A.1 - Termes MariaDB Essentiels
Définitions détaillées des **concepts fondamentaux** de MariaDB :
- Propriétés ACID et transactions
- Mécanismes de concurrence (MVCC, verrous)
- Moteurs de stockage (InnoDB, Aria, Galera)
- Technologies de haute disponibilité
- Concepts d'indexation et performance
- Nouvelles fonctionnalités MariaDB 11.8

### A.2 - Acronymes Courants
Signification et contexte d'utilisation des **acronymes techniques** :
- Abréviations liées aux bases de données (FK, PK, DDL, DML)
- Termes de réplication (GTID, SST, IST)
- Concepts SQL avancés (CTE, Window Functions)
- Technologies DevOps et Cloud
- Standards et protocoles

---

## 🔍 Comment Utiliser ce Glossaire

### Recherche par Terme
1. **Identifiez le type** : terme technique ou acronyme
2. **Consultez la section** appropriée (A.1 ou A.2)
3. **Lisez la définition** et les exemples d'usage
4. **Explorez les références** pour approfondir

### Format des Définitions

Chaque entrée du glossaire suit ce format structuré :

```
**[TERME]**
Définition concise et précise du concept.

📌 **Contexte d'utilisation** : Où et quand ce terme est utilisé
💡 **Exemple** : Illustration pratique (si pertinent)
🔗 **Voir aussi** : Termes connexes
```

### Navigation Rapide

- **Termes en gras** : Définition principale
- 📌 **Contexte** : Information sur l'usage pratique
- 💡 **Exemple** : Cas concret d'utilisation
- 🆕 **Badge** : Nouveauté MariaDB 11.8
- 🔄 **Badge** : Terme modifié ou déprécié
- 🔗 **Liens** : Références croisées

---

## 🎓 Conventions et Notations

### Niveaux de Complexité

Certains termes sont marqués selon leur niveau de complexité :

- 🟢 **Débutant** : Concept fondamental, accessible
- 🟡 **Intermédiaire** : Nécessite des bases MariaDB
- 🔴 **Avancé** : Concept technique spécialisé
- 🟣 **Expert** : Utilisation en production avancée

### Types de Termes

Les termes sont catégorisés pour faciliter la compréhension :

| Catégorie | Description | Exemples |
|-----------|-------------|----------|
| **Architecture** | Composants structurels | InnoDB, Buffer Pool, Binlog |
| **Opérations** | Actions et commandes | GRANT, COMMIT, ANALYZE |
| **Concepts** | Principes théoriques | ACID, Isolation, Normalisation |
| **Technologies** | Outils et frameworks | Galera, MaxScale, Kubernetes |
| **Performance** | Optimisation et tuning | Index, Partitioning, Caching |
| **Sécurité** | Protection et contrôle | SSL/TLS, Privileges, Encryption |

---

## 📊 Structure des Sections

### Section A.1 : Termes MariaDB Essentiels

Cette section couvre **environ 60-80 termes** organisés alphabétiquement :

**Catégories principales :**
- **Transactions et Cohérence** : ACID, Isolation, MVCC, Locking
- **Moteurs de Stockage** : InnoDB, Aria, ColumnStore, Memory
- **Réplication et HA** : Galera, GTID, Binlog, MaxScale
- **Performance** : Index, Buffer Pool, Query Cache, Partitioning
- **Sécurité** : SSL/TLS, Privileges, Encryption, Authentication
- **Nouveautés 11.8** : Vector, HNSW, PARSEC, Application Time Period

### Section A.2 : Acronymes Courants

Cette section définit **environ 50-70 acronymes** classés alphabétiquement :

**Domaines couverts :**
- **SQL** : DDL, DML, DCL, TCL, CTE
- **Clés et Contraintes** : PK, FK, UK, NN
- **Réplication** : GTID, SST, IST, PITR
- **Performance** : OLTP, OLAP, IOPS, QPS
- **Haute Disponibilité** : HA, DR, RPO, RTO
- **DevOps** : CI/CD, IaC, K8s, VPC

---

## 💡 Conseils d'Utilisation

### Pour les Débutants
- Commencez par les termes marqués 🟢
- Lisez les définitions dans l'ordre alphabétique
- Prenez le temps de comprendre les exemples
- N'hésitez pas à revenir régulièrement

### Pour les Intermédiaires
- Utilisez le glossaire comme référence lors de la lecture
- Approfondissez les termes marqués 🟡 et 🔴
- Explorez les liens "Voir aussi" pour élargir vos connaissances
- Comparez avec votre expérience pratique

### Pour les Experts
- Consultez prioritairement les nouveautés 🆕
- Utilisez-le pour standardiser la terminologie dans votre équipe
- Vérifiez les changements marqués 🔄
- Contribuez à l'amélioration du glossaire

---

## 🔄 Évolution et Mises à Jour

Ce glossaire est **vivant** et évolue avec MariaDB :

### Version Actuelle : 1.0 (Décembre 2025)
- ✅ Aligné sur MariaDB 11.8 LTS
- ✅ Intégration des nouveautés Vector, PARSEC, MaxScale 25.01
- ✅ Mise à jour des termes dépréciés
- ✅ Ajout des concepts Cloud et IA/ML

### Historique des Changements Majeurs
- **11.8 LTS** : Ajout Vector, HNSW, Application Time Period, PARSEC
- **11.4 LTS** : Politique de support 3 ans, nouveaux privilèges
- **10.11 LTS** : utf8mb4 par défaut, amélioration GTID

---

## 📚 Complémentarité avec la Formation

Le glossaire est **complémentaire** aux chapitres de la formation :

| Section Formation | Termes Clés du Glossaire |
|-------------------|--------------------------|
| **Partie 1-2** : Fondamentaux | SQL, DDL, DML, Constraints, ACID |
| **Partie 3** : Performance | Index, B-Tree, EXPLAIN, Buffer Pool |
| **Partie 4** : Moteurs | InnoDB, Aria, MVCC, Redo/Undo Log |
| **Partie 5** : Sécurité | SSL/TLS, GRANT, Roles, Authentication |
| **Partie 6** : Réplication | GTID, Binlog, SST, IST, Galera |
| **Partie 7** : Tuning | OLTP, OLAP, Partitioning, Sharding |
| **Partie 8** : DevOps | K8s, Docker, CI/CD, IaC |
| **Partie 9** : Avancé | Vector, HNSW, RAG, System-Versioned |
| **Partie 10** : Architecture | HA, DR, CDC, Event-Driven |

---

## 🎯 Points Clés à Retenir

### Utilisation Optimale
- ✅ **Référence rapide** lors de l'apprentissage
- ✅ **Vérification** de la compréhension des termes
- ✅ **Standardisation** du vocabulaire technique
- ✅ **Support** pour la communication en équipe

### Navigation Efficace
- 📖 Utilisez la recherche de votre éditeur (Ctrl+F / Cmd+F)
- 🔗 Suivez les liens "Voir aussi" pour les termes connexes
- 📌 Marquez vos termes favoris pour référence future
- 💡 Explorez les exemples pour une compréhension pratique

### Mises à Jour
- 🔄 Vérifiez les badges pour les changements récents
- 🆕 Explorez les nouveautés 11.8 en priorité
- 📅 Consultez régulièrement pour rester à jour
- 💬 Partagez vos retours pour améliorer le glossaire

---

## ✅ Checklist d'Utilisation

Avant de commencer la formation :
- [ ] Parcourir la structure du glossaire
- [ ] Identifier les termes inconnus
- [ ] Lire les définitions des concepts fondamentaux (ACID, SQL, Index)

Pendant la formation :
- [ ] Consulter le glossaire dès qu'un terme est flou
- [ ] Prendre des notes personnelles sur les définitions
- [ ] Explorer les termes connexes pour approfondir

Après la formation :
- [ ] Réviser les termes marqués 🔴 et 🟣
- [ ] Utiliser le glossaire comme référence au travail
- [ ] Partager avec vos collègues pour standardiser le vocabulaire

---

## 🔗 Sections du Glossaire

### → [A.1 Termes MariaDB Essentiels](./01-termes-mariadb-essentiels.md)
Définitions détaillées des concepts fondamentaux : ACID, MVCC, InnoDB, Galera, Buffer Pool, Binlog, GTID, Index, et plus encore.

### → [A.2 Acronymes Courants](./02-acronymes-courants.md)
Signification et contexte des abréviations techniques : FK, PK, CTE, DDL, DML, SST, IST, OLTP, OLAP, et plus encore.

---

## 📖 Ressources Complémentaires

Pour approfondir votre compréhension :

### Documentation Officielle
- [MariaDB Knowledge Base](https://mariadb.com/kb/en/) - Référence complète
- [MariaDB Server Documentation](https://mariadb.com/kb/en/documentation/) - Documentation serveur
- [MariaDB Glossary](https://mariadb.com/kb/en/mariadb-glossary/) - Glossaire officiel

### Standards et Spécifications
- [SQL:2023 Standard](https://www.iso.org/standard/76583.html) - Standard SQL ISO
- [ACID Properties](https://en.wikipedia.org/wiki/ACID) - Propriétés transactionnelles
- [CAP Theorem](https://en.wikipedia.org/wiki/CAP_theorem) - Théorème pour systèmes distribués

### Communauté
- [MariaDB Community Slack](https://mariadb.org/get-involved/community/) - Discussions techniques
- [Stack Overflow - MariaDB](https://stackoverflow.com/questions/tagged/mariadb) - Q&A technique

---

## 💬 Contribuer au Glossaire

Ce glossaire peut être enrichi avec votre expérience :

### Suggestions Bienvenues
- 📝 Nouvelles définitions à ajouter
- 🔧 Amélioration des définitions existantes
- 💡 Exemples pratiques supplémentaires
- 🔗 Liens vers des ressources pertinentes

### Comment Contribuer
1. Identifiez un terme manquant ou une amélioration
2. Proposez une définition concise et précise
3. Ajoutez un exemple d'usage si pertinent
4. Vérifiez les références officielles

---

**MariaDB Version** : 11.8 LTS  

---

## ➡️ Sections Suivantes

- **[A.1 Termes MariaDB Essentiels →](./01-termes-mariadb-essentiels.md)**
  Découvrez les définitions détaillées des concepts fondamentaux de MariaDB

- **[A.2 Acronymes Courants →](./02-acronymes-courants.md)**
  Comprenez la signification de tous les acronymes techniques utilisés dans la formation

---

*💡 Astuce : Gardez ce glossaire ouvert dans un onglet séparé pendant votre apprentissage pour une consultation rapide !*

⏭️ [Termes MariaDB essentiels (ACID, MVCC, InnoDB, Galera, etc.)](/annexes/glossaire/01-termes-mariadb-essentiels.md)
