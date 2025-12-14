🔝 Retour au [Sommaire](/SOMMAIRE.md)

# H.3 Blogs Techniques Recommandés 📝

> **Niveau** : Tous niveaux (Référence)  
> **Durée estimée** : 15-20 minutes  
> **Prérequis** : Aucun

## 🎯 Objectif de cette section

Présenter une **sélection curatée** des meilleurs blogs techniques MariaDB pour :
- Rester à jour sur les évolutions
- Apprendre des best practices
- Découvrir des cas d'usage réels
- Approfondir des sujets techniques
- S'inspirer de la communauté

---

## 📚 Taxonomie des Blogs

### Catégorisation par type

```
┌─────────────────────────────────────────────────────────────┐
│              Écosystème des Blogs MariaDB                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  🔵 Officiels                                               │
│     ├─ MariaDB Blog (officiel)                              │
│     ├─ MariaDB Foundation Blog                              │
│     └─ MaxScale Blog                                        │
│                                                             │
│  🟢 Agrégateurs Communautaires                              │
│     ├─ Planet MariaDB                                       │
│     └─ Planet MySQL (partiellement)                         │
│                                                             │
│  🟣 Experts & Consultants                                   │
│     ├─ Federico Razzoli (MariaDB expert)                    │
│     ├─ Monty Widenius (créateur)                            │
│     ├─ Daniel Bartholomew (Foundation)                      │
│     └─ Consultants indépendants                             │
│                                                             │
│  🟡 Sociétés Spécialisées                                   │
│     ├─ Percona Database Performance Blog                    │
│     ├─ Vettabase Blog                                       │
│     ├─ ScaleGrid Blog                                       │
│     └─ PlanetScale Blog                                     │
│                                                             │
│  🟠 Tech Généraliste                                        │
│     ├─ DZone Database Zone                                  │
│     ├─ InfoQ Databases                                      │
│     ├─ Hacker News                                          │
│     └─ Dev.to (tag #database)                               │
│                                                             │
│  🔴 Académique & Recherche                                  │
│     ├─ ACM Queue                                            │
│     ├─ Google Research Blog                                 │
│     └─ Microsoft Research                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 1️⃣ Blogs Officiels MariaDB

### MariaDB Blog (Officiel)

**URL** : [https://mariadb.org/blog/](https://mariadb.org/blog/)

**Fréquence** : 2-3 articles/semaine

**Qualité** : ⭐⭐⭐⭐⭐

**Langues** : Principalement anglais

#### Type de Contenu

| Catégorie | % | Niveau |
|-----------|---|--------|
| **Annonces releases** | 30% | Tous |
| **Tutoriels techniques** | 25% | Intermédiaire |
| **Best practices** | 20% | Intermédiaire+ |
| **Case studies** | 15% | Tous |
| **Événements** | 10% | Tous |

#### Articles Phares 2024-2025

**MariaDB Vector (Novembre 2025)** 🆕
```
Titre: "Introducing MariaDB Vector: Native Vector Search for AI Applications"
Lien: https://mariadb.org/blog/introducing-mariadb-vector/

Résumé:
- Présentation du type VECTOR et index HNSW
- Architecture pour applications RAG
- Benchmarks vs solutions concurrentes
- Exemples d'intégration avec OpenAI, Claude

Niveau: Intermédiaire
Durée lecture: 15 min
```

**Performance Tuning MariaDB 11.8 (Octobre 2025)** ⚡
```
Titre: "Performance Improvements in MariaDB 11.8 LTS"
Lien: https://mariadb.org/blog/performance-improvements-11-8/

Points clés:
- innodb_alter_copy_bulk: +60% vitesse ALTER TABLE
- Cost optimizer SSD-aware
- Optimisations SIMD pour Vector
- Benchmarks OLTP détaillés

Niveau: Avancé
Durée lecture: 20 min
```

**Migration MySQL to MariaDB (Septembre 2025)**
```
Titre: "Complete Guide: Migrating from MySQL 8.0 to MariaDB 11.8"
Lien: https://mariadb.org/blog/mysql-to-mariadb-migration-guide/

Couverture:
- Compatibilité et divergences
- Stratégies de migration (dump, replication)
- Checklist complète
- Troubleshooting courant

Niveau: Intermédiaire
Durée lecture: 25 min
```

#### Comment Suivre

```bash
# RSS Feed
https://mariadb.org/feed/

# Newsletter mensuelle
https://mariadb.org/newsletter/

# Twitter
@mariadb
```

---

### MariaDB Foundation Blog

**URL** : [https://mariadb.org/category/foundation/](https://mariadb.org/category/foundation/)

**Fréquence** : 1-2 articles/mois

**Focus** :
- Gouvernance et organisation
- Événements communautaires
- Annonces importantes
- Collaboration et sponsoring

**Intérêt** : 🟡 Moyen (sauf si intéressé par gouvernance)

---

## 2️⃣ Planet MariaDB - Agrégateur Communautaire

### Présentation

**URL** : [https://planet.mariadb.org/](https://planet.mariadb.org/)

**Concept** : Agrégateur RSS de **blogs de la communauté** MariaDB

**Fréquence** : 5-10 articles/semaine

**Qualité** : ⭐⭐⭐⭐ (variable selon auteurs)

### Blogueurs Vedettes

#### Monty Widenius (Créateur de MariaDB)

**Blog** : [https://monty-says.blogspot.com/](https://monty-says.blogspot.com/)

**Fréquence** : Irrégulière (1-2/mois)

**Thèmes** :
- Vision long-terme MariaDB
- Décisions d'architecture
- Open-source et philosophie
- Annonces stratégiques

**Style** : Personnel, visionnaire, technique

**Articles recommandés** :
- "Why MariaDB Will Win" (2024)
- "The Future of Database Storage Engines" (2025)
- "MariaDB Vector: Making AI Databases Democratic" (2025) 🆕

---

#### Federico Razzoli (MariaDB Consultant)

**Blog** : [https://federico-razzoli.com/](https://federico-razzoli.com/)

**Fréquence** : 2-3 articles/mois

**Qualité** : ⭐⭐⭐⭐⭐

**Spécialités** :
- MariaDB avancé
- Haute disponibilité
- Performance tuning
- DevOps et automation

**Séries populaires** :
- "MariaDB Hidden Gems" (features méconnues)
- "Galera Cluster Deep Dive"
- "MariaDB for Developers"

**Niveau** : 🟠 Avancé

**Exemple d'article** :
```
Titre: "MariaDB 11.8: What You Need to Know About Vector Indexes"
Date: Novembre 2025

Contenu:
- Fonctionnement interne HNSW
- Tuning paramètres M et ef_construction
- Comparaison avec pgvector
- Cas d'usage production

Niveau: Avancé
Durée: 30 min
```

---

#### Daniel Bartholomew (MariaDB Foundation)

**Blog** : [https://dbart.info/](https://dbart.info/)

**Fréquence** : 1-2 articles/mois

**Thèmes** :
- Documentation MariaDB
- Releases et changelogs
- Communauté et événements
- Tips & tricks

**Style** : Accessible, pédagogique

**Niveau** : 🟢 Tous niveaux

---

#### Ian Gilfillan (Documentation Lead)

**Blog** : Contributions sur Planet MariaDB

**Focus** :
- Nouvelles fonctionnalités expliquées
- Guides migration
- FAQ et troubleshooting

**Niveau** : 🟡 Intermédiaire

---

### Comment Utiliser Planet MariaDB

**S'abonner RSS** :
```
https://planet.mariadb.org/atom.xml
```

**Lecteurs RSS recommandés** :
- **Feedly** (web, mobile)
- **Inoreader** (web, mobile)
- **NetNewsWire** (macOS, iOS)
- **Thunderbird** (desktop, intégré email)

**Filtrage** :
- ⭐ Suivre auteurs favoris uniquement
- 🏷️ Tags/catégories pour trier
- 📧 Digest hebdomadaire plutôt que temps réel

---

## 3️⃣ Percona Database Performance Blog

### Présentation

**URL** : [https://www.percona.com/blog/](https://www.percona.com/blog/)

**Fréquence** : 3-5 articles/semaine

**Qualité** : ⭐⭐⭐⭐⭐

**Focus** :
- Performance optimization
- Monitoring et observabilité
- Outils Percona Toolkit
- Comparaisons MySQL/MariaDB/PostgreSQL

### Points Forts

- ✅ **Benchmarks détaillés** avec méthodologie reproductible
- ✅ **Profondeur technique** exceptionnelle
- ✅ **Cas réels** de clients (anonymisés)
- ✅ **Outils open-source** développés et partagés

### Séries Recommandées

#### "MySQL Performance Blog"

Articles MariaDB fréquents, compatibilité forte.

**Thèmes** :
- InnoDB internals
- Query optimization
- Index strategies
- Replication performance

**Niveau** : 🟠 Avancé

#### "Percona Monitoring and Management (PMM)"

Monitoring MariaDB avec PMM.

**Articles types** :
- Setup PMM pour MariaDB
- Dashboard personnalisés
- Alerting avancé
- Query Analytics

**Niveau** : 🟡 Intermédiaire

#### Exemple d'Article Marquant

```
Titre: "InnoDB Buffer Pool Tuning: The Complete Guide"
Auteur: Percona Team
Date: Août 2025

Contenu:
- Mécanismes internes buffer pool
- Formules de dimensionnement
- Tuning pour différents workloads
- Monitoring avec PMM
- Pitfalls et erreurs courantes

Niveau: Avancé
Durée: 45 min
Pages: 15+
Graphiques: 20+
```

### S'abonner

```
RSS: https://www.percona.com/blog/feed/
Newsletter: https://www.percona.com/resources/newsletter
```

---

## 4️⃣ Vettabase Blog

### Présentation

**URL** : [https://vettabase.com/blog/](https://vettabase.com/blog/)

**Société** : Vettabase Ltd (UK) - Consulting MariaDB/PostgreSQL

**Fréquence** : 2-3 articles/mois

**Qualité** : ⭐⭐⭐⭐

**Spécialités** :
- MariaDB vs PostgreSQL comparisons
- DevOps pour bases de données
- Kubernetes et cloud-native
- Migration strategies

### Articles Notables

**"MariaDB or PostgreSQL? Decision Framework"** (2024)
- Critères de choix objectifs
- Use cases par SGBD
- Migration considerations
- TCO (Total Cost of Ownership)

**"Running MariaDB on Kubernetes: Production Patterns"** (2025)
- StatefulSets best practices
- Storage classes pour performance
- Backup strategies
- Monitoring et observabilité

**Niveau moyen** : 🟡 Intermédiaire

---

## 5️⃣ Autres Blogs Sociétés

### ScaleGrid Blog

**URL** : [https://scalegrid.io/blog/](https://scalegrid.io/blog/)

**Focus** : DBaaS (Database as a Service)

**Thèmes MariaDB** :
- Cloud deployment (AWS, Azure, GCP)
- Managed services
- Cost optimization
- Security best practices

**Fréquence** : 1-2 articles/mois (MariaDB)

**Niveau** : 🟢 Tous niveaux

---

### DigitalOcean Community Tutorials

**URL** : [https://www.digitalocean.com/community/tags/mariadb](https://www.digitalocean.com/community/tags/mariadb)

**Format** : Tutoriels step-by-step

**Thèmes** :
- Installation sur Ubuntu/Debian
- Configuration sécurité
- Backup/restore
- Réplication master-slave

**Qualité** : ⭐⭐⭐⭐

**Niveau** : 🟢 Débutant à Intermédiaire

**Avantages** :
- ✅ Gratuit et communautaire
- ✅ Testé et validé
- ✅ Copy-paste friendly
- ✅ Régulièrement mis à jour

---

### AWS Database Blog

**URL** : [https://aws.amazon.com/blogs/database/](https://aws.amazon.com/blogs/database/)

**Articles MariaDB** : Occasionnels (RDS MariaDB, Aurora MySQL compatible)

**Thèmes** :
- RDS MariaDB best practices
- Migration to RDS
- Performance tuning on AWS
- Cost optimization

**Niveau** : 🟡 Intermédiaire

---

## 6️⃣ Blogs Techniques Généralistes

### DZone - Database Zone

**URL** : [https://dzone.com/database-sql-nosql-tutorials-tools-news](https://dzone.com/database-sql-nosql-tutorials-tools-news)

**Tag** : `#mariadb`

**Fréquence** : 2-3 articles MariaDB/mois

**Type** :
- Tutorials
- Opinion pieces
- Comparisons
- News aggregation

**Qualité** : ⭐⭐⭐ (variable)

**Niveau** : 🟢 Tous niveaux

---

### InfoQ - Databases

**URL** : [https://www.infoq.com/databases/](https://www.infoq.com/databases/)

**Fréquence** : 1-2 articles MariaDB/mois

**Format** :
- Articles longs et approfondis
- Interviews d'experts
- Trend reports
- Presentations from conferences

**Qualité** : ⭐⭐⭐⭐

**Niveau** : 🟡 Intermédiaire+

---

### Dev.to

**URL** : [https://dev.to/t/mariadb](https://dev.to/t/mariadb)

**Tag** : `#mariadb`

**Concept** : Plateforme communautaire de blogging développeurs

**Fréquence** : 5-10 articles/mois

**Qualité** : ⭐⭐⭐ (très variable)

**Type de contenu** :
- Tutoriels débutants
- "How I solved X"
- Project showcases
- Tips & tricks

**Niveau** : 🟢 Débutant à Intermédiaire

**Avantages** :
- ✅ Contenu très pratique
- ✅ Expériences réelles
- ✅ Communauté active
- ✅ Commentaires et discussions

---

### Hacker News

**URL** : [https://news.ycombinator.com/](https://news.ycombinator.com/)

**Recherche** : "MariaDB"

**Utilité** :
- Discussions techniques de qualité
- Annonces importantes upvotées
- Débats architecture/design
- Feedback utilisateurs

**Comment utiliser** :
```
Recherche: https://hn.algolia.com/?query=MariaDB
Trier par: Date ou Popularité
Lire: Article + Commentaires (souvent plus intéressants que l'article)
```

**Niveau** : 🟠 Avancé (discussions pointues)

---

## 7️⃣ Blogs Académiques et Recherche

### ACM Queue

**URL** : [https://queue.acm.org/](https://queue.acm.org/)

**Thème** : Computer Science research-to-practice

**Articles bases de données** : Mensuels

**Qualité** : ⭐⭐⭐⭐⭐

**Type** :
- Papers recherche accessibles
- Database internals
- Distributed systems
- Novel algorithms (ex: HNSW)

**Niveau** : 🔴 Expert

**Exemple** :
```
Titre: "Efficient Similarity Search in High-Dimensional Spaces"
Auteur: Research team
Pertinence: Explique algorithmes comme HNSW (MariaDB Vector)

Niveau: Expert
Durée: 60-90 min
```

---

### Google Research Blog

**URL** : [https://research.google/blog/](https://research.google/blog/)

**Tag** : "databases", "data management"

**Fréquence** : Occasionnelle

**Thèmes pertinents** :
- Vector search algorithms
- Database optimization techniques
- Distributed databases
- ML for databases

**Niveau** : 🔴 Expert

---

### The Morning Paper

**URL** : [https://blog.acolyer.org/](https://blog.acolyer.org/)

**Concept** : Résumés quotidiens de papers Computer Science

**Bases de données** : 1-2/semaine

**Qualité** : ⭐⭐⭐⭐⭐

**Utilité** :
- Découvrir papers importants
- Résumés accessibles
- Contexte et implications

**Niveau** : 🟠 Avancé à Expert

---

## 8️⃣ Veille et Agrégateurs

### Database Weekly

**URL** : [https://dbweekly.com/](https://dbweekly.com/)

**Format** : Newsletter hebdomadaire

**Contenu** :
- 10-15 liens/semaine
- Toutes bases de données
- News, articles, outils, jobs

**MariaDB** : 1-2 liens/semaine en moyenne

**Avantages** :
- ✅ Curation de qualité
- ✅ Veille automatisée
- ✅ Gratuit
- ✅ Archive consultable

---

### Lobsters

**URL** : [https://lobste.rs/](https://lobste.rs/)

**Tag** : `databases`

**Concept** : Alternative Hacker News (plus petit, plus focalisé)

**Qualité discussions** : ⭐⭐⭐⭐⭐

**Modération** : Stricte, invite-only posting

**Articles MariaDB** : 2-3/mois

---

### Reddit /r/databases

**URL** : [https://www.reddit.com/r/databases/](https://www.reddit.com/r/databases/)

**Abonnés** : ~100,000

**Contenu** :
- Discussions comparatives
- Annonces importantes
- Questions architecture
- Partage d'articles

**Filtrer MariaDB** : Recherche "MariaDB" ou flair

---

## 9️⃣ Blogs Francophones

### Journal du hacker

**URL** : [https://www.journalduhacker.net/](https://www.journalduhacker.net/)

**Tag** : "base de données"

**Fréquence MariaDB** : Occasionnelle

**Langue** : Français

**Qualité** : ⭐⭐⭐

---

### LinuxFr.org

**URL** : [https://linuxfr.org/](https://linuxfr.org/)

**Section** : Dépêches > Base de données

**Fréquence MariaDB** : 1-2/mois

**Type** :
- Annonces releases
- Tutoriels communautaires
- Discussions techniques

**Niveau** : 🟢 Tous niveaux

---

### Developpez.com

**URL** : [https://sgbd.developpez.com/](https://sgbd.developpez.com/)

**Section** : SGBD et SQL

**Articles MariaDB** : Occasionnels

**Type** :
- Tutoriels débutants
- Actualités traduites
- Forums actifs

**Niveau** : 🟢 Débutant à Intermédiaire

---

## 🔟 Comment Suivre Efficacement

### Stratégie de Veille

#### Niveau Débutant (30 min/semaine)

```
Sources (3) :
1. MariaDB Blog officiel (RSS)
2. Planet MariaDB (digest hebdo)
3. Database Weekly (newsletter)

Routine :
- Lundi matin : Newsletter Database Weekly
- Mercredi : MariaDB Blog (nouveaux articles)
- Vendredi : Planet MariaDB (sélection)
```

#### Niveau Intermédiaire (1-2h/semaine)

```
Sources (5) :
1. MariaDB Blog officiel
2. Planet MariaDB
3. Percona Blog (tag MariaDB)
4. Federico Razzoli blog
5. StackOverflow [mariadb] tag

Routine :
- Quotidien (15 min) : Twitter @mariadb, HackerNews
- Hebdomadaire (1h) : Lecture articles sélectionnés
- Mensuel (2h) : Deep dive sur un sujet
```

#### Niveau Avancé (3-5h/semaine)

```
Sources (10+) :
- Tous ci-dessus
+ ACM Queue
+ Mailing lists maria-discuss
+ GitHub MariaDB/server (watch releases)
+ Research papers (arXiv.org)
+ Conférences (vidéos enregistrées)

Routine :
- Quotidien (30 min) : Veille multi-sources
- Hebdomadaire (2h) : Lecture approfondie
- Mensuel (4h) : Expérimentation/tests
- Trimestriel : Contribution (blog post, talk)
```

### Outils de Veille

#### Lecteurs RSS

| Outil | Type | Avantages |
|-------|------|-----------|
| **Feedly** | Web/Mobile | ⭐⭐⭐⭐⭐ Interface, intégrations |
| **Inoreader** | Web/Mobile | ⭐⭐⭐⭐ Filtres avancés |
| **NetNewsWire** | macOS/iOS | ⭐⭐⭐⭐ Gratuit, open-source |
| **The Old Reader** | Web | ⭐⭐⭐ Simple, Google Reader-like |

#### Agrégateurs Automatiques

**IFTTT / Zapier** :
```
Recette :
SI nouveau article Planet MariaDB
AVEC "Vector" OU "Performance" dans titre
ALORS envoyer à Slack/Email

SI nouveau commit GitHub MariaDB/server
AVEC tag "release"
ALORS notification Telegram
```

#### Bookmarking Social

| Service | Usage |
|---------|-------|
| **Pocket** | Read-it-later |
| **Instapaper** | Read-it-later, highlights |
| **Raindrop.io** | Bookmarks organisés |
| **Pinboard** | Bookmarks archivés |

---

## 📊 Matrice de Recommandations

### Par Niveau

| Niveau | Blogs Essentiels | Temps/Semaine |
|--------|------------------|---------------|
| **Débutant** | MariaDB Blog, Database Weekly, Dev.to | 30 min |
| **Intermédiaire** | + Planet MariaDB, Percona, Federico Razzoli | 1-2h |
| **Avancé** | + Vettabase, ACM Queue, Research papers | 3-5h |
| **Expert** | + Mailing lists dev, Source code, Conférences | 5-10h |

### Par Objectif

| Objectif | Blogs Recommandés |
|----------|-------------------|
| **Rester à jour** | MariaDB Blog, Planet MariaDB, Database Weekly |
| **Apprendre** | DigitalOcean Tutorials, Dev.to, Percona Blog |
| **Optimiser performance** | Percona Blog, Federico Razzoli, Vettabase |
| **Préparer certification** | MariaDB Blog, Documentation KB |
| **Contribuer** | Planet MariaDB, maria-developers list |
| **Recherche** | ACM Queue, Google Research, ArXiv |

### Par Thématique

| Thématique | Blogs Spécialisés |
|------------|-------------------|
| **IA/Vector** | MariaDB Blog 🆕, Research blogs |
| **HA/Galera** | Federico Razzoli, Percona Blog |
| **Performance** | Percona Blog, Vettabase |
| **Cloud/K8s** | Vettabase, AWS Blog, ScaleGrid |
| **DevOps** | DigitalOcean, Vettabase, Percona |
| **Migration** | MariaDB Blog, Percona Blog |

---

## ✅ Points Clés à Retenir

- **MariaDB Blog officiel** est la source #1 pour actualités et annonces
- **Planet MariaDB** agrège les meilleurs blogs communautaires
- **Percona Blog** excellente ressource pour performance et tuning
- **Federico Razzoli** expert reconnu MariaDB, contenu avancé de qualité
- **Database Weekly** newsletter pour veille globale bases de données
- **Dev.to** bon pour tutoriels pratiques et retours d'expérience
- **Lecteur RSS** (Feedly, Inoreader) essentiel pour suivre efficacement
- **30 min/semaine** suffisant pour débutants (3 sources)
- **Combiner** blogs officiels + communautaires + généralistes
- **Participer** : Commenter, partager, écrire vos propres articles

---

## 🔗 Flux RSS Essentiels

### Top 10 Flux à Ajouter

```xml
1. MariaDB Blog
   https://mariadb.org/feed/

2. Planet MariaDB
   https://planet.mariadb.org/atom.xml

3. Percona Blog (tag MariaDB)
   https://www.percona.com/blog/tag/mariadb/feed/

4. Federico Razzoli
   https://federico-razzoli.com/feed/

5. Vettabase Blog
   https://vettabase.com/blog/feed/

6. Dev.to (tag mariadb)
   https://dev.to/feed/tag/mariadb

7. DigitalOcean (tag mariadb)
   https://www.digitalocean.com/community/tags/mariadb/feed

8. DZone Database Zone
   https://dzone.com/database-sql-nosql-tutorials-tools-news/rss

9. InfoQ Databases
   https://www.infoq.com/databases/feed

10. Database Weekly (email newsletter)
    https://dbweekly.com/
```

---

## 📚 Lectures Recommandées par Mois

### Janvier 2026 - Focus Vector/IA

- [ ] MariaDB Blog: "Vector Performance Benchmarks"
- [ ] Research: "HNSW Algorithm Explained"
- [ ] Tutorial: "Building RAG with MariaDB Vector"

### Février 2026 - Focus Performance

- [ ] Percona: "InnoDB Tuning Guide 2026"
- [ ] Federico Razzoli: "Query Optimization Patterns"
- [ ] Vettabase: "SSD Optimization for MariaDB"

### Mars 2026 - Focus HA

- [ ] MariaDB Blog: "Galera Cluster Best Practices"
- [ ] Percona: "Replication Monitoring"
- [ ] Federico Razzoli: "MaxScale 25.01 Deep Dive"

💡 Adapter selon vos besoins et niveau !

---

## ➡️ Section Suivante

- **H.4** [Conférences et événements](./04-conferences-evenements.md)


---

💡 **Conseil final** : Commencez avec **3 sources maximum** (MariaDB Blog + Planet MariaDB + Database Weekly) et ajoutez progressivement selon vos intérêts. La régularité (15-30 min/semaine) vaut mieux que des sessions marathon irrégulières ! 📚

⏭️ [Conférences et événements](/annexes/ressources-documentation/04-conferences-evenements.md)
