🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.7 Virtual IP et keepalived

> **Niveau** : Expert  
> **Durée estimée** : 2-3 heures  
> **Prérequis** : Section 14.6 (Failover), connaissances réseau (IP, ARP, routing)

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- **Comprendre** le concept de Virtual IP et son rôle dans la haute disponibilité
- **Configurer** keepalived pour gérer le failover IP automatique
- **Implémenter** VRRP pour la redondance réseau
- **Intégrer** VIP avec MariaDB et MaxScale
- **Diagnostiquer** et résoudre les problèmes de VIP
- **Sécuriser** votre infrastructure VIP
- **Comparer** keepalived avec les alternatives cloud-native
- **Opérer** une infrastructure VIP en production

---

## Introduction

Le **Virtual IP (VIP)** est une adresse IP flottante qui peut être déplacée automatiquement d'un serveur à un autre lors d'un failover. C'est un composant essentiel de toute architecture haute disponibilité car il fournit un **point d'accès unique et stable** aux applications, indépendamment du serveur physique qui traite réellement les requêtes.

**Problème sans VIP** :
```
Application
    │
    └─── Se connecte à : master.db.example.com (10.0.1.10)
                         │
                         ▼
                    Master (10.0.1.10) ✖ CRASH
                         
Application
    │
    └─── Doit reconfigurer pour : replica.db.example.com (10.0.1.11)
         ❌ Nécessite redéploiement application
         ❌ Downtime pendant changement configuration
         ❌ DNS cache peut prendre minutes/heures
```

**Solution avec VIP** :
```
Application
    │
    └─── Se connecte TOUJOURS à : vip.db.example.com (10.0.1.100)
                                  │
                                  │ VIP flottante
                                  │
         ┌────────────────────────┼────────────────────────┐
         ▼                        ▼                        ▼
    Master (10.0.1.10)      Replica (10.0.1.11)     Replica (10.0.1.12)
         │                                              
         ✖ CRASH                                      
         │                                              
    Failover automatique (30s)                        
         │                                              
         └──► VIP migre vers Replica (10.0.1.11)
         
Application
    │
    └─── Continue vers : vip.db.example.com (10.0.1.100)
         ✅ Aucun changement configuration
         ✅ Transparence totale
         ✅ Downtime = 30 secondes
```

> 💡 **Principe Clé** : "L'application ne doit jamais connaître l'infrastructure physique. Elle doit uniquement connaître un endpoint logique (VIP)."

---

## 1. Concepts Fondamentaux

### 1.1 Virtual IP (VIP)

**Définition** : Adresse IP qui n'est pas liée de manière permanente à une interface réseau physique, mais peut être assignée dynamiquement à différents serveurs.

**Caractéristiques** :
```
┌─────────────────────────────────────────────────┐
│  VIP : 10.0.1.100                               │
│                                                 │
│  État Normal :                                  │
│  ├─ Serveur1 (10.0.1.10) : VIP active ✅        │
│  ├─ Serveur2 (10.0.1.11) : VIP inactive         │
│  └─ Serveur3 (10.0.1.12) : VIP inactive         │
│                                                 │
│  Après Failover :                               │
│  ├─ Serveur1 (10.0.1.10) : DOWN                 │
│  ├─ Serveur2 (10.0.1.11) : VIP active ✅        │
│  └─ Serveur3 (10.0.1.12) : VIP inactive         │
└─────────────────────────────────────────────────┘
```

**Types de VIP** :

1. **Service VIP** : Pour les services applicatifs (base de données, web, etc.)
2. **Management VIP** : Pour l'administration centralisée
3. **Cluster VIP** : Pour la communication inter-nœuds (Galera, etc.)

### 1.2 VRRP (Virtual Router Redundancy Protocol)

**VRRP** (RFC 3768) est le protocole standard qui permet à plusieurs routeurs/serveurs de partager une adresse IP virtuelle.

**Fonctionnement** :
```
VRRP Group (VRID = 51)
VIP : 10.0.1.100

Node1 (MASTER)          Node2 (BACKUP)         Node3 (BACKUP)
Priority: 100           Priority: 90           Priority: 80
    │                        │                      │
    ├──► Advertisement ──────┼──────────────────────┤
    │    (every 1 sec)       │                      │
    │                        │                      │
    │    "I'm MASTER"        │  "OK, you're MASTER" │
    │                        │                      │
    
Node1 CRASH
    ✖
                        Node2 détecte :
                        "No advertisement received"
                        "I have highest priority among BACKUP"
                        
                        Node2 → MASTER
                        Priority: 90
                            │
                            ├──► Advertisement ────────────────┤
                            │                                  │
                            │    "I'm MASTER now"              │
                            │                      "OK, you're MASTER"
```

**Paramètres VRRP** :

| Paramètre | Description | Valeur Typique |
|-----------|-------------|----------------|
| **VRID** | Virtual Router ID (1-255) | 51 |
| **Priority** | Priorité du nœud (1-255) | 100 (master), 90, 80 (backups) |
| **Advertisement Interval** | Fréquence heartbeat | 1 seconde |
| **Preemption** | Reprendre MASTER si priorité supérieure | Activé |
| **Authentication** | Sécurité VRRP | PASS (simple) ou AH (IPsec) |

### 1.3 Gratuitous ARP

**Problème** : Après migration VIP, le switch réseau pointe toujours l'ancienne MAC address dans sa table ARP.

```
État initial :
VIP 10.0.1.100 → MAC_ADDRESS_NODE1 (dans ARP cache switch)

Failover :
VIP migre vers Node2

État problématique :
Switch : VIP 10.0.1.100 → MAC_ADDRESS_NODE1 (obsolète !)
Réseau : Trafic envoyé vers Node1 (down) ❌
```

**Solution** : Gratuitous ARP (GARP)

```
Node2 devient MASTER :
1. Assigne VIP à son interface
2. Envoie Gratuitous ARP broadcast :
   "WHO-HAS 10.0.1.100? TELL 10.0.1.100"
   "10.0.1.100 is at MAC_ADDRESS_NODE2"
   
Switch met à jour sa table ARP :
VIP 10.0.1.100 → MAC_ADDRESS_NODE2 ✅

Réseau : Trafic envoyé vers Node2 ✅
```

**Commande manuelle** :
```bash
# Envoyer GARP
arping -c 3 -A -I eth0 10.0.1.100
# -c 3 : 3 paquets
# -A : Mode ARP announcement
# -I eth0 : Interface
```

---

## 2. keepalived : Configuration et Déploiement

### 2.1 Installation

#### **Ubuntu/Debian**
```bash
# Installation
apt-get update
apt-get install -y keepalived

# Vérification version
keepalived --version
# Keepalived v2.2.8

# Activer au boot
systemctl enable keepalived
```

#### **RHEL/CentOS**
```bash
# Installation
yum install -y keepalived

# Activer
systemctl enable keepalived
systemctl start keepalived
```

#### **Vérification réseau**
```bash
# Autoriser VRRP dans firewall
# VRRP utilise protocole IP 112 (pas TCP/UDP)

# iptables
iptables -A INPUT -p vrrp -j ACCEPT
iptables -A OUTPUT -p vrrp -j ACCEPT

# firewalld
firewall-cmd --add-rich-rule='rule protocol value="vrrp" accept' --permanent
firewall-cmd --reload

# ufw
# Pas de support natif VRRP, utiliser iptables directement
```

### 2.2 Configuration de Base (MariaDB Master-Slave)

#### **Architecture Cible**
```
┌──────────────────────────────────────────┐
│  Applications                            │
└────────────┬─────────────────────────────┘
             │
             │ Connexion à VIP : 10.0.1.100
             │
        ┌────┴────┐
        │   VIP   │ (flottante)
        └────┬────┘
             │
   ┌─────────┼─────────┐
   │                   │
┌──▼──────────┐  ┌─────▼─────────┐
│   Node1     │  │    Node2      │
│ 10.0.1.10   │  │  10.0.1.11    │
│             │  │               │
│ MariaDB     │──│─► MariaDB     │
│ MASTER      │  │   REPLICA     │
│             │  │               │
│ keepalived  │  │  keepalived   │
│ MASTER      │  │  BACKUP       │
│ Priority:100│  │  Priority:90  │
└─────────────┘  └───────────────┘
```

#### **Configuration Node1 (MASTER)**
```bash
# /etc/keepalived/keepalived.conf

global_defs {
    # Identification unique du serveur
    router_id MARIADB_HA_NODE1
    
    # Notifications email (optionnel)
    notification_email {
        dba-team@example.com
        ops-team@example.com
    }
    notification_email_from keepalived@node1.example.com
    smtp_server smtp.example.com
    smtp_connect_timeout 30
    
    # Logging
    enable_script_security
    script_user root
}

# Health check script
vrrp_script check_mariadb {
    script "/usr/local/bin/check_mariadb.sh"
    interval 2          # Vérifier toutes les 2 secondes
    timeout 3           # Timeout à 3 secondes
    weight -20          # Réduire priorité de 20 si échec
    fall 3              # 3 échecs consécutifs = DOWN
    rise 2              # 2 succès consécutifs = UP
}

# Instance VRRP
vrrp_instance MARIADB_VIP {
    state MASTER        # État initial (MASTER ou BACKUP)
    interface eth0      # Interface réseau
    virtual_router_id 51  # VRID (doit être identique sur tous les nœuds)
    priority 100        # Priorité (plus élevée = préféré)
    advert_int 1        # Interval heartbeat (secondes)
    
    # Authentification (sécurité VRRP)
    authentication {
        auth_type PASS
        auth_pass SecureVRRPPassword123
    }
    
    # VIP à gérer
    virtual_ipaddress {
        10.0.1.100/24 dev eth0 label eth0:vip
    }
    
    # Health checks
    track_script {
        check_mariadb
    }
    
    # Scripts de notification
    notify_master "/usr/local/bin/notify_master.sh"
    notify_backup "/usr/local/bin/notify_backup.sh"
    notify_fault "/usr/local/bin/notify_fault.sh"
    
    # Preemption (reprendre MASTER si priorité supérieure)
    preempt_delay 300   # Attendre 5 minutes avant preemption
}
```

#### **Configuration Node2 (BACKUP)**
```bash
# /etc/keepalived/keepalived.conf

global_defs {
    router_id MARIADB_HA_NODE2
    notification_email {
        dba-team@example.com
        ops-team@example.com
    }
    notification_email_from keepalived@node2.example.com
    smtp_server smtp.example.com
    smtp_connect_timeout 30
}

vrrp_script check_mariadb {
    script "/usr/local/bin/check_mariadb.sh"
    interval 2
    timeout 3
    weight -20
    fall 3
    rise 2
}

vrrp_instance MARIADB_VIP {
    state BACKUP        # ← BACKUP (différence principale)
    interface eth0
    virtual_router_id 51  # ← Identique à Node1
    priority 90         # ← Priorité inférieure à Node1
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass SecureVRRPPassword123  # ← Identique à Node1
    }
    
    virtual_ipaddress {
        10.0.1.100/24 dev eth0 label eth0:vip
    }
    
    track_script {
        check_mariadb
    }
    
    notify_master "/usr/local/bin/notify_master.sh"
    notify_backup "/usr/local/bin/notify_backup.sh"
    notify_fault "/usr/local/bin/notify_fault.sh"
    
    preempt_delay 300
}
```

### 2.3 Health Check Script

```bash
#!/bin/bash
# /usr/local/bin/check_mariadb.sh

# Configuration
MYSQL_HOST="localhost"
MYSQL_PORT="3306"
MYSQL_USER="monitor"
MYSQL_PASS="SecureMonitorPassword"
TIMEOUT=2

# Fonction de vérification
check_mysql_alive() {
    mysqladmin ping \
        -h "$MYSQL_HOST" \
        -P "$MYSQL_PORT" \
        -u "$MYSQL_USER" \
        -p"$MYSQL_PASS" \
        --connect_timeout="$TIMEOUT" \
        2>/dev/null
    
    return $?
}

check_mysql_writable() {
    mysql -h "$MYSQL_HOST" \
          -P "$MYSQL_PORT" \
          -u "$MYSQL_USER" \
          -p"$MYSQL_PASS" \
          --connect_timeout="$TIMEOUT" \
          -e "SELECT @@read_only" 2>/dev/null | grep -q "0"
    
    return $?
}

check_replication_lag() {
    # Vérifier si replica, et lag acceptable
    IS_SLAVE=$(mysql -h "$MYSQL_HOST" -u "$MYSQL_USER" -p"$MYSQL_PASS" \
                -N -e "SELECT COUNT(*) FROM information_schema.SLAVE_STATUS" 2>/dev/null)
    
    if [ "$IS_SLAVE" -eq 1 ]; then
        LAG=$(mysql -h "$MYSQL_HOST" -u "$MYSQL_USER" -p"$MYSQL_PASS" \
              -N -e "SELECT Seconds_Behind_Master FROM information_schema.SLAVE_STATUS" 2>/dev/null)
        
        # Lag > 60 secondes = KO
        if [ -n "$LAG" ] && [ "$LAG" != "NULL" ] && [ "$LAG" -gt 60 ]; then
            return 1
        fi
    fi
    
    return 0
}

# Exécution des checks
check_mysql_alive || exit 1
check_mysql_writable || exit 1
check_replication_lag || exit 1

exit 0
```

**Permissions** :
```bash
chmod +x /usr/local/bin/check_mariadb.sh
chown root:root /usr/local/bin/check_mariadb.sh
```

### 2.4 Scripts de Notification

#### **notify_master.sh**
```bash
#!/bin/bash
# /usr/local/bin/notify_master.sh
# Exécuté quand ce nœud devient MASTER

TYPE=$1      # "INSTANCE" ou "GROUP"
NAME=$2      # Nom de l'instance VRRP
STATE=$3     # "MASTER"
PRIORITY=$4  # Priorité

HOSTNAME=$(hostname)
VIP="10.0.1.100"

# Logging
logger -t keepalived "Transition to MASTER state on $HOSTNAME"

# Gratuitous ARP (forcer mise à jour ARP)
arping -c 3 -A -I eth0 $VIP

# Notification Slack
curl -X POST https://hooks.slack.com/services/YOUR/WEBHOOK/URL \
  -H 'Content-Type: application/json' \
  -d "{
    \"text\": \"🟢 VIP Failover\",
    \"attachments\": [{
      \"color\": \"good\",
      \"fields\": [
        {\"title\": \"Event\", \"value\": \"Became MASTER\", \"short\": true},
        {\"title\": \"Node\", \"value\": \"$HOSTNAME\", \"short\": true},
        {\"title\": \"VIP\", \"value\": \"$VIP\", \"short\": true},
        {\"title\": \"Time\", \"value\": \"$(date)\", \"short\": true}
      ]
    }]
  }"

# Métriques Prometheus (optionnel)
echo "keepalived_state{node=\"$HOSTNAME\"} 1" | \
  curl --data-binary @- http://pushgateway.example.com:9091/metrics/job/keepalived

# PagerDuty alert (optionnel)
curl -X POST https://api.pagerduty.com/incidents \
  -H 'Authorization: Token token=YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d "{
    \"incident\": {
      \"type\": \"incident\",
      \"title\": \"MariaDB VIP failover to $HOSTNAME\",
      \"service\": {\"id\": \"YOUR_SERVICE_ID\", \"type\": \"service_reference\"},
      \"urgency\": \"high\"
    }
  }"

exit 0
```

#### **notify_backup.sh**
```bash
#!/bin/bash
# /usr/local/bin/notify_backup.sh
# Exécuté quand ce nœud redevient BACKUP

HOSTNAME=$(hostname)

logger -t keepalived "Transition to BACKUP state on $HOSTNAME"

curl -X POST https://hooks.slack.com/services/YOUR/WEBHOOK/URL \
  -H 'Content-Type: application/json' \
  -d "{
    \"text\": \"🟡 VIP Status Change\",
    \"attachments\": [{
      \"color\": \"warning\",
      \"fields\": [
        {\"title\": \"Event\", \"value\": \"Became BACKUP\", \"short\": true},
        {\"title\": \"Node\", \"value\": \"$HOSTNAME\", \"short\": true}
      ]
    }]
  }"

exit 0
```

#### **notify_fault.sh**
```bash
#!/bin/bash
# /usr/local/bin/notify_fault.sh
# Exécuté en cas de problème détecté

HOSTNAME=$(hostname)

logger -t keepalived "FAULT state detected on $HOSTNAME"

curl -X POST https://hooks.slack.com/services/YOUR/WEBHOOK/URL \
  -H 'Content-Type: application/json' \
  -d "{
    \"text\": \"🔴 VIP FAULT\",
    \"attachments\": [{
      \"color\": \"danger\",
      \"fields\": [
        {\"title\": \"Event\", \"value\": \"FAULT detected\", \"short\": true},
        {\"title\": \"Node\", \"value\": \"$HOSTNAME\", \"short\": true},
        {\"title\": \"Action\", \"value\": \"Check MariaDB health immediately\", \"short\": false}
      ]
    }]
  }"

exit 0
```

**Permissions** :
```bash
chmod +x /usr/local/bin/notify_*.sh
chown root:root /usr/local/bin/notify_*.sh
```

### 2.5 Démarrage et Validation

```bash
# Démarrer keepalived sur les deux nœuds
systemctl start keepalived

# Vérifier statut
systemctl status keepalived

# Vérifier logs
tail -f /var/log/syslog | grep keepalived
# ou
journalctl -u keepalived -f

# Vérifier VIP assignée (sur MASTER uniquement)
ip addr show eth0
# eth0:vip: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP
#     inet 10.0.1.100/24 scope global secondary eth0:vip

# Tester connexion via VIP
mysql -h 10.0.1.100 -u root -p -e "SELECT @@hostname"
# Doit retourner le hostname du MASTER actuel

# Vérifier état VRRP
cat /proc/net/vrrp
# ou utiliser tcpdump pour voir advertisements
tcpdump -i eth0 -n vrrp
# 10.0.1.10 > 224.0.0.18: VRRPv2, Advertisement, vrid 51, prio 100
```

---

## 3. Configuration Avancée

### 3.1 Multiple VIPs

```bash
# keepalived.conf
vrrp_instance MARIADB_VIP {
    # ... configuration standard ...
    
    virtual_ipaddress {
        10.0.1.100/24 dev eth0 label eth0:vip1  # VIP applicative
        10.0.1.101/24 dev eth0 label eth0:vip2  # VIP admin
        10.0.1.102/24 dev eth0 label eth0:vip3  # VIP monitoring
    }
}
```

**Cas d'usage** :
- VIP1 : Applications production
- VIP2 : Outils d'administration (adminer, phpMyAdmin)
- VIP3 : Exporters monitoring (mysqld_exporter)

### 3.2 keepalived pour MaxScale

```bash
# /etc/keepalived/keepalived.conf
# Configuration pour MaxScale HA

vrrp_script check_maxscale {
    script "/usr/local/bin/check_maxscale.sh"
    interval 2
    timeout 3
    weight -20
    fall 3
    rise 2
}

vrrp_instance MAXSCALE_VIP {
    state MASTER
    interface eth0
    virtual_router_id 52  # Différent de MariaDB (51)
    priority 100
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass MaxScaleVRRPPass456
    }
    
    virtual_ipaddress {
        10.0.2.100/24 dev eth0 label eth0:maxscale_vip
    }
    
    track_script {
        check_maxscale
    }
}
```

```bash
#!/bin/bash
# /usr/local/bin/check_maxscale.sh

# Vérifier MaxScale opérationnel
maxctrl show maxscale &>/dev/null || exit 1

# Vérifier au moins un serveur disponible
AVAILABLE=$(maxctrl list servers --tsv | grep -c "Running")
[ "$AVAILABLE" -ge 1 ] || exit 1

exit 0
```

### 3.3 Unicast VRRP (sans multicast)

**Problème** : Certains environnements cloud (AWS, Azure) ne supportent pas multicast.

**Solution** : Unicast VRRP

```bash
# keepalived.conf
vrrp_instance MARIADB_VIP {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    
    # Mode unicast (au lieu de multicast)
    unicast_src_ip 10.0.1.10  # IP de ce nœud
    unicast_peer {
        10.0.1.11               # IP des autres nœuds
        10.0.1.12
    }
    
    # ... reste de la configuration ...
}
```

### 3.4 VRRP Sync Groups

**Cas d'usage** : Synchroniser plusieurs VIPs (failover groupé)

```bash
# keepalived.conf

# Groupe de synchronisation
vrrp_sync_group MARIADB_CLUSTER {
    group {
        MARIADB_VIP
        MAXSCALE_VIP
        ADMIN_VIP
    }
    
    notify_master "/usr/local/bin/sync_group_master.sh"
    notify_backup "/usr/local/bin/sync_group_backup.sh"
}

# Les 3 VIPs basculent ensemble
vrrp_instance MARIADB_VIP {
    # ... configuration ...
}

vrrp_instance MAXSCALE_VIP {
    # ... configuration ...
}

vrrp_instance ADMIN_VIP {
    # ... configuration ...
}
```

### 3.5 Virtual Routes

```bash
# Ajouter des routes virtuelles (failover routing)
vrrp_instance MARIADB_VIP {
    # ... configuration standard ...
    
    virtual_ipaddress {
        10.0.1.100/24 dev eth0
    }
    
    # Routes virtuelles (failover avec la VIP)
    virtual_routes {
        10.0.2.0/24 via 10.0.1.1 dev eth0
        192.168.1.0/24 via 10.0.1.1 dev eth0
    }
}
```

---

## 4. Intégration avec MariaDB/MaxScale

### 4.1 Architecture Complète HA

```
┌────────────────────────────────────────────────────┐
│                  Applications                      │
└──────────────────────┬─────────────────────────────┘
                       │
                       │ Connexion VIP MaxScale
                       │ 10.0.2.100:3306
                       │
          ┌────────────▼────────────┐
          │    VIP MaxScale         │
          │    (keepalived)         │
          └────────────┬────────────┘
                       │
          ┌────────────┼────────────┐
          │                         │
    ┌─────▼──────┐            ┌─────▼──────┐
    │ MaxScale 1 │            │ MaxScale 2 │
    │ 10.0.2.10  │            │ 10.0.2.11  │
    │ MASTER     │            │ BACKUP     │
    └─────┬──────┘            └─────┬──────┘
          │                         │
          └────────────┬────────────┘
                       │
                       │ Backend connections
                       │
          ┌────────────▼────────────┐
          │    VIP MariaDB          │
          │    (keepalived)         │
          └────────────┬────────────┘
                       │
          ┌────────────┼────────────┐
          │            │            │
    ┌─────▼──────┐ ┌───▼───────┐ ┌──▼─────────┐
    │ MariaDB 1  │ │ MariaDB 2 │ │ MariaDB 3  │
    │ 10.0.1.10  │ │ 10.0.1.11 │ │ 10.0.1.12  │
    │ MASTER     │ │ REPLICA   │ │ REPLICA    │
    └────────────┘ └───────────┘ └────────────┘
```

### 4.2 Configuration Multi-Tier VIP

```bash
# MaxScale tier
# /etc/keepalived/keepalived_maxscale.conf
vrrp_instance MAXSCALE_VIP {
    state MASTER
    interface eth0
    virtual_router_id 52
    priority 100
    advert_int 1
    
    virtual_ipaddress {
        10.0.2.100/24 dev eth0
    }
    
    track_script {
        check_maxscale
    }
}

# MariaDB tier
# /etc/keepalived/keepalived_mariadb.conf
vrrp_instance MARIADB_VIP {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    
    virtual_ipaddress {
        10.0.1.100/24 dev eth0
    }
    
    track_script {
        check_mariadb
    }
}
```

**Démarrage** :
```bash
# Utiliser deux instances keepalived (optionnel)
keepalived -f /etc/keepalived/keepalived_maxscale.conf -p /var/run/keepalived_maxscale.pid
keepalived -f /etc/keepalived/keepalived_mariadb.conf -p /var/run/keepalived_mariadb.pid

# Ou une seule instance avec les deux configurations
```

---

## 5. Troubleshooting et Diagnostics

### 5.1 Problèmes Courants

#### **Problème 1 : Split-Brain VIP (Deux nœuds MASTER)**

**Symptômes** :
```bash
# Sur Node1
ip addr show eth0 | grep 10.0.1.100
# inet 10.0.1.100/24 scope global secondary eth0:vip

# Sur Node2
ip addr show eth0 | grep 10.0.1.100
# inet 10.0.1.100/24 scope global secondary eth0:vip

# ❌ Les DEUX ont la VIP !
```

**Causes** :
- Firewall bloque VRRP (protocole 112)
- VRID différent entre nœuds
- Partition réseau

**Diagnostic** :
```bash
# Vérifier VRRP advertisements
tcpdump -i eth0 -n vrrp
# Rien affiché = VRRP bloqué

# Vérifier firewall
iptables -L | grep vrrp
# Doit montrer ACCEPT

# Vérifier configuration
grep virtual_router_id /etc/keepalived/keepalived.conf
# Doit être identique sur tous les nœuds
```

**Solution** :
```bash
# Ouvrir firewall
iptables -I INPUT -p vrrp -j ACCEPT
iptables -I OUTPUT -p vrrp -j ACCEPT

# Ou désactiver firewall temporairement pour test
systemctl stop firewalld

# Redémarrer keepalived
systemctl restart keepalived
```

#### **Problème 2 : VIP Non Accessible**

**Symptômes** :
```bash
ping 10.0.1.100
# Request timeout
```

**Diagnostic** :
```bash
# VIP assignée ?
ip addr show | grep 10.0.1.100
# Si vide = VIP pas assignée

# Keepalived en cours ?
systemctl status keepalived

# Logs
journalctl -u keepalived -n 50
```

**Causes fréquentes** :
```
1. Health check échoue
   → Check script retourne erreur
   → Vérifier manuellement : /usr/local/bin/check_mariadb.sh

2. Priorité insuffisante
   → Tous les nœuds BACKUP, aucun MASTER
   → Augmenter priorité d'un nœud

3. Configuration invalide
   → Erreur syntaxe keepalived.conf
   → Vérifier logs : grep ERROR /var/log/syslog
```

#### **Problème 3 : Flapping (Basculements Répétés)**

**Symptômes** :
```bash
# Logs montrent basculements fréquents
tail -f /var/log/syslog | grep keepalived
# Mar 15 14:32:10 node1 Keepalived_vrrp: VRRP_Instance(MARIADB_VIP) Entering MASTER STATE
# Mar 15 14:32:15 node1 Keepalived_vrrp: VRRP_Instance(MARIADB_VIP) Entering BACKUP STATE
# Mar 15 14:32:20 node1 Keepalived_vrrp: VRRP_Instance(MARIADB_VIP) Entering MASTER STATE
# ... répétition ...
```

**Causes** :
- Health check trop agressif (timeout court)
- Charge serveur élevée (slow responses)
- Problèmes réseau intermittents

**Solution** :
```bash
# Ajuster health check
vrrp_script check_mariadb {
    script "/usr/local/bin/check_mariadb.sh"
    interval 5      # ← Augmenter (2 → 5s)
    timeout 10      # ← Augmenter (3 → 10s)
    weight -20
    fall 5          # ← Augmenter (3 → 5 échecs)
    rise 3          # ← Augmenter (2 → 3 succès)
}

# Ajouter délai preemption
vrrp_instance MARIADB_VIP {
    # ...
    preempt_delay 600  # ← 10 minutes avant reprendre MASTER
}
```

### 5.2 Commandes de Diagnostic

```bash
# État keepalived détaillé
kill -USR1 $(pidof keepalived)
# Écrit état dans /tmp/keepalived.data

cat /tmp/keepalived.data

# Statistiques VRRP
kill -USR2 $(pidof keepalived)
# Écrit stats dans /tmp/keepalived.stats

# Forcer basculement (test)
# Sur MASTER actuel, arrêter keepalived
systemctl stop keepalived
# Observer BACKUP devenir MASTER

# Tester health check manuellement
/usr/local/bin/check_mariadb.sh
echo $?
# 0 = OK, 1 = FAIL

# Capture trafic VRRP
tcpdump -i eth0 -vvv -n vrrp
# Analyse advertisements

# Vérifier table ARP
arp -n | grep 10.0.1.100
# Voir quelle MAC possède VIP
```

### 5.3 Monitoring keepalived

```bash
#!/bin/bash
# /usr/local/bin/monitor_keepalived.sh

# Vérifier processus keepalived
if ! pgrep -x keepalived > /dev/null; then
    echo "CRITICAL: keepalived not running"
    systemctl start keepalived
    exit 2
fi

# Vérifier état VRRP
VIP="10.0.1.100"
HAS_VIP=$(ip addr show | grep -c "$VIP")

if [ "$HAS_VIP" -eq 1 ]; then
    # Ce nœud a la VIP (MASTER)
    STATE="MASTER"
    
    # Vérifier MariaDB accessible sur VIP
    mysql -h "$VIP" -u monitor -pSecurePassword -e "SELECT 1" &>/dev/null
    if [ $? -ne 0 ]; then
        echo "CRITICAL: VIP assigned but MariaDB not accessible"
        exit 2
    fi
else
    # Ce nœud n'a pas la VIP (BACKUP)
    STATE="BACKUP"
fi

echo "OK: keepalived $STATE state"
exit 0
```

**Intégration Nagios/Icinga** :
```bash
# /etc/nagios/nrpe.d/keepalived.cfg
command[check_keepalived]=/usr/local/bin/monitor_keepalived.sh
```

---

## 6. Sécurité

### 6.1 Authentification VRRP

**Simple Password (PASS)** :
```bash
authentication {
    auth_type PASS
    auth_pass SecurePassword123  # Max 8 caractères !
}
```
⚠️ **Limitation** : Mot de passe envoyé en clair, limité à 8 caractères.

**IPsec AH (Authentication Header)** :
```bash
authentication {
    auth_type AH
    auth_pass SecurePassword12345678  # Jusqu'à 8 chars utilisés
}
```
✅ **Plus sécurisé** : Hash HMAC-MD5, intégrité garantie.

### 6.2 Restriction Réseau

```bash
# Limiter VRRP à subnet local uniquement
iptables -A INPUT -p vrrp -s 10.0.1.0/24 -j ACCEPT
iptables -A INPUT -p vrrp -j DROP

# Ou avec firewalld
firewall-cmd --permanent --zone=internal --add-rich-rule='
  rule family="ipv4" source address="10.0.1.0/24" protocol value="vrrp" accept'
firewall-cmd --reload
```

### 6.3 Prévention Split-Brain

```bash
# Utiliser health check robuste
vrrp_script check_mariadb {
    script "/usr/local/bin/check_mariadb_robust.sh"
    interval 2
    timeout 5
    weight -50  # ← Pénalité forte en cas d'échec
    fall 3
    rise 3
}
```

```bash
#!/bin/bash
# /usr/local/bin/check_mariadb_robust.sh

# Check 1: MySQL alive
mysqladmin ping -h localhost &>/dev/null || exit 1

# Check 2: Not read-only (master check)
READ_ONLY=$(mysql -N -e "SELECT @@read_only" 2>/dev/null)
[ "$READ_ONLY" = "0" ] || exit 1

# Check 3: No replication lag (if replica)
SLAVE_STATUS=$(mysql -e "SHOW SLAVE STATUS\G" 2>/dev/null)
if [ -n "$SLAVE_STATUS" ]; then
    LAG=$(echo "$SLAVE_STATUS" | grep "Seconds_Behind_Master:" | awk '{print $2}')
    if [ "$LAG" = "NULL" ] || [ "$LAG" -gt 60 ]; then
        exit 1
    fi
fi

# Check 4: Quorum check (si Galera)
if mysql -e "SHOW STATUS LIKE 'wsrep_cluster_status'" 2>/dev/null | grep -q "Primary"; then
    # Galera - vérifier quorum
    CLUSTER_SIZE=$(mysql -N -e "SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME='wsrep_cluster_size'" 2>/dev/null)
    [ "$CLUSTER_SIZE" -ge 2 ] || exit 1
fi

exit 0
```

---

## 7. Alternatives et Solutions Cloud

### 7.1 AWS : Elastic IP

```bash
# AWS : Utiliser Elastic IP au lieu de keepalived

# Script de failover (appelé par mariadbmon ou Orchestrator)
#!/bin/bash
# /usr/local/bin/aws_eip_failover.sh

EIP_ALLOCATION_ID="eipalloc-12345678"
NEW_INSTANCE_ID="$1"  # Instance ID du nouveau master

# Réassocier Elastic IP
aws ec2 associate-address \
    --allocation-id "$EIP_ALLOCATION_ID" \
    --instance-id "$NEW_INSTANCE_ID" \
    --allow-reassociation

# Vérifier association
aws ec2 describe-addresses \
    --allocation-ids "$EIP_ALLOCATION_ID" \
    --query "Addresses[0].InstanceId" \
    --output text
```

**Avantages** :
- ✅ Natif AWS
- ✅ Pas de VRRP (fonctionne dans VPC)
- ✅ API simple

**Inconvénients** :
- ❌ Latence failover (~30-60s)
- ❌ Lock-in AWS
- ❌ Coût Elastic IP

### 7.2 Azure : Load Balancer

```bash
# Azure : Utiliser Load Balancer avec health probes

# ARM template (extrait)
{
  "type": "Microsoft.Network/loadBalancers",
  "name": "mariadb-lb",
  "properties": {
    "frontendIPConfigurations": [{
      "name": "mariadb-frontend",
      "properties": {
        "privateIPAddress": "10.0.1.100",
        "privateIPAllocationMethod": "Static"
      }
    }],
    "backendAddressPools": [{
      "name": "mariadb-backend"
    }],
    "loadBalancingRules": [{
      "name": "mariadb-rule",
      "properties": {
        "frontendPort": 3306,
        "backendPort": 3306,
        "protocol": "Tcp",
        "probe": {
          "id": "[concat(resourceId('Microsoft.Network/loadBalancers', 'mariadb-lb'), '/probes/mariadb-probe')]"
        }
      }
    }],
    "probes": [{
      "name": "mariadb-probe",
      "properties": {
        "protocol": "Tcp",
        "port": 3306,
        "intervalInSeconds": 5,
        "numberOfProbes": 2
      }
    }]
  }
}
```

### 7.3 GCP : Internal Load Balancer

```bash
# GCP : Internal TCP/UDP Load Balancer

# Créer health check
gcloud compute health-checks create tcp mariadb-health \
    --port=3306 \
    --check-interval=5s \
    --timeout=3s \
    --unhealthy-threshold=2 \
    --healthy-threshold=2

# Créer backend service
gcloud compute backend-services create mariadb-backend \
    --load-balancing-scheme=internal \
    --protocol=tcp \
    --health-checks=mariadb-health \
    --region=europe-west1

# Créer forwarding rule (VIP)
gcloud compute forwarding-rules create mariadb-lb \
    --load-balancing-scheme=internal \
    --address=10.0.1.100 \
    --ports=3306 \
    --backend-service=mariadb-backend \
    --region=europe-west1
```

### 7.4 Kubernetes : Service ClusterIP

```yaml
# Kubernetes : Service avec selector sur master
apiVersion: v1
kind: Service
metadata:
  name: mariadb-master
spec:
  type: ClusterIP
  clusterIP: 10.96.0.100  # VIP statique
  selector:
    app: mariadb
    role: master  # Selector sur le pod master actuel
  ports:
  - port: 3306
    targetPort: 3306
```

**Failover géré par** :
- Operator (mariadb-operator)
- StatefulSet avec readiness probes
- Automatic pod rescheduling

### 7.5 Comparaison

| Solution | On-Premise | AWS | Azure | GCP | K8s |
|----------|------------|-----|-------|-----|-----|
| **keepalived** | ✅ Optimal | ⚠️ Pas de multicast | ⚠️ Pas de multicast | ⚠️ Pas de multicast | ❌ Non |
| **Elastic IP** | ❌ N/A | ✅ Natif | ❌ N/A | ❌ N/A | ❌ N/A |
| **Load Balancer** | ⚠️ HAProxy | ✅ NLB | ✅ Azure LB | ✅ ILB | ✅ Service |
| **Corosync/Pacemaker** | ✅ Enterprise | ⚠️ Complexe | ⚠️ Complexe | ⚠️ Complexe | ❌ Non |

---

## ✅ Points Clés à Retenir

- **VIP** : Point d'accès unique, abstraction de l'infrastructure physique
- **VRRP** : Protocole standard (RFC 3768) pour redondance IP
- **keepalived** : Solution open source mature pour VRRP
- **Gratuitous ARP** : Essentiel pour mise à jour rapide des tables ARP
- **Health checks** : Robustes et multi-critères (alive, writable, lag)
- **Unicast VRRP** : Nécessaire dans environnements cloud (pas de multicast)
- **Monitoring** : Détecter split-brain, flapping, failovers
- **Cloud alternatives** : Elastic IP (AWS), Load Balancers (Azure/GCP)
- **Sécurité** : Authentification VRRP, restriction réseau
- **Testing** : Valider failover régulièrement (chaos engineering)

---

## 🔗 Ressources et Références

### Documentation Officielle
- [📖 keepalived Documentation](https://keepalived.readthedocs.io/)
- [📖 VRRP RFC 3768](https://tools.ietf.org/html/rfc3768)
- [📖 keepalived Configuration Guide](https://www.keepalived.org/doc/)

### Guides et Tutoriels
- **"High Availability with keepalived"** - DigitalOcean Tutorial
- **"VRRP Deep Dive"** - Cisco Networking Academy
- **"keepalived Best Practices"** - Red Hat Documentation

### Alternatives
- [Corosync/Pacemaker](https://clusterlabs.org/)
- [HAProxy](https://www.haproxy.org/)
- [Cloud Load Balancers Documentation](https://cloud.google.com/load-balancing)

---

## ➡️ Section Suivante

**[14.8 Stratégies de récupération après incident](/14-haute-disponibilite/08-strategies-recuperation.md)**

La section suivante abordera les stratégies complètes de disaster recovery : RPO/RTO planning, procédures de restauration, tests réguliers, et documentation des runbooks.

---

**Un VIP bien configuré avec keepalived est la fondation invisible mais essentielle de toute architecture haute disponibilité. La simplicité de configuration cache une robustesse éprouvée en production.**

⏭️ [Stratégies de récupération après incident](/14-haute-disponibilite/08-strategies-recuperation.md)
