🔝 Retour au [Sommaire](/SOMMAIRE.md)

# H. Ressources et Documentation 📚

> **Niveau** : Tous niveaux (Référence)  
> **Durée estimée** : 10-15 minutes  
> **Prérequis** : Aucun

## 🎯 Objectif de cette annexe

Fournir une **liste complète et organisée** des ressources essentielles pour :
- Approfondir vos connaissances MariaDB
- Résoudre des problèmes techniques
- Rester à jour sur les évolutions
- Échanger avec la communauté
- Participer à des événements

---

## 📖 Vue d'Ensemble des Ressources

### Cartographie des ressources

```
┌─────────────────────────────────────────────────────────────┐
│            Écosystème de Ressources MariaDB                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1️⃣ Documentation Officielle                                │
│     ├─ Knowledge Base (KB)                                  │
│     ├─ API Reference                                        │
│     ├─ Release Notes                                        │
│     └─ Training Materials                                   │
│                                                             │
│  2️⃣ Communautés et Forums                                   │
│     ├─ Zulip Chat                                           │
│     ├─ Mailing Lists                                        │
│     ├─ Reddit & StackOverflow                               │
│     └─ Social Media (Twitter, LinkedIn)                     │
│                                                             │
│  3️⃣ Blogs et Publications                                   │
│     ├─ MariaDB Blog officiel                                │
│     ├─ Planet MariaDB                                       │
│     ├─ Blogs techniques indépendants                        │
│     └─ Publications académiques                             │
│                                                             │
│  4️⃣ Conférences et Événements                               │
│     ├─ MariaDB Server Fest (annuel)                         │
│     ├─ Percona Live                                         │
│     ├─ FOSDEM                                               │
│     └─ Meetups locaux                                       │
│                                                             │
│  5️⃣ Formation et Certification                              │
│     ├─ Cours officiels MariaDB                              │
│     ├─ Certifications                                       │
│     ├─ Webinars                                             │
│     └─ Tutoriels vidéo                                      │
│                                                             │
│  6️⃣ Code et Contribution                                    │
│     ├─ GitHub MariaDB Server                                │
│     ├─ Jira (bug tracking)                                  │
│     ├─ Code de contribution                                 │
│     └─ Projets satellites                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 📊 Matrice de Ressources par Niveau

| Niveau | Ressources Recommandées | Priorité |
|--------|------------------------|----------|
| **Débutant** | KB Getting Started, Forums, Tutoriels vidéo | 🔥 |
| **Intermédiaire** | KB Advanced Topics, Blogs techniques, Webinars | ⚡ |
| **Avancé** | Source code, Jira, Mailing lists développeurs | 📊 |
| **Expert** | Contribution code, Conférences (speaker), Research papers | 🎯 |

---

## 1️⃣ Documentation Officielle

### MariaDB Knowledge Base (KB)

**URL** : [https://mariadb.com/kb/en/](https://mariadb.com/kb/en/)

La ressource **#1** pour toute information technique sur MariaDB.

#### Sections clés

| Section | Description | Public | Lien |
|---------|-------------|--------|------|
| **Getting Started** | Installation, premiers pas | 🟢 Débutant | [KB Getting Started](https://mariadb.com/kb/en/getting-started/) |
| **SQL Statements** | Référence SQL complète | 🟡 Tous niveaux | [KB SQL](https://mariadb.com/kb/en/sql-statements-structure/) |
| **Built-in Functions** | Toutes les fonctions SQL | 🟡 Tous niveaux | [KB Functions](https://mariadb.com/kb/en/built-in-functions/) |
| **Server Administration** | Configuration, optimisation | 🟠 Intermédiaire+ | [KB Admin](https://mariadb.com/kb/en/server-administration/) |
| **High Availability** | Galera, réplication, MaxScale | 🔴 Avancé | [KB HA](https://mariadb.com/kb/en/high-availability/) |
| **MariaDB Vector** | Type VECTOR, HNSW, IA | 🆕 Tous niveaux | [KB Vector](https://mariadb.com/kb/en/vector/) |
| **Development** | Plugins, UDFs, APIs | 🔴 Expert | [KB Development](https://mariadb.com/kb/en/development/) |

### Release Notes et Changelogs

- 🆕 [MariaDB 11.8 Release Notes](https://mariadb.com/kb/en/mariadb-1180-release-notes/)
- 📋 [MariaDB 11.4 Release Notes](https://mariadb.com/kb/en/mariadb-1140-release-notes/)
- 📋 [MariaDB 10.11 Release Notes](https://mariadb.com/kb/en/mariadb-10110-release-notes/)
- 📅 [All Release Notes](https://mariadb.com/kb/en/release-notes/)

### Documentation API et Connecteurs

| Langage/Outil | Documentation | Qualité |
|---------------|---------------|---------|
| **C Connector** | [MariaDB Connector/C](https://mariadb.com/kb/en/mariadb-connector-c/) | ⭐⭐⭐⭐⭐ |
| **Java Connector** | [MariaDB Connector/J](https://mariadb.com/kb/en/about-mariadb-connector-j/) | ⭐⭐⭐⭐⭐ |
| **Python** | [MariaDB Connector/Python](https://mariadb.com/kb/en/mariadb-connector-python/) | ⭐⭐⭐⭐ |
| **Node.js** | [MariaDB Connector/Node.js](https://mariadb.com/kb/en/nodejs-connector/) | ⭐⭐⭐⭐ |
| **ODBC** | [MariaDB Connector/ODBC](https://mariadb.com/kb/en/mariadb-connector-odbc/) | ⭐⭐⭐⭐ |

💡 **Astuce** : Utiliser la barre de recherche du KB avec des mots-clés précis en anglais pour trouver rapidement l'info recherchée.

---

## 2️⃣ Communautés et Forums

### Zulip Chat (Officiel)

**URL** : [https://mariadb.zulipchat.com/](https://mariadb.zulipchat.com/)

Chat en temps réel avec la communauté MariaDB.

**Channels recommandés** :
- `#general` - Discussions générales
- `#support` - Support technique
- `#development` - Développement MariaDB
- `#mariadb-vector` - Discussions sur Vector/IA
- `#galera` - Galera Cluster

**Avantages** :
- ✅ Réponses rapides (< 1 heure en moyenne)
- ✅ Développeurs MariaDB actifs
- ✅ Historique consultable
- ✅ Gratuit

---

### Mailing Lists

**URL** : [https://mariadb.org/get-involved/mailing-lists/](https://mariadb.org/get-involved/mailing-lists/)

#### Lists principales

| Liste | Description | Volume | Audience |
|-------|-------------|--------|----------|
| **maria-discuss** | Discussions générales | 🟡 Moyen | Tous |
| **maria-developers** | Développement serveur | 🟢 Faible | Développeurs |
| **maria-docs** | Documentation | 🟢 Très faible | Contributeurs docs |
| **commits** | Notifications commits Git | 🔴 Élevé | Développeurs actifs |

💡 **Conseil** : S'inscrire à `maria-discuss` pour suivre les discussions importantes de la communauté.

---

### StackOverflow

**URL** : [https://stackoverflow.com/questions/tagged/mariadb](https://stackoverflow.com/questions/tagged/mariadb)

**Tag** : `[mariadb]`

**Statistiques** :
- 15,000+ questions
- Réponse moyenne : 2-4 heures
- Taux de réponse : ~75%

**Bonnes pratiques** :
- ✅ Rechercher avant de poster (souvent déjà répondu)
- ✅ Fournir version MariaDB, système d'exploitation
- ✅ Inclure requête SQL et schéma si pertinent
- ✅ Montrer ce qui a déjà été tenté

---

### Reddit

**URL** : [https://www.reddit.com/r/MariaDB/](https://www.reddit.com/r/MariaDB/)

**Subreddit** : `/r/MariaDB`

**Abonnés** : ~5,000

**Type de contenu** :
- Questions/réponses techniques
- Annonces de releases
- Discussions architecture
- Partage d'expériences

---

### Social Media

| Plateforme | Compte Officiel | Intérêt |
|------------|-----------------|---------|
| **Twitter/X** | [@mariadb](https://twitter.com/mariadb) | 🔥 Annonces, actualités |
| **LinkedIn** | [MariaDB](https://www.linkedin.com/company/mariadb-corporation/) | 📊 Actualités corporate, jobs |
| **YouTube** | [MariaDB](https://www.youtube.com/c/MariaDBServer) | 🎥 Webinars, tutoriels |
| **Mastodon** | [@mariadb@fosstodon.org](https://fosstodon.org/@mariadb) | 🆕 Alternative Twitter |

---

## 3️⃣ Blogs et Publications Techniques

### Blog Officiel MariaDB

**URL** : [https://mariadb.org/blog/](https://mariadb.org/blog/)

**Fréquence** : 2-3 articles/semaine

**Thèmes** :
- Annonces de releases
- Tutorials techniques
- Best practices
- Case studies clients

**Articles recommandés (2024-2025)** :
- 🆕 "Introducing MariaDB Vector for AI Applications"
- ⚡ "Performance Improvements in MariaDB 11.8"
- 🔒 "Security Enhancements: TLS by Default"
- 📊 "Building RAG Applications with MariaDB"

---

### Planet MariaDB

**URL** : [https://planet.mariadb.org/](https://planet.mariadb.org/)

Agrégateur de **blogs de la communauté** MariaDB.

**Blogueurs notables** :
- **Monty (Michael Widenius)** - Créateur de MariaDB
- **Otto Kekäläinen** - MariaDB Debian/Ubuntu
- **Daniel Bartholomew** - MariaDB Foundation
- **Federico Razzoli** - MariaDB consultant

💡 **S'abonner au RSS** pour ne rien manquer des publications communautaires.

---

### Blogs Techniques Indépendants Recommandés

| Blog | Auteur | Focus | Niveau |
|------|--------|-------|--------|
| **Percona Database Performance Blog** | Percona Team | Performance, tuning | 🟠 Avancé |
| **MySQL Server Blog** | Oracle MySQL Team | Comparaisons MySQL/MariaDB | 🟡 Intermédiaire |
| **Federico Razzoli's Blog** | Federico Razzoli | MariaDB avancé, HA | 🔴 Expert |
| **Vettabase Blog** | Vettabase Team | MariaDB, PostgreSQL | 🟡 Intermédiaire |
| **ScaleGrid Blog** | ScaleGrid Team | Cloud, DBaaS | 🟢 Tous niveaux |

**URLs** :
- Percona : [https://www.percona.com/blog/](https://www.percona.com/blog/)
- Razzoli : [https://federico-razzoli.com/](https://federico-razzoli.com/)
- Vettabase : [https://vettabase.com/blog/](https://vettabase.com/blog/)

---

### Publications Académiques et Research

**Bases de données** :
- 📚 [ACM Digital Library](https://dl.acm.org/) - Recherche "MariaDB"
- 📚 [arXiv.org](https://arxiv.org/) - Catégorie "Databases"
- 📚 [VLDB](http://www.vldb.org/) - Very Large Data Bases conference

**Papers recommandés** :
- "Efficient Approximate Nearest Neighbor Search in High Dimensions" (HNSW algorithm)
- "MariaDB Galera Cluster: Performance Evaluation"
- "Comparative Study: MariaDB vs PostgreSQL for OLTP Workloads"

---

## 4️⃣ Conférences et Événements

### Événements Majeurs

#### MariaDB Server Fest 🎉

**Description** : Conférence annuelle officielle MariaDB  
**Fréquence** : 1 fois/an (Juin)  
**Localisation** : Helsinki, Finlande + Virtual  
**Prix** : Gratuit (virtuel), ~200€ (présentiel)  
**Durée** : 2 jours

**Programme type** :
- Keynotes MariaDB Foundation
- Annonces produits
- Technical deep-dives
- Workshops pratiques
- Networking

**URL** : [https://mariadb.org/fest/](https://mariadb.org/fest/)

💡 **Prochaine édition** : Juin 2026 (annonce Q1 2026)

---

#### Percona Live 🔥

**Description** : Conférence majeure sur bases de données open-source  
**Fréquence** : 1 fois/an (Mai)  
**Localisation** : USA (Austin, Denver) + Europe  
**Prix** : ~500-800€  
**Durée** : 3 jours

**Thèmes** :
- MySQL, MariaDB, PostgreSQL
- Performance tuning
- Cloud databases
- High availability

**URL** : [https://www.percona.com/live/](https://www.percona.com/live/)

---

#### FOSDEM 🐧

**Description** : Conférence open-source européenne  
**Fréquence** : 1 fois/an (Février)  
**Localisation** : Bruxelles, Belgique  
**Prix** : **Gratuit**  
**Durée** : 2 jours (week-end)

**DevRoom MariaDB** :
- Presentations communautaires
- Lightning talks
- Nouveautés MariaDB
- Networking

**URL** : [https://fosdem.org/](https://fosdem.org/)

💡 **Astuce** : Gratuit, excellent pour networking, réserver hôtel 6 mois à l'avance.

---

#### PGConf / PostgresConf 🐘

**Intérêt pour MariaDB** : Comparaisons, migration patterns, features communes

**URL** : [https://postgresconf.org/](https://postgresconf.org/)

---

### Meetups et Groupes Locaux

**Rechercher** : [https://www.meetup.com/topics/mariadb/](https://www.meetup.com/topics/mariadb/)

**Principaux groupes** :
- 🇫🇷 Paris MariaDB Meetup
- 🇬🇧 London MariaDB Users Group
- 🇺🇸 San Francisco MySQL/MariaDB Meetup
- 🇩🇪 Berlin Database Meetup

**Fréquence** : 1-2 fois/trimestre

**Format** :
- Présentations techniques (45 min)
- Lightning talks (5-10 min)
- Networking (1h)
- Gratuit, souvent avec pizza/bière 🍕🍺

---

### Webinars MariaDB

**URL** : [https://mariadb.com/resources/webinars/](https://mariadb.com/resources/webinars/)

**Fréquence** : 2-4 par mois

**Types** :
- Product announcements
- Technical deep-dives
- Customer case studies
- Best practices

**Avantages** :
- ✅ Gratuit
- ✅ Enregistrés (replay disponible)
- ✅ Q&A avec experts
- ✅ Certificat de participation

**Webinars recommandés (archives)** :
- "MariaDB Vector: AI-Powered Database" (Nov 2025)
- "Migrating from MySQL to MariaDB 11.8" (Oct 2025)
- "High Availability with Galera Cluster" (Sept 2025)

---

## 5️⃣ Formation et Certification

### Formations Officielles MariaDB

**URL** : [https://mariadb.com/services/training/](https://mariadb.com/services/training/)

#### Cursus disponibles

| Cours | Durée | Niveau | Prix |
|-------|-------|--------|------|
| **MariaDB Essentials** | 3 jours | 🟢 Débutant | ~1,500€ |
| **MariaDB Administration** | 4 jours | 🟡 Intermédiaire | ~2,000€ |
| **MariaDB Performance Tuning** | 2 jours | 🟠 Avancé | ~1,500€ |
| **MariaDB High Availability** | 3 jours | 🟠 Avancé | ~1,800€ |
| **MariaDB Developer** | 3 jours | 🟡 Intermédiaire | ~1,600€ |

**Formats** :
- 🏫 Présentiel (public/intra-entreprise)
- 💻 Virtuel (live online)
- 📹 E-learning (self-paced)

---

### Certifications MariaDB

**URL** : [https://mariadb.com/services/certification/](https://mariadb.com/services/certification/)

#### Niveaux de certification

| Certification | Niveau | Examen | Prix |
|---------------|--------|--------|------|
| **MariaDB Certified Associate** | 🟢 Basique | 1h, 40 QCM | ~150€ |
| **MariaDB Certified Professional DBA** | 🟡 Avancé | 2h, 60 QCM + pratique | ~300€ |
| **MariaDB Certified Expert** | 🔴 Expert | 3h, pratique avancée | ~500€ |

**Validité** : 3 ans (recertification recommandée)

💡 **Conseil** : Suivre formation officielle avant certification pour maximiser chances de succès.

---

### MOOCs et Cours en Ligne

| Plateforme | Cours | Niveau | Prix |
|------------|-------|--------|------|
| **Udemy** | "Complete MariaDB Masterclass" | 🟡 Tous | ~20€ |
| **Coursera** | "Database Systems" (inclut MariaDB) | 🟢 Débutant | Gratuit (audit) |
| **LinkedIn Learning** | "MariaDB Essential Training" | 🟡 Intermédiaire | Abonnement |
| **YouTube** | Chaîne MariaDB officielle | 🟢 Tous | Gratuit |

---

## 6️⃣ Code et Contribution

### GitHub - MariaDB Server

**URL** : [https://github.com/MariaDB/server](https://github.com/MariaDB/server)

**Stars** : ~5,500  
**Forks** : ~1,800  
**Contributors** : 400+

**Branches principales** :
- `11.8` - Version LTS actuelle
- `11.4` - Version LTS précédente
- `10.11` - Version LTS legacy
- `main` - Development branch

**Comment contribuer** :
1. Fork le repository
2. Créer une branche feature
3. Commits avec tests
4. Pull Request + description
5. Code review par mainteneurs

**Guide de contribution** : [CONTRIBUTING.md](https://github.com/MariaDB/server/blob/11.8/CONTRIBUTING.md)

---

### Jira - Bug Tracking

**URL** : [https://jira.mariadb.org/](https://jira.mariadb.org/)

**Utilisation** :
- 🐛 Reporter des bugs
- 💡 Proposer des features
- 📊 Suivre roadmap
- ✅ Vérifier status de bugs

**Projets principaux** :
- `MDEV` - MariaDB Server
- `CONC` - Connectors
- `MXS` - MaxScale
- `MCOL` - ColumnStore

💡 **Astuce** : Chercher si le bug existe déjà avant de créer un nouveau ticket.

---

### Projets Satellites

| Projet | Description | GitHub |
|--------|-------------|--------|
| **mariadb-operator** | Kubernetes operator | [mariadb-operator/mariadb-operator](https://github.com/mariadb-operator/mariadb-operator) |
| **MaxScale** | Database proxy | [mariadb-corporation/MaxScale](https://github.com/mariadb-corporation/MaxScale) |
| **Galera** | Cluster replication | [codership/galera](https://github.com/codership/galera) |
| **MariaDB Connector/J** | Java connector | [mariadb-corporation/mariadb-connector-j](https://github.com/mariadb-corporation/mariadb-connector-j) |

---

## 🔧 Outils et Utilitaires Communautaires

### Outils de Monitoring

| Outil | Description | URL |
|-------|-------------|-----|
| **PMM (Percona Monitoring)** | Monitoring complet | [percona.com/software/pmm](https://www.percona.com/software/database-tools/percona-monitoring-and-management) |
| **mysqld_exporter** | Prometheus exporter | [prometheus/mysqld_exporter](https://github.com/prometheus/mysqld_exporter) |
| **Grafana Dashboards** | Dashboards communautaires | [grafana.com/dashboards](https://grafana.com/grafana/dashboards/) |

### Outils de Migration

| Outil | Description | URL |
|-------|-------------|-----|
| **gh-ost** | Online schema change | [github/gh-ost](https://github.com/github/gh-ost) |
| **pt-online-schema-change** | Percona Toolkit | [percona/toolkit](https://www.percona.com/software/database-tools/percona-toolkit) |
| **mydumper/myloader** | Parallel backup/restore | [mydumper](https://github.com/mydumper/mydumper) |

---

## 📬 S'Abonner aux Actualités

### Newsletters Recommandées

| Newsletter | Fréquence | Focus | Inscription |
|------------|-----------|-------|-------------|
| **MariaDB Newsletter** | Mensuelle | Nouvelles produit | [mariadb.org/newsletter](https://mariadb.org/newsletter/) |
| **Percona Newsletter** | Bi-mensuelle | Performance, tips | [percona.com/newsletter](https://www.percona.com/resources/newsletter) |
| **DBWeekly** | Hebdomadaire | Toutes BDD | [dbweekly.com](https://dbweekly.com/) |

### RSS Feeds

- 📡 MariaDB Blog : `https://mariadb.org/feed/`
- 📡 Planet MariaDB : `https://planet.mariadb.org/atom.xml`
- 📡 MariaDB KB : `https://mariadb.com/kb/en/feed/`

---

## 🌍 Communauté Francophone

### Ressources en Français

| Ressource | Type | URL |
|-----------|------|-----|
| **MariaDB.fr** | Documentation FR (non officielle) | [mariadb.fr](https://mariadb.fr/) |
| **LinuxFr** | Actualités tech FR | [linuxfr.org](https://linuxfr.org/) (tag MariaDB) |
| **Developpez.com** | Forum FR | [developpez.com](https://www.developpez.com/forums/) |
| **Discord BDD FR** | Chat communautaire | Rechercher "Bases de données FR" |

### Groupes Meetup Francophones

- 🇫🇷 **Paris MariaDB User Group**
- 🇧🇪 **Brussels Database Meetup**
- 🇨🇭 **Genève/Lausanne Tech Meetup**
- 🇨🇦 **Montréal Database Community**

---

## ✅ Checklist d'Apprentissage Continu

### Routine recommandée

**Quotidien** (15 min) :
- [ ] Consulter MariaDB Twitter/LinkedIn
- [ ] Vérifier Zulip pour discussions importantes

**Hebdomadaire** (1-2h) :
- [ ] Lire 2-3 articles de blogs techniques
- [ ] Parcourir nouvelles questions StackOverflow
- [ ] Tester 1 nouvelle feature MariaDB

**Mensuel** (3-4h) :
- [ ] Lire release notes versions récentes
- [ ] Suivre 1 webinar MariaDB
- [ ] Contribuer à 1 discussion communautaire

**Annuel** :
- [ ] Assister à 1 conférence (FOSDEM, Percona Live, etc.)
- [ ] Mettre à jour connaissances via formation
- [ ] Partager expérience (blog post, présentation)

---

## 📊 Matrice de Ressources par Objectif

| Objectif | Ressources Prioritaires | Temps |
|----------|------------------------|-------|
| **Apprendre les bases** | KB Getting Started + YouTube tutorials | 10-20h |
| **Résoudre un problème** | StackOverflow + Zulip + KB search | 1-4h |
| **Optimiser performance** | Percona Blog + KB Performance + PMM | 20-40h |
| **Implémenter HA** | KB Galera + MaxScale docs + Webinars | 40-80h |
| **Contribuer au code** | GitHub + Jira + Mailing lists dev | 100h+ |
| **Rester à jour** | Newsletter + Twitter + Planet MariaDB | 2h/semaine |

---

## 🎓 Parcours de Montée en Compétences

### Débutant → Intermédiaire (3-6 mois)

1. **Mois 1-2** : KB Getting Started + Udemy course
2. **Mois 3-4** : MariaDB Essentials training (officiel)
3. **Mois 5-6** : Certification Associate + Projets pratiques

### Intermédiaire → Avancé (6-12 mois)

1. **Mois 1-3** : KB Administration + Performance blog posts
2. **Mois 4-6** : MariaDB Admin training (officiel)
3. **Mois 7-9** : Projets HA (Galera) + Webinars
4. **Mois 10-12** : Certification Professional DBA

### Avancé → Expert (12-24 mois)

1. **Mois 1-6** : Source code exploration + Contribution mineure
2. **Mois 7-12** : Mailing lists dev + Conférences
3. **Mois 13-18** : Contributions majeures + Blog posts techniques
4. **Mois 19-24** : Certification Expert + Speaking à conférences

---

## ✅ Points Clés à Retenir

- **MariaDB KB** est la ressource #1 pour la documentation technique
- **Zulip Chat** pour support rapide et discussions communautaires
- **Planet MariaDB** pour suivre l'actualité de la communauté
- **MariaDB Server Fest** événement annuel gratuit (virtuel)
- **FOSDEM** meilleure conférence gratuite en Europe
- **Certifications** disponibles en 3 niveaux (Associate, Professional, Expert)
- **GitHub** pour contribuer au code et suivre développement
- **Routine apprentissage** : 15 min/jour, 2h/semaine, 1 conf/an
- **Communauté francophone** existe mais principalement anglophone
- **Percona Blog** excellent complément pour performance/tuning

---

## 🔗 Liens Rapides Essentiels

### Top 10 des URLs à Bookmarker

1. 📖 [MariaDB KB](https://mariadb.com/kb/en/) - Documentation officielle
2. 💬 [Zulip Chat](https://mariadb.zulipchat.com/) - Support communautaire
3. 📝 [MariaDB Blog](https://mariadb.org/blog/) - Actualités officielles
4. 🌍 [Planet MariaDB](https://planet.mariadb.org/) - Blogs communauté
5. ❓ [StackOverflow MariaDB](https://stackoverflow.com/questions/tagged/mariadb) - Q&A
6. 🐛 [Jira MariaDB](https://jira.mariadb.org/) - Bug tracking
7. 💻 [GitHub MariaDB](https://github.com/MariaDB/server) - Code source
8. 🎓 [MariaDB Training](https://mariadb.com/services/training/) - Formations
9. 🎉 [MariaDB Server Fest](https://mariadb.org/fest/) - Conférence annuelle
10. 📊 [Release Notes](https://mariadb.com/kb/en/release-notes/) - Toutes versions

---

## 📑 Sous-sections de cette Annexe

- **H.1** [Documentation officielle](./01-documentation-officielle.md) - Guide détaillé du KB
- **H.2** [Communautés et forums](./02-communautes-forums.md) - Participer aux discussions
- **H.3** [Blogs techniques recommandés](./03-blogs-techniques.md) - Sélection curatée
- **H.4** [Conférences et événements](./04-conferences-evenements.md) - Calendrier détaillé


---

💡 **Conseil final** : La meilleure façon d'apprendre est de **pratiquer régulièrement** et de **participer à la communauté**. N'hésitez pas à poser des questions sur Zulip ou StackOverflow, la communauté MariaDB est accueillante et prête à aider ! 🚀

⏭️ [Documentation officielle](/annexes/ressources-documentation/01-documentation-officielle.md)
