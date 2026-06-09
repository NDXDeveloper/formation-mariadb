🔝 Retour au [Sommaire](/SOMMAIRE.md)

# H.1 — Documentation officielle

La documentation officielle est la **source faisant autorité** pour la syntaxe, les variables, les comportements et leurs différences d'une version à l'autre. Un changement récent est à connaître : jusqu'en juin 2025, la documentation se trouvait dans le MariaDB *Knowledge Base* (KB), avant d'être migrée, à de rares exceptions près, vers une nouvelle plateforme. Les anciens liens vers `mariadb.com/kb` peuvent donc être obsolètes ou archivés — préférez le nouveau site.

## La documentation de référence

- **[MariaDB Documentation](https://mariadb.com/docs/)** — le point d'entrée principal, qui regroupe la documentation, les notes de version et les ressources d'apprentissage de l'ensemble des produits MariaDB.
- **[MariaDB Server](https://mariadb.com/docs/server/)** — la documentation du serveur proprement dit, la plus utile pour cette formation : SQL, moteurs de stockage, configuration, administration.
- **[Notes de version — Community Server](https://mariadb.com/docs/release-notes/community-server)** — les changelogs versionnés. C'est ici que l'on confirme ce qui change exactement d'une version à l'autre — réflexe d'autant plus important que la formation a souligné de nombreuses différences de comportement entre versions.

## Documentation par produit et outil

Chaque composant de l'écosystème dispose de sa propre section :

- **[MaxScale](https://mariadb.com/docs/maxscale/)** — le proxy de base de données (*load balancing*, *read/write split*, *failover*).
- **[ColumnStore](https://mariadb.com/docs/analytics/)** — le moteur analytique / OLAP.
- **[Galera Cluster](https://mariadb.com/docs/galera-cluster/)** — la réplication synchrone multi-maître.
- **[Connecteurs](https://mariadb.com/docs/connectors/)** — les pilotes (Connector/J, Connector/C, etc.).
- **[Outils](https://mariadb.com/docs/tools/)** — utilitaires et outils d'administration.

À noter : le site documente les versions **Enterprise** de l'opérateur Kubernetes et du serveur MCP. L'**opérateur communautaire** mis en avant dans la formation (§16.5) est un projet distinct — [`mariadb-operator`](https://github.com/mariadb-operator/mariadb-operator) —, compatible avec les images officielles MariaDB ; sa documentation, sa référence d'API et ses exemples sont hébergés avec le projet.

## Ressources de la Fondation et consultation hors-ligne

- **[mariadb.org](https://mariadb.org/)** — le site de la Fondation MariaDB : téléchargements, actualités et accès à la documentation.
- **[Documentation au format PDF](https://mariadb.org/documentation-as-pdf/)** — l'ensemble de la documentation serveur en un fichier unique, pratique pour une consultation hors-ligne.
- **Pages de manuel (*man pages*)** — une fois MariaDB installé, chaque binaire dispose de sa propre page de manuel, consultable via `man <commande>` (par exemple `man mariadb`).

## Pour creuser : le suivi des évolutions

Le **[suivi MDEV sur JIRA](https://jira.mariadb.org/)** recense les anomalies et les évolutions de MariaDB, sous le projet « MDEV ». Les notes de version y renvoient fréquemment : c'est la ressource pour comprendre en détail un comportement propre à une version donnée.

---

En complément, pensez à confronter ce que vous consultez à votre propre version : le tableau des versions figure en [Annexe G](../g-versions-reference/README.md), et les spécificités de la cible 12.3 en [Annexe F](../f-nouveautes-12-3/README.md).

⏭️ [Communautés et forums](/annexes/h-ressources-documentation/02-communautes-forums.md)
