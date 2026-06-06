🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.6 Automatisation des sauvegardes

> **Chapitre 12 — Sauvegarde et Restauration** · MariaDB 12.3 LTS

---

## Introduction

Une stratégie de sauvegarde ne vaut que si elle s'exécute **régulièrement et sans faille**. Une sauvegarde manuelle est oubliée un jour de surcharge, lancée avec les mauvaises options, ou interrompue sans que personne ne s'en aperçoive. L'automatisation est donc moins une commodité qu'une **condition de fiabilité** : elle garantit que les sauvegardes ont lieu à heure fixe, de manière identique, et qu'une défaillance est détectée.

Automatiser ne se résume pas à inscrire une commande dans le planificateur. Un système de sauvegarde digne de ce nom combine plusieurs composants : une **planification**, un **script** robuste, une gestion **sécurisée des identifiants**, une **rotation** des anciennes sauvegardes, et — point trop souvent négligé — une **supervision** capable de signaler un échec. Cette section passe ces éléments en revue.

---

## La planification : `cron` et *systemd timers*

### `cron`

Le planificateur historique d'Unix reste le plus répandu. Une ligne dans la *crontab* suffit à déclencher un script à intervalle régulier :

```cron
# Sauvegarde complète chaque jour à 2 h du matin
0 2 * * * /usr/local/bin/sauvegarde-mariadb.sh >> /var/log/sauvegarde-mariadb.log 2>&1
```

La redirection vers un fichier de log permet de conserver une trace des exécutions.

### *systemd timers*

Sur les systèmes modernes, les *timers* de systemd offrent une alternative plus riche : journalisation intégrée (via `journald`), gestion des dépendances, planification souple (`OnCalendar`), et surtout la prise en charge des exécutions **manquées** (avec `Persistent=true`, une sauvegarde non exécutée — machine éteinte — est rattrapée au démarrage). Ils reposent sur un couple de fichiers, un service et un *timer* :

```ini
# /etc/systemd/system/sauvegarde-mariadb.service
[Unit]
Description=Sauvegarde MariaDB

[Service]
Type=oneshot
ExecStart=/usr/local/bin/sauvegarde-mariadb.sh
```

```ini
# /etc/systemd/system/sauvegarde-mariadb.timer
[Unit]
Description=Planifie la sauvegarde MariaDB quotidienne

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

On active ensuite le *timer* avec `systemctl enable --now sauvegarde-mariadb.timer`.

### Pourquoi pas l'Event Scheduler de MariaDB ?

On pourrait penser à l'Event Scheduler de MariaDB pour planifier les sauvegardes. Or les **events s'exécutent à l'intérieur du serveur** et ne peuvent pas lancer de commandes système : ils sont donc inadaptés pour piloter `mariadb-dump` ou `mariabackup`, qui sont des outils **externes** agissant sur le système de fichiers. La planification des sauvegardes relève bien de `cron` ou de systemd.

---

## Le script de sauvegarde

Le cœur de l'automatisation est un script qui enveloppe l'outil de sauvegarde avec les bonnes options, **gère les erreurs** et **journalise** son déroulement. En voici un squelette commenté, pour une sauvegarde physique :

```bash
#!/bin/bash
set -euo pipefail   # arrêt immédiat en cas d'erreur

CONF=/etc/mysql/backup.cnf
DEST=/sauvegardes/full-$(date +%F)
RETENTION=14        # jours

# 1. Sauvegarde puis préparation
mariabackup --defaults-extra-file="$CONF" --backup --target-dir="$DEST"
mariabackup --prepare --target-dir="$DEST"

# 2. Rotation : supprimer les sauvegardes de plus de RETENTION jours
find /sauvegardes -maxdepth 1 -name 'full-*' -type d \
     -mtime +"$RETENTION" -exec rm -rf {} +

# 3. (copie hors site, vérification, notification… cf. plus bas)
```

La directive `set -euo pipefail` est essentielle : elle interrompt le script à la **moindre erreur** (plutôt que de poursuivre en silence après une sauvegarde ratée). Le répertoire cible est **daté**, conformément aux bonnes pratiques vues en [12.3.1](03.1-full-backup.md). Pour une stratégie mêlant sauvegardes complètes et incrémentales ([12.1](01-strategies-sauvegarde.md)), on prévoit généralement deux scripts (ou un script paramétré) planifiés à des fréquences différentes — par exemple une complète hebdomadaire et des incréments quotidiens.

---

## Gérer les identifiants en sécurité

C'est un point critique trop souvent bâclé. Il ne faut **jamais** :

- passer le mot de passe **sur la ligne de commande** : il serait visible dans la liste des processus (`ps`) ;
- **coder en dur** les identifiants dans le script.

La bonne méthode consiste à placer les identifiants dans un **fichier d'options** dédié, protégé par des permissions strictes :

```ini
# /etc/mysql/backup.cnf  — chmod 600, propriété de l'utilisateur de sauvegarde
[client]
user=mariabackup
password=mot_de_passe

[mariabackup]
user=mariabackup
password=mot_de_passe
```

On y fait référence avec l'option `--defaults-extra-file` (comme dans le script ci-dessus). Le fichier doit impérativement être en **`chmod 600`** et appartenir au seul utilisateur habilité. La même approche vaut pour `mariadb-dump`, qui lit notamment les sections `[client]` et `[mariadb-dump]`.

---

## La rotation et la rétention

Sans rotation, les sauvegardes s'accumulent jusqu'à saturer le disque. Le script doit donc appliquer la **politique de rétention** définie en [section 12.1](01-strategies-sauvegarde.md) — par exemple un schéma **GFS** (quotidiennes conservées quelques jours, hebdomadaires quelques semaines, mensuelles plusieurs mois). La forme la plus simple repose sur l'ancienneté des fichiers :

```bash
# Supprimer les sauvegardes complètes de plus de 14 jours
find /sauvegardes -maxdepth 1 -name 'full-*' -mtime +14 -exec rm -rf {} +
```

> ⚠️ **Intégrité de la chaîne incrémentale.** Si vous utilisez des sauvegardes incrémentales ([12.3.2](03.2-incremental-backup.md)), ne supprimez **jamais** une sauvegarde complète dont des incréments dépendent encore : vous rendriez toute la chaîne inexploitable. La rétention doit raisonner par **cycle complet** (base + ses incréments), pas fichier par fichier.

Enfin, la rotation locale ne dispense pas de la **copie hors site** : conformément à la règle 3-2-1 ([12.1](01-strategies-sauvegarde.md)), une copie au moins doit résider ailleurs que sur le serveur — sujet développé pour le cloud en [section 12.8](08-sauvegarde-cloud-native.md).

---

## La supervision : savoir si une sauvegarde a échoué

C'est le maillon le plus souvent oublié, et pourtant le plus important : **une sauvegarde qui échoue en silence est pire que pas de sauvegarde du tout**, car elle entretient un faux sentiment de sécurité. Une automatisation sérieuse doit donc **alerter en cas d'échec** et **vérifier** que la sauvegarde s'est bien déroulée.

Plusieurs contrôles complémentaires :

- **Vérifier le code de retour** de chaque commande (ce que fait `set -e`), et **notifier** en cas d'échec (courriel, ou intégration à un système de supervision — Prometheus/Grafana, voir chapitre [16.9](../16-devops-automatisation/09-monitoring-prometheus-grafana.md)).
- **Contrôler le résultat** : présence des fichiers attendus (`mariadb_backup_checkpoints`, `mariadb_backup_info`), taille plausible de la sauvegarde.
- **Surveiller l'âge** de la dernière sauvegarde réussie : une alerte doit se déclencher si aucune sauvegarde récente n'existe.
- **Journaliser** chaque exécution pour pouvoir diagnostiquer après coup.

---

## Outils et solutions existants

Plutôt que de tout réécrire, on peut s'appuyer sur des solutions éprouvées. Des **scripts d'enrobage** open source pour `mariabackup` (gérant complète + incrémentale, compression, rotation) sont largement diffusés et constituent une bonne base. Des **outils dédiés** de gestion de sauvegarde existent également. Dans un contexte conteneurisé, l'**opérateur Kubernetes** de MariaDB (`mariadb-operator`) propose des **sauvegardes planifiées** déclaratives (voir chapitre [16.5](../16-devops-automatisation/05-mariadb-operator.md)), et les approches **cloud-native** sont traitées en [section 12.8](08-sauvegarde-cloud-native.md). Pour les environnements complexes, il est souvent plus sûr de réutiliser ces briques que de maintenir un script maison.

---

## Bonnes pratiques

- **Automatisez intégralement** : aucune étape manuelle, aucune dépendance à la mémoire d'un opérateur.
- **Planifiez** avec `cron` ou, de préférence, un *systemd timer* (`Persistent=true` pour rattraper les exécutions manquées).
- **Sécurisez les identifiants** dans un fichier d'options en `chmod 600`, jamais sur la ligne de commande.
- **Gérez la rotation** selon votre politique de rétention, en préservant l'intégrité des chaînes incrémentales.
- **Conservez une copie hors site** (règle 3-2-1).
- **Supervisez et alertez** : un échec silencieux doit être impossible.
- **Testez les sauvegardes automatisées** en les restaurant régulièrement — c'est l'objet de la [section 12.7](07-tests-restauration-pra.md).

---

## À retenir

- L'automatisation est une **condition de fiabilité** : elle garantit des sauvegardes régulières, identiques et surveillées.
- La **planification** passe par `cron` ou les *systemd timers* (plus riches : journalisation, rattrapage des exécutions manquées) ; l'**Event Scheduler** de MariaDB ne convient pas, car il ne lance pas de commandes système.
- Le **script** doit gérer les erreurs (`set -euo pipefail`), journaliser, et nommer les sauvegardes par date.
- Les **identifiants** se placent dans un **fichier d'options** (`chmod 600`) référencé par `--defaults-extra-file`, jamais en clair sur la ligne de commande.
- La **rotation** applique la politique de rétention sans casser les chaînes incrémentales, et n'exclut pas la **copie hors site**.
- La **supervision** est indispensable : une sauvegarde qui échoue en silence est pire que pas de sauvegarde.

---

⏮️ [12.5.2 — Point-in-time recovery (PITR)](05.2-pitr.md) · ⏭️ [12.7 — Tests de restauration et plan de reprise (PRA)](07-tests-restauration-pra.md)

⏭️ [Tests de restauration et plan de reprise (PRA)](/12-sauvegarde-restauration/07-tests-restauration-pra.md)
