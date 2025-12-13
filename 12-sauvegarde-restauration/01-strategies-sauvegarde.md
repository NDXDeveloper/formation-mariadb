🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.1 Stratégies de sauvegarde : Full, Incrémentale, Différentielle

> **Niveau** : Avancé  
> **Durée estimée** : 2 heures  
> **Prérequis** : Introduction au chapitre 12, Notions de RPO/RTO

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** les différences fondamentales entre Full, Incrémentale et Différentielle
- **Choisir** la stratégie adaptée en fonction des contraintes RPO/RTO et ressources
- **Dimensionner** l'espace de stockage et calculer les fenêtres de sauvegarde
- **Concevoir** des stratégies hybrides optimales pour la production
- **Évaluer** l'impact sur les performances et les temps de restauration

---

## Introduction

Le choix de la stratégie de sauvegarde est une décision architecturale critique qui impacte directement :

- **Les coûts** : Stockage, bande passante, infrastructure
- **Les performances** : Impact sur la production, utilisation CPU/I/O
- **La complexité opérationnelle** : Nombre d'étapes de restauration, risques d'erreur
- **Les SLA** : Capacité à respecter les objectifs RPO/RTO

Il n'existe pas de stratégie universelle — le choix optimal dépend du contexte métier, des volumes de données, de la fréquence de changement et des contraintes réglementaires.

💡 **Principe clé** : *"La meilleure stratégie de sauvegarde est celle que vous pouvez restaurer rapidement et fiablement"*.

---

## Sauvegarde complète (Full Backup)

### Principe

Une sauvegarde complète copie **l'intégralité** des données de la base de données à un instant T, sans tenir compte de l'état des sauvegardes précédentes.

```
┌────────────────────────────────────────────────────┐
│              Sauvegarde Full (100 Go)              │
├────────────────────────────────────────────────────┤
│                                                    │
│  ┌──────────────────────────────────────────────┐  │
│  │ Base de données complète au 2025-12-13 02:00 │  │
│  │  - Toutes les tables                         │  │
│  │  - Tous les index                            │  │
│  │  - Toutes les données                        │  │
│  │  - Structures système                        │  │
│  └──────────────────────────────────────────────┘  │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Caractéristiques techniques

**Avantages** :
- ✅ **Simplicité** : Une seule unité de sauvegarde autosuffisante
- ✅ **Restauration rapide** : Pas de dépendances entre fichiers
- ✅ **Fiabilité maximale** : Pas de chaîne de dépendances qui pourrait se rompre
- ✅ **Gestion simple** : Rétention claire (garder N sauvegardes complètes)
- ✅ **PITR simplifié** : Un seul point de départ pour le Point-in-Time Recovery

**Inconvénients** :
- ❌ **Volumétrie** : Consommation d'espace maximale (100% des données à chaque fois)
- ❌ **Temps d'exécution** : Fenêtre de backup longue pour les grosses bases
- ❌ **Impact I/O** : Lecture intégrale de toutes les données
- ❌ **Coût réseau** : Transfert important vers stockage distant
- ❌ **RPO large** : Si sauvegarde quotidienne, jusqu'à 24h de perte potentielle

### Cas d'usage recommandés

**Parfait pour** :
- Petites bases de données (< 100 Go)
- Bases à faible taux de changement
- Environnements de développement/test
- Point de référence hebdomadaire ou mensuel

**Exemple de dimensionnement** :

```
Base de données : 500 Go
Taux de compression : 60% (après compression)
Stockage effectif par backup : 200 Go
Rétention : 30 jours

Espace total requis = 200 Go × 30 = 6 To
```

### Implémentation avec MariaDB

**Sauvegarde logique (mariadb-dump)** :
```bash
#!/bin/bash
# full_backup_logical.sh

BACKUP_DIR="/backups/full/$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"

mariadb-dump \
  --single-transaction \
  --routines \
  --triggers \
  --events \
  --all-databases \
  --result-file="$BACKUP_DIR/full_backup.sql"

# Compression
gzip "$BACKUP_DIR/full_backup.sql"

# Checksum pour validation
md5sum "$BACKUP_DIR/full_backup.sql.gz" > "$BACKUP_DIR/full_backup.md5"

echo "Full backup completed: $(du -h $BACKUP_DIR/full_backup.sql.gz)"
```

**Sauvegarde physique (Mariabackup)** :
```bash
#!/bin/bash
# full_backup_physical.sh

BACKUP_DIR="/backups/mariabackup/full/$(date +%Y%m%d_%H%M%S)"

mariabackup --backup \
  --target-dir="$BACKUP_DIR" \
  --user=backup_user \
  --password="$BACKUP_PASSWORD" \
  --compress \
  --compress-threads=4

# Vérification de l'intégrité
mariabackup --prepare --target-dir="$BACKUP_DIR"

echo "Full backup completed: $(du -sh $BACKUP_DIR)"
```

### Métriques de performance

| Taille base | Temps backup | Temps restauration | Débit I/O moyen |
|-------------|--------------|-------------------|-----------------|
| 100 Go      | 15-20 min    | 20-25 min         | 100-120 MB/s    |
| 500 Go      | 1h15-1h30    | 1h30-2h           | 110-130 MB/s    |
| 1 To        | 2h30-3h      | 3h-3h30           | 120-140 MB/s    |
| 5 To        | 12h-14h      | 14h-16h           | 120-150 MB/s    |

⚠️ **Attention** : Ces temps sont indicatifs et dépendent fortement du hardware (type de disques, bande passante réseau, CPU).

---

## Sauvegarde incrémentale (Incremental Backup)

### Principe

Une sauvegarde incrémentale copie **uniquement les modifications** survenues depuis la **dernière sauvegarde** (qu'elle soit complète ou incrémentale).

```
Timeline de sauvegardes incrémentales sur 1 semaine
────────────────────────────────────────────────────────

Dimanche     Lundi      Mardi      Mercredi    Jeudi
┌──────┐    ┌────┐     ┌────┐     ┌─────┐    ┌────┐
│ FULL │───►│Inc1│────►│Inc2│────►│ Inc3│───►│Inc4│
│100 Go│    │ 5Go│     │ 6Go│     │  4Go│    │ 7Go│
└──────┘    └────┘     └────┘     └─────┘    └────┘
            (depuis    (depuis     (depuis    (depuis
             FULL)      Inc1)       Inc2)      Inc3)

Chaîne de restauration complète jusqu'à jeudi :
FULL → Inc1 → Inc2 → Inc3 → Inc4
```

### Caractéristiques techniques

**Avantages** :
- ✅ **Espace minimal** : Ne sauvegarde que les deltas
- ✅ **Fenêtre courte** : Backup rapide (quelques minutes vs heures)
- ✅ **Impact faible** : Moins de données à lire/écrire
- ✅ **RPO réduit** : Possibilité de sauvegardes très fréquentes (toutes les heures)
- ✅ **Bande passante optimisée** : Transferts réseau minimaux

**Inconvénients** :
- ❌ **Restauration complexe** : Nécessite FULL + TOUS les incrémentaux dans l'ordre
- ❌ **Chaîne fragile** : Perte d'un maillon = restauration impossible au-delà
- ❌ **Temps de restauration** : Plus long que full (application séquentielle)
- ❌ **Gestion complexe** : Rétention délicate, impossibilité de supprimer un maillon
- ❌ **Risque accru** : Probabilité de corruption augmente avec le nombre de fichiers

### Cas d'usage recommandés

**Parfait pour** :
- Grosses bases de données (> 1 To) avec taux de changement modéré
- Besoins de RPO agressif (< 4 heures)
- Environnements avec contraintes de fenêtre de backup
- Bande passante réseau limitée

**Exemple de dimensionnement** :

```
Base de données : 2 To
Taux de changement quotidien : 2% (40 Go)
Compression : 60%

Full hebdomadaire : 800 Go (2 To × 40%)
Incrémental quotidien : 16 Go (40 Go × 40%)

Stockage pour 4 semaines :
- 4 Full : 4 × 800 Go = 3.2 To
- 28 Inc : 28 × 16 Go = 448 Go
Total : ~3.7 To (vs 22.4 To avec full quotidien)
```

### Implémentation avec Mariabackup

**Premier backup complet** :
```bash
#!/bin/bash
# incremental_01_full.sh

FULL_DIR="/backups/incremental/full/$(date +%Y%m%d)"

mariabackup --backup \
  --target-dir="$FULL_DIR" \
  --user=backup_user \
  --password="$BACKUP_PASSWORD"

echo "Full base created: $FULL_DIR"
```

**Backups incrémentaux quotidiens** :
```bash
#!/bin/bash
# incremental_02_daily.sh

FULL_DIR="/backups/incremental/full/20251207"  # Base de référence
INC_DIR="/backups/incremental/inc/$(date +%Y%m%d_%H%M%S)"

# Trouver le dernier backup (full ou dernier inc)
LAST_BACKUP=$(find /backups/incremental -type d -name "inc_*" | sort -r | head -1)
if [ -z "$LAST_BACKUP" ]; then
  LAST_BACKUP="$FULL_DIR"
fi

mariabackup --backup \
  --target-dir="$INC_DIR" \
  --incremental-basedir="$LAST_BACKUP" \
  --user=backup_user \
  --password="$BACKUP_PASSWORD"

echo "Incremental backup created: $INC_DIR (based on $LAST_BACKUP)"
```

**Restauration incrémentale** :
```bash
#!/bin/bash
# incremental_03_restore.sh

FULL_DIR="/backups/incremental/full/20251207"
INC1_DIR="/backups/incremental/inc/20251208_020000"
INC2_DIR="/backups/incremental/inc/20251209_020000"
INC3_DIR="/backups/incremental/inc/20251210_020000"

# Étape 1 : Préparer la base complète
mariabackup --prepare --apply-log-only --target-dir="$FULL_DIR"

# Étape 2 : Appliquer chaque incrément dans l'ordre
mariabackup --prepare --apply-log-only \
  --target-dir="$FULL_DIR" \
  --incremental-dir="$INC1_DIR"

mariabackup --prepare --apply-log-only \
  --target-dir="$FULL_DIR" \
  --incremental-dir="$INC2_DIR"

# Étape 3 : Dernier incrément (sans --apply-log-only)
mariabackup --prepare \
  --target-dir="$FULL_DIR" \
  --incremental-dir="$INC3_DIR"

# Étape 4 : Restauration finale
mariabackup --copy-back --target-dir="$FULL_DIR"
chown -R mysql:mysql /var/lib/mysql
```

⚠️ **CRITICAL** : L'ordre des incrémentaux est absolu. Une erreur dans la séquence rend la restauration impossible.

### Métriques de performance

| Scénario | Taille backup | Temps backup | Temps restauration |
|----------|---------------|--------------|-------------------|
| Full 2 To | 800 Go | 4h | 5h |
| Inc quotidien (2%) | 16 Go | 15 min | +30 min par inc |
| Restauration J+7 | Full + 7 Inc | - | 5h + 3h30 = 8h30 |

---

## Sauvegarde différentielle (Differential Backup)

### Principe

Une sauvegarde différentielle copie **toutes les modifications** survenues depuis la **dernière sauvegarde complète**, indépendamment des différentielles précédentes.

```
Timeline de sauvegardes différentielles sur 1 semaine
─────────────────────────────────────────────────────

Dimanche     Lundi       Mardi      Mercredi    Jeudi
┌──────┐    ┌────┐      ┌────┐      ┌─────┐    ┌────┐
│ FULL │    │Dif1│      │Dif2│      │ Dif3│    │Dif4│
│100 Go│    │ 5Go│      │11Go│      │ 15Go│    │22Go│
└──────┘    └──┬─┘      └──┬─┘      └──┬──┘    └──┬─┘
   │           │           │           │          │
   └───────────┴───────────┴───────────┴──────────┘
        (toutes basées sur FULL du dimanche)

Chaîne de restauration jusqu'à jeudi :
FULL → Dif4  (seulement 2 fichiers !)
```

### Caractéristiques techniques

**Avantages** :
- ✅ **Restauration simple** : FULL + dernière différentielle uniquement
- ✅ **Fiabilité** : Moins de dépendances que l'incrémentale
- ✅ **RPO correct** : Backups fréquents possibles
- ✅ **Gestion souple** : Possibilité de supprimer les différentielles anciennes
- ✅ **Compromis optimal** : Entre full (simple) et incrémentale (économe)

**Inconvénients** :
- ❌ **Croissance progressive** : Chaque différentielle grossit jusqu'au prochain full
- ❌ **Redondance** : Données modifiées en début de cycle sauvegardées N fois
- ❌ **Fenêtre croissante** : Temps de backup augmente au fil de la semaine
- ❌ **Espace intermédiaire** : Plus que incrémentale, moins que full quotidien

### Cas d'usage recommandés

**Parfait pour** :
- Bases moyennes à grandes (500 Go - 2 To)
- Équilibre entre simplicité et économie d'espace
- Environnements où la fenêtre de backup peut varier
- Compromis production entre RPO et complexité

**Exemple de dimensionnement** :

```
Base de données : 1 To
Taux de changement : 3% quotidien (30 Go)
Compression : 60%

Full hebdomadaire : 400 Go
Différentielles (cumul sur 7 jours) :
  Lundi : 12 Go (3%)
  Mardi : 24 Go (6%)
  Mercredi : 36 Go (9%)
  Jeudi : 48 Go (12%)
  Vendredi : 60 Go (15%)
  Samedi : 72 Go (18%)

Stockage pour 4 semaines :
- 4 Full : 4 × 400 Go = 1.6 To
- 4 × 6 Dif : 4 × (12+24+36+48+60+72) = 1.01 To
Total : ~2.6 To
```

### Implémentation avec Mariabackup

**Backup différentiel** :
```bash
#!/bin/bash
# differential_backup.sh

FULL_DIR="/backups/differential/full/$(date +%Y%m%d -d 'last Sunday')"
DIFF_DIR="/backups/differential/diff/$(date +%Y%m%d_%H%M%S)"

# Vérifier que le full de référence existe
if [ ! -d "$FULL_DIR" ]; then
  echo "ERROR: Full backup not found at $FULL_DIR"
  exit 1
fi

# Créer la différentielle (toujours basée sur le même full)
mariabackup --backup \
  --target-dir="$DIFF_DIR" \
  --incremental-basedir="$FULL_DIR" \
  --user=backup_user \
  --password="$BACKUP_PASSWORD"

echo "Differential backup created: $DIFF_DIR"
echo "Size: $(du -sh $DIFF_DIR)"
```

**Restauration différentielle** :
```bash
#!/bin/bash
# differential_restore.sh

FULL_DIR="/backups/differential/full/20251207"
DIFF_DIR="/backups/differential/diff/20251212_020000"  # Dernière diff

# Étape 1 : Préparer la base complète
mariabackup --prepare --apply-log-only --target-dir="$FULL_DIR"

# Étape 2 : Appliquer la différentielle
mariabackup --prepare \
  --target-dir="$FULL_DIR" \
  --incremental-dir="$DIFF_DIR"

# Étape 3 : Restauration
mariabackup --copy-back --target-dir="$FULL_DIR"
chown -R mysql:mysql /var/lib/mysql

echo "Restoration complete from Full + Differential"
```

💡 **Simplification** : Avec la différentielle, seuls 2 fichiers sont nécessaires pour restaurer, contre potentiellement 7+ avec l'incrémentale.

### Métriques de performance

| Jour | Taille backup | Temps backup | Temps restauration |
|------|---------------|--------------|-------------------|
| Full (Dim) | 400 Go | 2h | 2h30 |
| Diff Lun | 12 Go | 10 min | 2h30 + 20 min |
| Diff Mar | 24 Go | 20 min | 2h30 + 25 min |
| Diff Jeu | 48 Go | 35 min | 2h30 + 35 min |
| Diff Sam | 72 Go | 50 min | 2h30 + 45 min |

---

## Tableau comparatif

| Critère | Full | Incrémentale | Différentielle |
|---------|------|--------------|----------------|
| **Espace de stockage** | ⭐ Maximum | ⭐⭐⭐ Minimum | ⭐⭐ Intermédiaire |
| **Temps de backup** | ⭐ Long | ⭐⭐⭐ Court | ⭐⭐ Variable |
| **Temps de restauration** | ⭐⭐⭐ Rapide | ⭐ Lent | ⭐⭐ Moyen |
| **Complexité** | ⭐⭐⭐ Simple | ⭐ Complexe | ⭐⭐ Modérée |
| **Fiabilité** | ⭐⭐⭐ Maximale | ⭐ Chaîne fragile | ⭐⭐ Bonne |
| **Gestion** | ⭐⭐⭐ Facile | ⭐ Délicate | ⭐⭐ Correcte |
| **RPO** | ⭐ 24h typ. | ⭐⭐⭐ < 1h possible | ⭐⭐ < 4h typ. |

### Matrice de décision

```
┌─────────────────────────────────────────────────────────┐
│        Choisir votre stratégie de sauvegarde            │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Taille DB < 200 Go                                     │
│  └──► FULL quotidien                                    │
│                                                         │
│  200 Go < Taille DB < 1 To + Changement modéré          │
│  └──► DIFFÉRENTIELLE (Full hebdo + Diff quotidien)      │
│                                                         │
│  Taille DB > 1 To + Changement < 5% quotidien           │
│  └──► INCRÉMENTALE (Full hebdo + Inc quotidien)         │
│                                                         │
│  Taille DB > 5 To + Changement important                │
│  └──► HYBRID (Full mensuel + Inc quotidien + Binlogs)   │
│                                                         │
│  RPO < 1h requis                                        │
│  └──► INCRÉMENTALE horaire + Binary logs continus       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Stratégies hybrides (Production-Ready)

En production, les stratégies pures sont rarement utilisées. On privilégie des **approches hybrides** combinant plusieurs types de backups.

### Modèle 1 : Full hebdomadaire + Différentielle quotidienne

**Configuration** :
- Dimanche 02:00 → Full backup
- Lundi-Samedi 02:00 → Differential backup
- En continu → Binary logs

```
Semaine type :
┌──────┬─────┬─────┬─────┬─────┬─────┬─────┐
│ Dim  │ Lun │ Mar │ Mer │ Jeu │ Ven │ Sam │
├──────┼─────┼─────┼─────┼─────┼─────┼─────┤
│ FULL │ Dif │ Dif │ Dif │ Dif │ Dif │ Dif │
│400 Go│ 12  │ 24  │ 36  │ 48  │ 60  │ 72  │
└──────┴─────┴─────┴─────┴─────┴─────┴─────┘

Restauration jeudi : Full (Dim) + Diff (Jeu) = 2 fichiers
RPO : 24h (ou < 1h avec binary logs)
```

**Avantages** :
- Restauration simple (2 fichiers max)
- Fenêtre de backup prévisible
- Bon compromis espace/temps

**Cas d'usage** : Bases 500 Go - 2 To, taux de changement < 5% quotidien

### Modèle 2 : Full mensuel + Incrémentale quotidienne

**Configuration** :
- 1er du mois 02:00 → Full backup
- Quotidien 02:00 → Incremental backup
- En continu → Binary logs

```
Mois type :
┌──────┬─────┬─────┬─────┬─────┬─────┬─────┐
│ J1   │ J2  │ J3  │ ... │ J28 │ J29 │ J30 │
├──────┼─────┼─────┼─────┼─────┼─────┼─────┤
│ FULL │ Inc │ Inc │ Inc │ Inc │ Inc │ Inc │
│800 Go│ 16  │ 16  │ 16  │ 16  │ 16  │ 16  │
└──────┴─────┴─────┴─────┴─────┴─────┴─────┘

Restauration J15 : Full (J1) + Inc2 + Inc3 + ... + Inc15 = 15 fichiers
RPO : 24h (ou < 1h avec binary logs)
```

**Avantages** :
- Stockage minimal
- Fenêtre de backup courte
- RPO flexible avec binary logs

**Inconvénients** :
- Restauration longue (nombreux fichiers)
- Chaîne de dépendance importante

**Cas d'usage** : Bases > 2 To, contraintes d'espace strictes

### Modèle 3 : Full hebdomadaire + Incrémentale horaire + Binary logs

**Configuration** :
- Dimanche 02:00 → Full backup
- Toutes les 4h → Incremental backup
- En continu → Binary logs (PITR)

```
Semaine avec RPO < 1h :
┌──────────────────────────────────────────────┐
│ Dimanche 02h : FULL                          │
│ Lun 02h, 06h, 10h, 14h, 18h, 22h : Inc       │
│ Mar 02h, 06h, 10h, 14h, 18h, 22h : Inc       │
│ ... (idem pour toute la semaine)             │
│ Binary logs : Rotation toutes les heures     │
└──────────────────────────────────────────────┘

Restauration mercredi 15h37 :
1. Full (Dimanche 02h)
2. Inc (Lun 02h, 06h, 10h, 14h, 18h, 22h)
3. Inc (Mar 02h, 06h, 10h, 14h, 18h, 22h)
4. Inc (Mer 02h, 06h, 10h, 14h)
5. Binary logs de 14h00 à 15h37

RPO effectif : < 1h (délai de détection + restauration)
```

**Avantages** :
- RPO très agressif
- Protection maximale contre la perte de données
- PITR à la seconde près

**Inconvénients** :
- Complexité opérationnelle élevée
- Nombreux fichiers à gérer
- Stockage important des binary logs

**Cas d'usage** : Environnements critiques (finance, santé, e-commerce)

### Modèle 4 : Stratégie GFS (Grandfather-Father-Son)

**Configuration** :
- Quotidien (Son) : Différentielle
- Hebdomadaire (Father) : Full
- Mensuel (Grandfather) : Full archivé longue durée

```
Cycle GFS sur 3 mois :
┌──────────────────────────────────────────────────────┐
│ Quotidien (7 jours)  : Différentielle                │
│ Hebdomadaire (4 sem) : Full (4 copies)               │
│ Mensuel (12 mois)    : Full archivé (3 copies)       │
└──────────────────────────────────────────────────────┘

Rétention totale : ~20 backups actifs
Stockage : 7 Diff + 4 Full + 3 Full_Archive
```

**Avantages** :
- Rétention long terme structurée
- Conformité réglementaire facilitée
- Granularité de restauration multiple

**Cas d'usage** : Exigences réglementaires (SOX, GDPR, HIPAA)

---

## Calculs de dimensionnement

### Formules essentielles

**Espace de stockage total** :
```
Stockage_Total = (N_Full × Taille_Full) + (N_Inc × Taille_Moyenne_Inc) + (N_Diff × Taille_Moyenne_Diff)
```

**Fenêtre de backup** :
```
Fenêtre_Backup = Taille_Backup / Débit_Effectif

Débit_Effectif = Débit_Disque × Taux_Compression × Efficacité_Réseau
```

**Temps de restauration** :
```
Temps_Restore_Full = Taille_Full / Débit_Restauration

Temps_Restore_Incremental = Temps_Restore_Full + (N_Inc × Temps_Application_Inc)

Temps_Application_Inc ≈ 20-30% du temps de lecture de l'incrément
```

### Exemple concret de dimensionnement

**Contexte** :
- Base de données : 3 To
- Taux de changement quotidien : 2% (60 Go)
- Compression : 65% (taux effectif)
- RPO cible : 6 heures
- RTO cible : 4 heures
- Rétention : 30 jours

**Stratégie choisie** : Full hebdomadaire + Incrémentale quotidienne + Binary logs

**Calculs** :

```
Taille après compression :
- Full : 3 To × 0.65 = 1.95 To
- Inc quotidien : 60 Go × 0.65 = 39 Go

Stockage pour 30 jours (4 semaines complètes) :
- Full : 4 × 1.95 To = 7.8 To
- Inc : 28 × 39 Go = 1.09 To
- Binary logs (30 jours, ~10 Go/jour) : 300 Go

Total : 7.8 + 1.09 + 0.3 = 9.19 To
Avec marge 20% : ~11 To requis

Fenêtre de backup :
- Full : 1.95 To / 150 MB/s = ~3h40
- Inc : 39 Go / 150 MB/s = ~4 min

Temps de restauration (pire cas - J+30) :
- Full : 1.95 To / 200 MB/s = ~2h50
- Application 28 Inc : 28 × 5 min = 2h20
- Application binary logs : ~20 min
Total : ~5h30

⚠️ RTO non respecté ! Ajustement nécessaire.
```

**Ajustement de stratégie** :

Pour respecter RTO 4h, passer à Full tous les 3 jours :

```
Stockage ajusté :
- Full (tous les 3 jours) : 10 × 1.95 To = 19.5 To
- Inc : 20 × 39 Go = 0.78 To
- Binary logs : 300 Go

Total : 19.5 + 0.78 + 0.3 = 20.58 To

Temps de restauration (pire cas - 2 Inc) :
- Full : ~2h50
- Application 2 Inc : ~10 min
- Binary logs : ~20 min
Total : ~3h20 ✅ RTO respecté
```

---

## Impact sur les performances en production

### Charge système pendant les backups

**Sauvegarde Full** :
```
CPU : 15-25% (compression, checksum)
I/O Read : 80-95% (lecture séquentielle de tous les fichiers)
I/O Write : Variable (dépend de la destination)
Réseau : 50-80% si backup distant
Mémoire : Stable (buffer pool reste en cache)

Recommandation : Planifier en heures creuses
```

**Sauvegarde Incrémentale** :
```
CPU : 10-15% (moins de données à compresser)
I/O Read : 5-15% (lecture des pages modifiées uniquement)
I/O Write : Faible
Réseau : 5-20% si backup distant
Mémoire : Stable

Recommandation : Possible en journée avec impact minimal
```

### Optimisations pour réduire l'impact

**Configuration MariaDB** :
```ini
# /etc/mysql/mariadb.conf.d/backup-tuning.cnf

[mariadb]
# Limiter l'impact I/O des backups
innodb_io_capacity = 2000
innodb_io_capacity_max = 4000

# Prioriser les transactions en cours
innodb_thread_concurrency = 0  # Auto

# Buffer pool suffisant pour absorber la charge
innodb_buffer_pool_size = 24G  # 60-70% RAM
```

**Options Mariabackup** :
```bash
# Throttling I/O pour éviter saturation
mariabackup --backup \
  --throttle=100  # Limite à 100 MB/s en lecture \
  --compress \
  --compress-threads=4 \
  --parallel=4  # Parallélisme des lectures
```

**Nice et ionice** :
```bash
# Baisser la priorité du processus de backup
nice -n 19 ionice -c2 -n7 mariabackup --backup ...

# -c2 = Best effort (classe I/O basse priorité)
# -n7 = Priorité la plus basse
```

---

## Rétention et purge automatique

### Politique de rétention typique

```
┌────────────────────────────────────────────────────┐
│            Politique de rétention                  │
├────────────────────────────────────────────────────┤
│                                                    │
│  Quotidien    : 7 derniers jours                   │
│  Hebdomadaire : 4 dernières semaines               │
│  Mensuel      : 12 derniers mois                   │
│  Annuel       : 7 dernières années (compliance)    │
│                                                    │
└────────────────────────────────────────────────────┘
```

### Script de purge automatique

```bash
#!/bin/bash
# backup_retention.sh

BACKUP_ROOT="/backups/mariadb"

# Suppression des backups quotidiens > 7 jours
find "$BACKUP_ROOT/daily" -type d -mtime +7 -exec rm -rf {} \;

# Suppression des backups hebdomadaires > 28 jours
find "$BACKUP_ROOT/weekly" -type d -mtime +28 -exec rm -rf {} \;

# Suppression des backups mensuels > 365 jours
find "$BACKUP_ROOT/monthly" -type d -mtime +365 -exec rm -rf {} \;

# Logs
echo "[$(date)] Backup retention policy applied"
```

💡 **Bonne pratique** : Conserver une copie archivée hors ligne (tape, glacier) pour conformité long terme.

---

## Cas d'usage par secteur d'activité

### E-commerce

**Contraintes** :
- RPO : < 1 heure (perte de commandes inacceptable)
- RTO : < 2 heures (chaque minute offline = revenus perdus)
- Pic d'activité : Black Friday, soldes, promotions

**Stratégie recommandée** :
```
- Full : Quotidien à 03:00
- Incrémentale : Toutes les heures
- Binary logs : Rotation 15 minutes
- Snapshot K8s : Toutes les 6h

Total : RPO effectif < 15 min, RTO < 1h
```

### Finance / Banque

**Contraintes** :
- RPO : 0 (aucune perte de transaction acceptable)
- RTO : < 1 heure
- Conformité : SOX, PCI-DSS, GDPR
- Audit trail complet

**Stratégie recommandée** :
```
- Full : Quotidien
- Incrémentale : Toutes les 30 minutes
- Binary logs : En continu, rétention 7 ans
- Réplication synchrone (Galera) + Backups
- Snapshots SAN toutes les heures

Total : RPO ≈ 0 (réplication), RTO < 30 min
```

### SaaS / Startup

**Contraintes** :
- Budget limité
- Simplicité opérationnelle
- Croissance rapide

**Stratégie recommandée** :
```
- Full : Hebdomadaire (dimanche)
- Différentielle : Quotidien
- Binary logs : 7 jours
- Cloud S3 avec lifecycle policies

Total : RPO < 24h, RTO < 4h, coût optimisé
```

### Média / Publishing

**Contraintes** :
- Gros volumes (vidéos, images)
- Taux de changement élevé
- Pic lors des publications

**Stratégie recommandée** :
```
- Full : Mensuel
- Incrémentale : Quotidien
- Stockage objet S3 (moteur S3 pour médias)
- Archivage Glacier pour contenu > 1 an

Total : Coût optimisé, restauration granulaire
```

---

## ✅ Points clés à retenir

- **Full backup** : Simple et fiable, mais coûteux en espace et temps — idéal pour petites bases
- **Incrémentale** : Minimise l'espace et la fenêtre, mais restauration complexe — pour grosses bases avec RPO serré
- **Différentielle** : Compromis optimal pour la plupart des cas — restauration simple (2 fichiers max)
- **Stratégies hybrides** sont la norme en production — combiner plusieurs approches
- **Dimensionnement** : Toujours calculer espace requis, fenêtre de backup et temps de restauration avant de choisir
- **RPO/RTO** dictent la stratégie — pas de solution universelle, tout dépend des SLA métier
- **Binary logs** sont essentiels pour PITR — activer systématiquement en production
- **Impact production** : Full en heures creuses, Incrémentale possible en journée avec throttling
- **Rétention** : GFS (quotidien/hebdo/mensuel) est un modèle éprouvé et conforme
- **Tests réguliers** : Valider temps de restauration réels, pas seulement théoriques

---

## 🔗 Ressources et références

### Documentation officielle MariaDB

- [📖 Mariabackup Overview - MariaDB KB](https://mariadb.com/kb/en/mariabackup-overview/)
- [📖 Full Backup and Restore - MariaDB KB](https://mariadb.com/kb/en/full-backup-and-restore-with-mariabackup/)
- [📖 Incremental Backup - MariaDB KB](https://mariadb.com/kb/en/incremental-backup-and-restore-with-mariabackup/)

### Articles techniques

- [Backup Strategy Best Practices - Percona](https://www.percona.com/blog/backup-strategy-best-practices/)
- [MySQL Backup Methods Comparison - Oracle](https://dev.mysql.com/doc/refman/8.0/en/backup-methods.html)
- [Database Backup and Recovery Strategies - AWS](https://docs.aws.amazon.com/prescriptive-guidance/latest/backup-recovery/backup-recovery.html)

### Outils de calcul

- [Backup Sizing Calculator](https://www.veritas.com/support/en_US/article.100040016) (Veritas)
- [Storage Calculator for Backup](https://www.cohesity.com/solutions/backup-and-recovery/backup-storage-calculator/)

---

## ➡️ Section suivante

**[12.2 - Sauvegarde logique](./02-sauvegarde-logique.md)** : Guide détaillé de mariadb-dump et mydumper/myloader avec options avancées, optimisations de performance et cas d'usage spécifiques.

---


⏭️ [Sauvegarde logique](/12-sauvegarde-restauration/02-sauvegarde-logique.md)
