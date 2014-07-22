
Installation d'un parefeu et de différents services de sécurité, sauvegarde et supervision sur un serveur.

Mise en place d'un dump quotidien des bases MySQL
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Mettre en place le script **mysql_dump** dans **/usr/local/sbin/** (droit *0750*) et adapter les variables *USER* et *PWD* qui doivent le login et mot de passe de l'utilisateur utilisé pour les sauvegardes.

Mettre ensuite en place le cron quotidien ::

  echo "22 22 * * * root /usr/local/sbin/mysql_dump" >> /etc/cron.d/databases_dump


Mise en place d'un dump quotidien des bases PostgreSQL
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Mettre en place le script **psql_dump** dans **/usr/local/bin/** (droit *0755*)

Créer le dossier qui contiendra les sauvegardes et adapter ces droits ::

  mkdir /var/backups/postgres
  chown postgres: /var/backups/postgres
  chmod 0750 /var/backups/postgres

Mettre ensuite en place le cron quotidien ::

  echo "44 22 * * * postgres /usr/local/bin/psql_dump" >> /etc/cron.d/databases_dump


Mise en place d'un dump quotidien des bases MongoDB
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Mettre en place le script **mongodb_dump** dans **/usr/local/sbin/** (droit *0750*)

Mettre ensuite en place le cron journalier ::

  echo "33 22 * * * root /usr/local/sbin/mongodb_dump" >> /etc/cron.d/databases_dump


Installation du client NRPE
---------------------------

::

  apt-get install nagios-nrpe-server nagios-plugins

Installation des plugins supplémentaires ::

  echo "nagios ALL=NOPASSWD: /usr/lib/nagios/plugins/check_apt -u -U -t 60" > /etc/sudoers.d/nagios-apt
  chmod 0440 /etc/sudoers.d/nagios-apt

Installation de la configuration des checks ::

  echo "command[check_apt]=sudo /usr/lib/nagios/plugins/check_apt -u -U -t 60" > /etc/nagios/nrpe.d/apt.cfg
  echo "command[check_disk]=/usr/lib/nagios/plugins/check_disk -w 15% -c 10% -W 15% -K 10% -l -x /dev/shm -e -m" > /etc/nagios/nrpe.d/disk.cfg
  echo "allowed_hosts=37.187.72.214" > /etc/nagios/nrpe.d/etalab.cfg
  echo "command[check_load]=/usr/lib/nagios/plugins/check_load -w 3,5,7 -c 6,8,10" > /etc/nagios/nrpe.d/load.cfg
  echo "command[check_ntp]=/usr/lib/nagios/plugins/check_ntp -H localhost -w 30 -c 60" > /etc/nagios/nrpe.d/ntp.cfg
  echo "command[check_swap]=/usr/lib/nagios/plugins/check_swap -w 40% -c 20%" > /etc/nagios/nrpe.d/swap.cfg
  echo "command[check_fail2ban]=/usr/lib/nagios/plugins/check_procs -c1:1 -C fail2ban-server" > /etc/nagios/nrpe.d/fail2ban.cfg
  echo "command[check_proc_cron]=/usr/lib/nagios/plugins/check_procs -c1:1024 -C cron" > /etc/nagios/nrpe.d/cron.cfg
  echo "command[check_smtp]=/usr/lib/nagios/plugins/check_smtp -H 127.0.0.1" > /etc/nagios/nrpe.d/smtp.cfg

  service nagios-nrpe-server restart


Mise en place d'un check RAID
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

  git clone https://github.com/glensc/nagios-plugin-check_raid /usr/local/src/nagios-plugin-check_raid
  mkdir -p /usr/local/lib/nagios/plugins
  ln -s /usr/local/src/nagios-plugin-check_raid/check_raid.pl /usr/local/lib/nagios/plugins/check_raid.pl
  echo "command[check_raid]=/usr/local/lib/nagios/plugins/check_raid.pl" > /etc/nagios/nrpe.d/raid.cfg


Mise en place d'un check Ceph
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Préparation des noeuds ceph pour la supervision
"""""""""""""""""""""""""""""""""""""""""""""""

* Sur **ns235513** ::

  ceph auth get-or-create client.nagios mon 'allow r' > /etc/ceph/ceph.client.nagios.keyring
  scp /etc/ceph/ceph.client.nagios.keyring 192.168.0.2:/etc/ceph/
  ssh root@192.168.0.2 'chown nagios: /etc/ceph/ceph.client.nagios.keyring'
  scp /etc/ceph/ceph.client.nagios.keyring 192.168.0.3:/etc/ceph/
  ssh root@192.168.0.3 'chown nagios: /etc/ceph/ceph.client.nagios.keyring'


Installation du plugin de supervision Ceph
""""""""""""""""""""""""""""""""""""""""""

Installation du plugin ::

  git clone https://github.com/valerytschopp/ceph-nagios-plugins.git /usr/local/src/ceph-nagios-plugins
  ln -s /usr/local/src/ceph-nagios-plugins/src/check_ceph_health /usr/local/lib/nagios/plugins/check_ceph_health

  git clone http://git.zionetrix.net/git/check_ceph_usage /usr/local/src/check_ceph_usage
  ln -s /usr/local/src/check_ceph_usage/check_ceph_usage /usr/local/lib/nagios/plugins/check_ceph_usage

  git clone http://git.zionetrix.net/git/check_ceph_status /usr/local/src/check_ceph_status
  ln -s /usr/local/src/check_ceph_status/check_ceph_status /usr/local/lib/nagios/plugins/check_ceph_status


Installation de la configuration des checks ::

  echo "command[check_ceph_health]=/usr/local/lib/nagios/plugins/check_ceph_health -d -i nagios -k /etc/ceph/ceph.client.nagios.keyring" > /etc/nagios/nrpe.d/ceph.cfg
  echo "command[check_ceph_usage]=/usr/local/lib/nagios/plugins/check_ceph_usage -i nagios -k /etc/ceph/ceph.client.nagios.keyring --warning-data 50 --critical-data 60 --warning-allocated 80 --critical-allocated 90" >> /etc/nagios/nrpe.d/ceph.cfg
  echo "command[check_ceph_status]=/usr/local/lib/nagios/plugins/check_ceph_status -i nagios -k /etc/ceph/ceph.client.nagios.keyring" >>/etc/nagios/nrpe.d/ceph.cfg
  service nagios-nrpe-server reload


Installation du plugin de supervision du repos Git de configuration des hyperviseurs
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

**A faire sur les trois noeuds Ceph**

Installation du plugin ::

  git clone http://git.zionetrix.net/git/check_git_config /usr/local/src/check_git_config
  ln -s /usr/local/src/check_git_config/check_git_config /usr/local/lib/nagios/plugins/check_git_config

Installation de la configuration des checks ::

  echo "nagios  ALL=NOPASSWD:/usr/local/lib/nagios/plugins/check_git_config" > /etc/sudoers.d/nagios-git-config
  chmod 440 /etc/sudoers.d/nagios-git-config
  echo "command[check_git_config]=sudo /usr/local/lib/nagios/plugins/check_git_config /srv/common" > /etc/nagios/nrpe.d/config.cfg
  service nagios-nrpe-server reload


Mise en place d'un check MySQL
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Créér l'utilisateur MySQL *nagios* avec la requête *SQL* suivante ::

  CREATE USER nagios@localhost IDENTIFIED BY '[pwd]';

**Avec :**

  * **[pwd] :**  le mot de passe de votre choix

Puis ajouter la commande *check_mysql* dans la configuration d'NRPE et recharger sa configuration ::

  echo "command[check_mysql]=/usr/lib/nagios/plugins/check_mysql -u nagios -p '[mdp]'" > /etc/nagios/nrpe.d/mysql.cfg
  service nagios-nrpe-server reload


Mise en place d'un check PostgreSQL
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Créér l'utilisateur PostgreSQL local *nagios* et lui donner les droits de se connecter ::

  su - postgres
  createuser -D -R -S nagios
  exit
  echo "local    template1       nagios                                  ident" >> /etc/postgresql/9.1/main/pg_hba.conf
  service postgresql reload

Puis ajouter la commande *check_psql* dans la configuration d'NRPE et recharger sa configuration ::

  echo "# Postgresql connexion
  # Requirement:
  #        * as user postgres, run "createuser -D -R -S nagios"
  #        * add this on top of pg_hba.conf rules:
  #          local template1 nagios ident
  command[check_pgsql]=/usr/lib/nagios/plugins/check_pgsql -l nagios" > /etc/nagios/nrpe.d/postgresql.cfg
  service nagios-nrpe-server reload


Mise en place d'un check MongoDB
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Installation du plugin nagios ::

  git clone https://github.com/tag1consulting/check_mongo /usr/local/src/check_mongo
  ln -s /usr/local/src/check_mongo/check_mongo /usr/local/lib/nagios/plugins/

.. note:: Il y avait une coquille dans le script original que j'ai corrigé manuellement. Une pull request a été proposé sur le github du projet pour corrigé cela.

Puis ajouter la commande *check_mongo* dans la configuration d'NRPE et recharger sa configuration ::

  echo "command[check_mongo]=/usr/local/lib/nagios/plugins/check_mongo -H 127.0.0.1 -P 27017 -A connect" > /etc/nagios/nrpe.d/mongo.cfg
  service nagios-nrpe-server reload


Mise en place d'un check Redis
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

  echo "command[check_redis]=/usr/lib/nagios/plugins/check_tcp -H 127.0.0.1 -p 6379" > /etc/nagios/nrpe.d/redis.cfg
  service nagios-nrpe-server reload


Installation de l'agent Munin
-----------------------------

Installation du paquet debian

::

  apt-get install munin-node

On autorise les connexions du serveur centrale en ajoutant dans le fichier */etc/munin/munin-node.conf* ::

  allow ^37\.187\.72\.214$

Puis on relance le service ::

  service munin-node restart


Installation du plugin munin pour Apache
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

  apt-get install liblwp-useragent-determined-perl
  ln -s '/usr/share/munin/plugins/apache_accesses' '/etc/munin/plugins/apache_accesses'
  ln -s '/usr/share/munin/plugins/apache_processes' '/etc/munin/plugins/apache_processes'
  ln -s '/usr/share/munin/plugins/apache_volume' '/etc/munin/plugins/apache_volume'
  service munin-node restart


Installation du plugin munin pour MySQL
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

  apt-get install libcache-cache-perl
  ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_bin_relay_log'
  ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_commands'
  ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_connections'
  ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_files_tables'
  ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_innodb_bpool'
  ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_innodb_bpool_act'
  ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_innodb_insert_buf'
  ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_innodb_io'
  ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_innodb_io_pend'
  ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_innodb_log'
  ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_innodb_rows'
  ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_innodb_semaphores'
  ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_innodb_tnx'
  ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_myisam_indexes'
  ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_network_traffic'
  ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_qcache'
  ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_qcache_mem'
  ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_replication'
  ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_select_types'
  ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_slow'
  ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_sorts'
  ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_table_locks'
  ln -s '/usr/share/munin/plugins/mysql_' '/etc/munin/plugins/mysql_tmp_tables'
  service munin-node restart


Installation du plugin munin pour PostgreSQL
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

  apt-get install libdbd-pg-perl
  ln -s '/usr/share/munin/plugins/postgres_autovacuum' '/etc/munin/plugins/postgres_autovacuum'
  ln -s '/usr/share/munin/plugins/postgres_bgwriter' '/etc/munin/plugins/postgres_bgwriter'
  ln -s '/usr/share/munin/plugins/postgres_cache_' '/etc/munin/plugins/postgres_cache_ALL'
  ln -s '/usr/share/munin/plugins/postgres_cache_' '/etc/munin/plugins/postgres_cache_ckan_default'
  ln -s '/usr/share/munin/plugins/postgres_cache_' '/etc/munin/plugins/postgres_cache_datastore_default'
  ln -s '/usr/share/munin/plugins/postgres_checkpoints' '/etc/munin/plugins/postgres_checkpoints'
  ln -s '/usr/share/munin/plugins/postgres_connections_' '/etc/munin/plugins/postgres_connections_ALL'
  ln -s '/usr/share/munin/plugins/postgres_connections_' '/etc/munin/plugins/postgres_connections_ckan_default'
  ln -s '/usr/share/munin/plugins/postgres_connections_' '/etc/munin/plugins/postgres_connections_datastore_default'
  ln -s '/usr/share/munin/plugins/postgres_connections_db' '/etc/munin/plugins/postgres_connections_db'
  ln -s '/usr/share/munin/plugins/postgres_locks_' '/etc/munin/plugins/postgres_locks_ALL'
  ln -s '/usr/share/munin/plugins/postgres_locks_' '/etc/munin/plugins/postgres_locks_ckan_default'
  ln -s '/usr/share/munin/plugins/postgres_locks_' '/etc/munin/plugins/postgres_locks_datastore_default'
  ln -s '/usr/share/munin/plugins/postgres_querylength_' '/etc/munin/plugins/postgres_querylength_ALL'
  ln -s '/usr/share/munin/plugins/postgres_querylength_' '/etc/munin/plugins/postgres_querylength_ckan_default'
  ln -s '/usr/share/munin/plugins/postgres_querylength_' '/etc/munin/plugins/postgres_querylength_datastore_default'
  ln -s '/usr/share/munin/plugins/postgres_scans_' '/etc/munin/plugins/postgres_scans_ckan_default'
  ln -s '/usr/share/munin/plugins/postgres_scans_' '/etc/munin/plugins/postgres_scans_datastore_default'
  ln -s '/usr/share/munin/plugins/postgres_size_' '/etc/munin/plugins/postgres_size_ALL'
  ln -s '/usr/share/munin/plugins/postgres_size_' '/etc/munin/plugins/postgres_size_ckan_default'
  ln -s '/usr/share/munin/plugins/postgres_size_' '/etc/munin/plugins/postgres_size_datastore_default'
  ln -s '/usr/share/munin/plugins/postgres_transactions_' '/etc/munin/plugins/postgres_transactions_ALL'
  ln -s '/usr/share/munin/plugins/postgres_transactions_' '/etc/munin/plugins/postgres_transactions_ckan_default'
  ln -s '/usr/share/munin/plugins/postgres_transactions_' '/etc/munin/plugins/postgres_transactions_datastore_default'
  ln -s '/usr/share/munin/plugins/postgres_tuples_' '/etc/munin/plugins/postgres_tuples_ckan_default'
  ln -s '/usr/share/munin/plugins/postgres_tuples_' '/etc/munin/plugins/postgres_tuples_datastore_default'
  ln -s '/usr/share/munin/plugins/postgres_users' '/etc/munin/plugins/postgres_users'
  ln -s '/usr/share/munin/plugins/postgres_xlog' '/etc/munin/plugins/postgres_xlog'
  service munin-node restart


Installation du plugin munin pour MongoDB
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

  git clone https://github.com/erh/mongo-munin /usr/local/src/mongo-munin
  ln -s /usr/local/src/mongo-munin/mongo_* /etc/munin/plugins/
  service munin-node restart


Mise en place des plugins munin pour CEPH
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


On prépare tout d'abord un utilisateur ceph pour munin (sur **ns235513**) ::

  ceph auth get-or-create client.munin mon 'allow r' > /etc/ceph/ceph.client.munin.keyring
  chown munin: /etc/ceph/ceph.client.munin.keyring
  chmod 400 /etc/ceph/ceph.client.munin.keyring
  scp /etc/ceph/ceph.client.munin.keyring 192.168.0.2:/etc/ceph
  ssh 192.168.0.2 'chown munin: /etc/ceph/ceph.client.munin.keyring'
  scp /etc/ceph/ceph.client.munin.keyring 192.168.0.3:/etc/ceph
  ssh 192.168.0.3 'chown munin: /etc/ceph/ceph.client.munin.keyring'

On récupère le repos git *contrib* du projet *Munin* ::

  apt-get install git-core
  cd /usr/local/src
  git clone http://git.zionetrix.net/git/munin-ceph-status

On met en place les plugins ::

  cd /etc/munin/plugins
  ln -s /usr/local/src/munin-ceph-status/ceph_status ceph_usage
  ln -s /usr/local/src/munin-ceph-status/ceph_status ceph_osd
  ln -s /usr/local/src/munin-ceph-status/ceph_status ceph_mon

On configure le plugin en ajoutant le bloc suivant dans le fichier */etc/munin/plugin-conf.d/munin-node* ::

  [ceph_*]
  user munin
  env.ceph_keyring /etc/ceph/ceph.client.munin.keyring
  env.ceph_id munin

On redémarre *munin-node* pour qu'il prenne en compte ces nouveaux plugins ::

  service munin-node restart

Au prochain lancement du cron sur le serveur centrale, les nouveaux plugin seront détectés et graphés.

Mise en place des plugins munin pour HAProxy
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
On installe munin ainsi que certaines dépendance nécessaire dans l'environnement d'haproxy ::

    apt-get install munin-node munin-plugins-extra ca-certificates socat

On installe le dépot contrib de munin ::

    cd /usr/local/src/
    git clone https://github.com/munin-monitoring/contrib.git /usr/local/src/munin-monitoring

On met en place les plugins ::

    ln -s /usr/local/src/munin-monitoring/plugins/haproxy/haproxy_rate_frontend /etc/munin/plugins/
    ln -s /usr/local/src/munin-monitoring/plugins/haproxy/haproxy-sessions-by-servers /etc/munin/plugins/
    ln -s /usr/share/munin/plugins/ip_ /etc/munin/plugins/ip_37.59.183.74
    ln -s /usr/share/munin/plugins/ip_ /etc/munin/plugins/ip_37.59.183.75
    ln -s /usr/share/munin/plugins/ip_ /etc/munin/plugins/ip_37.59.183.80
    ln -s /usr/share/munin/plugins/ip_ /etc/munin/plugins/ip_37.59.183.81
    ln -s /usr/share/munin/plugins/ip_ /etc/munin/plugins/ip_37.59.183.70
    ln -s /usr/share/munin/plugins/if_ /etc/munin/plugins/if_eth0
    ln -s /usr/share/munin/plugins/if_ /etc/munin/plugins/if_eth1
    ln -s /usr/share/munin/plugins/if_ /etc/munin/plugins/if_eth2


On configure le plugin en ajoutant le bloc suivant dans le fichier */etc/munin/plugin-conf.d/munin-node* ::
    
    [haproxy*]
    user root
    env.backend WEBUI-DEV API-DEV DATAGOUVFR WIKI-DATAGOUVFR STATIC-DATAGOUVFR
    env.frontend WEBUI-DEV API-DEV DATAGOUVFR WIKI-DATAGOUVFR STATIC-DATAGOUVFR
    env.socket /var/run/haproxy.stat

On redémarre *munin-node* pour qu'il prenne en compte ces nouveaux plugins ::

  service munin-node restart

Au prochain lancement du cron sur le serveur centrale, les nouveaux plugins seront détectés et graphés.

Mise en place des plugins munin pour Varnish
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
On installe le dépôt de basiszwo ::

    cd /usr/local/src/
    git clone git://github.com/basiszwo/munin-varnish.git
    chmod a+x /usr/share/munin/plugins/munin-varnish/varnish_*

On met en place les plugins ::

    cd /etc/munin/plugins
    ln -s /usr/local/src/munin-varnish/varnish_cachehitratio 
    ln -s /usr/local/src/munin-varnish/varnish_hitrate

On configure les plugins ::

    cat < EOF >> /etc/munin/plugin-conf.d/munin-node
    
    [varnish*]
    user root
    EOF
    
On redémarre *munin-node* pour qu'il prenne en compte ces nouveaux plugins ::

  service munin-node restart

Au prochain lancement du cron sur le serveur centrale, les nouveaux plugins seront détectés et graphés.



