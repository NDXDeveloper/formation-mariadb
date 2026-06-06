🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.8 — Audit et logging

L'audit est la **couche de traçabilité** du modèle de défense en profondeur (10.1). Là où l'authentification et les privilèges *contrôlent* l'accès, et où le chiffrement *protège* les échanges, l'audit *enregistre* ce qui se passe : **qui** a fait **quoi**, **quand** et **depuis où**. Il ne bloque rien, mais il crée une trace exploitable a posteriori. Cette section présente les enjeux de l'audit, sa distinction avec la journalisation ordinaire, et le mécanisme dédié de MariaDB ; les sous-sections détaillent le *Server Audit Plugin* (10.8.1), l'audit des connexions et des requêtes (10.8.2) et les améliorations récentes de la journalisation (10.8.3).

---

## Pourquoi auditer ?

Un journal d'audit répond à trois besoins complémentaires :

- **Surveillance de sécurité** : repérer des activités suspectes — tentatives de connexion répétées, accès inhabituels, modifications de privilèges — éventuellement en temps quasi réel.
- **Investigation (*forensics*)** : après un incident, reconstituer la chronologie des actions pour comprendre ce qui s'est passé et en mesurer la portée.
- **Conformité** : de nombreuses réglementations (RGPD, HIPAA, PCI-DSS, SOX…) exigent ou recommandent la **traçabilité des accès** aux données sensibles.

Le point essentiel est la distinction avec les couches précédentes : l'authentification et l'autorisation **empêchent** des actions, tandis que l'audit les **consigne**. L'audit ne protège pas en lui-même — il rend les actions **imputables**, ce qui dissuade, documente et permet de réagir.

---

## Audit ou journalisation ?

MariaDB produit déjà plusieurs **journaux opérationnels**, traités au chapitre 11 : le journal d'erreurs, le journal des requêtes générales, le journal des requêtes lentes et le journal binaire (*binary log*). Ces journaux servent au **diagnostic** et à l'exploitation, pas à la sécurité.

L'audit est différent par sa **finalité** : il est conçu pour enregistrer des **événements de sécurité** (connexions, requêtes, accès aux tables) de façon **structurée et filtrable**, en ne retenant que ce qui est pertinent. Le journal des requêtes générales pourrait en théorie capturer les requêtes, mais il n'est pas adapté à l'audit : pas de filtrage par utilisateur ou par type d'événement, coût de performance élevé, et absence d'orientation sécurité. Pour auditer, MariaDB dispose donc d'un outil spécifique : le **MariaDB Audit Plugin**.

---

## Le MariaDB Audit Plugin

Le *Server Audit Plugin* (bibliothèque `server_audit`) est le mécanisme d'audit de MariaDB. C'est un **plugin serveur** qui consigne les événements de sécurité dans un fichier dédié ou dans le **syslog** du système. Ses caractéristiques principales :

- il enregistre des événements de type **connexion**, **requête** et **accès aux tables** ;
- sa sortie peut être un **fichier** (avec rotation) ou **syslog** ;
- il se **filtre** par type d'événement et par utilisateur (listes d'inclusion/exclusion) ;
- il se configure via une famille de variables système préfixées `server_audit_`.

Comme la plupart des plugins, il est **fourni avec MariaDB mais pas activé par défaut** : on l'installe et on le configure (détaillé en 10.8.1). On vérifie sa présence et son état comme tout plugin :

```sql
SHOW PLUGINS;                          -- le plugin SERVER_AUDIT doit être ACTIVE
SHOW VARIABLES LIKE 'server_audit%';   -- ses paramètres de configuration
```

---

## Les types d'événements

L'audit MariaDB s'articule autour de quelques grandes catégories d'événements, présentées en détail en 10.8.2 :

- **CONNECT** — connexions, déconnexions et **tentatives de connexion échouées** (un signal précieux pour détecter des attaques) ;
- **QUERY** — requêtes exécutées, avec des sous-catégories selon la nature (DDL, DML, DCL) ;
- **TABLE** — accès au niveau des **tables** concernées par les requêtes.

Le choix des événements à journaliser est un compromis : auditer trop large alourdit le système et noie l'information ; auditer trop étroit laisse passer des actions importantes.

---

## Considérations importantes

Mettre en place un audit utile suppose d'anticiper plusieurs points :

- **Protéger les journaux d'audit.** Un audit n'a de valeur que si ses traces sont fiables. Un attaquant capable de **modifier ou effacer** les journaux d'audit en annule l'intérêt. On les stocke donc de façon restreinte, et l'on envisage de les **exporter** vers un système centralisé (syslog distant, SIEM) hors de portée d'une compromission du serveur.
- **Maîtriser le coût de performance.** Auditer chaque requête a un coût ; on **filtre** les événements et les utilisateurs pertinents, et la **journalisation bufferisée** (10.8.3) aide à réduire l'impact.
- **Conserver et exploiter.** Des journaux d'audit non relus ne servent à rien : il faut définir une **rétention** et, idéalement, un mécanisme d'**alerte** sur les événements critiques.
- **Tenir compte de la confidentialité.** Les journaux peuvent contenir le **texte des requêtes** — donc potentiellement des données sensibles. Ils méritent le même soin que les données qu'ils décrivent.

---

## Au programme

Les sous-sections suivantes approfondissent l'audit :

- **10.8.1 — Server Audit Plugin** : installation, activation et configuration du plugin (variables `server_audit_*`, sortie, rotation, filtrage).
- **10.8.2 — Audit de connexions et requêtes** : les types d'événements en détail et leur exploitation.
- **10.8.3 — Logging bufferisé, HOST:PORT et tls_version** 🆕 : les améliorations récentes — buffer d'écriture (`server_audit_file_buffer_size`), journalisation de l'hôte **et du port** des connexions, et ajout de la version TLS.

---

## À retenir

L'**audit** est la couche de **traçabilité** de la sécurité : il consigne qui fait quoi, quand et depuis où, au service de la **surveillance**, de l'**investigation** et de la **conformité**. Il se distingue des **journaux opérationnels** (erreurs, requêtes, binaire — chapitre 11) par sa finalité sécurité, sa structure et son filtrage. MariaDB l'assure via le **Server Audit Plugin** (`server_audit`), qui enregistre des événements **CONNECT**, **QUERY** et **TABLE** vers un fichier ou syslog, configurables par les variables `server_audit_*`. Un audit efficace suppose de **protéger les journaux** (idéalement hors du serveur), de **maîtriser leur coût**, de définir une **rétention/relecture**, et de tenir compte de la **confidentialité** de leur contenu.

---

> 🔔 **Note de version.** Le *Server Audit Plugin* et la famille de variables `server_audit_*` sont stables dans **MariaDB 12.3 LTS**. La série 12.x apporte des améliorations à la journalisation d'audit — buffer d'écriture, journalisation `HOST:PORT` et champ de version TLS — détaillées en 10.8.3. Les journaux opérationnels (erreurs, requêtes générales, requêtes lentes, journal binaire) relèvent du chapitre 11.

⏭️ [Server Audit Plugin](/10-securite-gestion-utilisateurs/08.1-server-audit-plugin.md)
