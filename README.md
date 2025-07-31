## Clusteur galera / HA webserver

## Haute disponibilité des serveurs web

Ce lab vous guide pas à pas pour mettre en place un environnement de haute disponibilité de serveurs web comprenant un cluster de bases de données MariaDB Galera, un load balancing HAProxy avec Keepalived pour la redondance, et des serveurs web Wordpress sur Rocky Linux dans VMware Workstation.

## Prérequis

- Utiliser **VMware Workstation à jour** (version recommandée 17.6.3).
- Configurer le réseau **NAT** dans VMware sur la plage `192.168.10.0/24` avec la passerelle `192.168.10.254`.
- Créer un **template VM Rocky Linux** dernière version (9.5 actuellement).
- Le lab nécessite **7 machines virtuelles** en fonctionnement simultané :
  - 3 serveurs MariaDB (cluster Galera)
  - 2 serveurs HAProxy (load balancing + keepalived)
  - 2 serveurs Wordpress

## Préparation des machines virtuelles

- Installer Rocky Linux 9 (version minimale) sur une VM.
- Ne pas définir de mot de passe root pendant l’installation.
- Activer un **profil de sécurité minimal**.
- Cloner la VM 7 fois.
- Configurer les noms d’hôtes et adresses IP statiques selon la topologie réseau planifiée.

## Installation et configuration MariaDB Galera

1. Installer les paquets nécessaires :

   ```bash
   sudo dnf install vim mariadb-server mariadb-server-galera galera policycoreutils-python-utils setools-console -y
   ```

2. Configurer une IP statique et modifier le nom d’hôte pour chaque serveur MariaDB.
3. Ouvrir le port Galera dans le firewall :

   ```bash
   sudo firewall-cmd --add-service=galera --permanent
   sudo firewall-cmd --reload
   ```

4. Copier la configuration Galera fournie dans `/etc/my.cnf.d/galera.conf` sur chaque nœud en remplaçant l’ancien fichier.
5. Initialiser le cluster uniquement sur **MariaDB1** :

   ```bash
   sudo galera_new_cluster
   ```

6. Lancer MariaDB sur tous les nœuds :

   ```bash
   sudo systemctl enable --now mariadb
   ```

7. Sécuriser l’installation MariaDB :

   ```bash
   sudo mysql_secure_installation
   ```

   - Répondre `n` à la question sur le switch vers UNIX socket.
   - Changer le mot de passe root.

8. Suivre les recommandations du fichier `mysql_secure_installation_output` pour finaliser la sécurisation.

## Installation et configuration de Keepalived

1. Installer Keepalived et HAProxy :

   ```bash
   sudo dnf install vim haproxy keepalived policycoreutils-python-utils setools-console -y
   ```

2. Copier respectivement les configurations `keepalived-master` sur HAProxy1 et `keepalived-slave` sur HAProxy2.
3. Activer et démarrer le service Keepalived sur les deux HAProxy :

   ```bash
   sudo systemctl enable --now keepalived
   ```

## Installation et configuration de HAProxy

1. Autoriser les services HTTP et MySQL dans le firewall :

   ```bash
   sudo firewall-cmd --add-service={http,mysql} --permanent
   sudo firewall-cmd --add-port={3307,9000}/tcp --permanent
   sudo firewall-cmd --reload
   ```

2. Configurer SELinux pour autoriser HAProxy à communiquer sur le réseau :

   ```bash
   sudo setsebool -P haproxy_connect_any 1
   sudo semanage port -a -t http_port_t -p tcp 3306
   sudo semanage port -a -t http_port_t -p tcp 3307
   ```

3. Copier la configuration HAProxy depuis `haproxy_conf` sur les deux serveurs.
4. Activer et lancer HAProxy :

   ```bash
   sudo systemctl enable --now haproxy
   ```

## Installation et configuration de Wordpress

1. Installer les paquets nécessaires :

   ```bash
   sudo dnf install httpd php php-mysqlnd php-fpm php-json -y
   ```

2. Démarrer et activer Apache :

   ```bash
   sudo systemctl enable --now httpd
   ```

3. Autoriser HTTP dans le firewall et recharger :

   ```bash
   sudo firewall-cmd --add-service=http --permanent
   sudo firewall-cmd --reload
   ```

4. Copier la configuration du fichier `wordpress-vhost` dans `/etc/httpd/conf.d/wordpress.conf` (à créer).
5. Supprimer le fichier `welcome.conf` par défaut :

   ```bash
   sudo rm /etc/httpd/conf.d/welcome.conf
   ```

6. Appliquer ces règles SELinux sur chaque serveur Wordpress :

   ```bash
   sudo setsebool -P httpd_can_network_connect 1
   sudo setsebool -P httpd_can_network_connect_db 1
   ```

7. Installer Wordpress dans `/var/www/html` selon la procédure habituelle.

## Vérification et utilisation

- Le cluster MariaDB Galera doit être opérationnel avec les trois nœuds synchronisés.
- HAProxy et Keepalived assurent un load balancing actif/passif entre les deux instances.
- Wordpress fonctionne en haute disponibilité sur les deux serveurs web.
- Vous pouvez consulter les statistiques HAProxy via le port 9000 sur la VIP ou les IP des HAProxy.
- Toutes les machines doivent communiquer sur le réseau NAT configuré.

Ce lab vous offre un environnement complet et opérationnel de haute disponibilité pour serveurs web, basé sur Rocky Linux dans VMware Workstation, conforme aux bonnes pratiques de sécurisation et de gestion de cluster pour un service résilient.
