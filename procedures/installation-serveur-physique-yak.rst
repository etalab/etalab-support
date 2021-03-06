************************************
Installation du serveur physique yak
************************************


Installation de l'OS de host de monitoring à partir de la console rescue-pro d'OVH
==================================================================================

Supprimer toute trace de l'ancien RAID :

::

  mdadm --stop /dev/md1
  mdadm --stop /dev/md2
  mdadm --zero-superblock /dev/sda1
  mdadm --zero-superblock /dev/sda2
  mdadm --zero-superblock /dev/sdb1
  mdadm --zero-superblock /dev/sdb2

Histoire d'être sûr :

::

  dd if=/dev/zero of=/dev/sda bs=1M count=100
  dd if=/dev/zero of=/dev/sdb bs=1M count=100

On repartionne sda & sba :
- une partition comprenant la totalité de l'espace disque en type fd (RAID Logiciel)

On créé un nouveau RAID 1 sur sda1 & sdb1 :

::

  mdadm --create /dev/md0 -l 1 -n 2 /dev/sda1 /dev/sdb1

On met du LVM sur /dev/md0 :

::

  pvcreate /dev/md0
  vgcreate vg_ns3362306 /dev/md0
  lvcreate -L4G -n root vg_ns3362306
  lvcreate -L5G -n var vg_ns3362306
  lvcreate -L1G -n tmp vg_ns3362306
  lvcreate -L2G -n swap vg_ns3362306
  mkfs.ext4 /dev/vg_ns3362306/root
  mkfs.ext4 /dev/vg_ns3362306/tmp
  mkfs.ext4 /dev/vg_ns3362306/var
  tune2fs  -i 0 -c 0 /dev/vg_ns3362306/root
  tune2fs  -i 0 -c 0 /dev/vg_ns3362306/tmp
  tune2fs  -i 0 -c 0 /dev/vg_ns3362306/var
  mkswap /dev/vg_ns3362306/swap

On monte la nouvelle arboressence du système dans /mnt/ :

::

  mount /dev/vg_ns3362306/root /mnt
  mkdir /mnt/var
  mkdir /mnt/tmp
  mount /dev/vg_ns3362306/var /mnt/var/
  mount /dev/vg_ns3362306/tmp /mnt/tmp

On lance un debbootstrap dans /mnt/ :

::

   debootstrap --arch=amd64 wheezy /mnt/ http://debian.mirrors.ovh.net/debian/

On alimente le /dev du chroot :

::

  rsync -av /dev/ /mnt/dev/

On entre dans le chroot :

::

  chroot /mnt/

On définit le mot de passe root

::

  passwd

On créé le fichier fstab :

::

  # /etc/fstab: static file system information.
  #
  # Use 'blkid' to print the universally unique identifier for a
  # device; this may be used with UUID= as a more robust way to name devices
  # that works even if disks are added and removed. See fstab(5).
  #
  # <file system> <mount point>   <type>  <options>       <dump>  <pass>
  proc                          /proc   proc    defaults                        0       0
  sysfs                         /sys    sysfs   defaults                        0       0
  /dev/mapper/vg_ns3362306-root  /       ext4    errors=remount-ro,relatime      0       1
  /dev/mapper/vg_ns3362306-var   /var    ext4    defaults                        0       0
  /dev/mapper/vg_ns3362306-tmp   /tmp    ext4    defaults                        0       0
  /dev/mapper/vg_ns3362306-swap  none    swap    defaults                        0       0

On monte le /proc et /sys :

::

  mount /proc
  mount /sys

- Dans ``/etc/hosts``, mettre les lignes ::

  127.0.1.1       yak.data.gouv.fr yak
  37.187.72.214   ns3362306

- Modifier ``/etc/network/interfaces`` ::

::

  # This file describes the network interfaces available on your system
  # and how to activate them. For more information, see interfaces(5).

  # The loopback network interface
  auto lo
  iface lo inet loopback

  auto eth0
  iface eth0 inet static
  	address 37.187.72.214
  	netmask 255.255.255.0
  	network 37.187.72.0
  	broadcast 37.187.72.255
  	gateway 37.187.72.254

- /etc/resolv.conf :

::

  nameserver 213.186.33.99

- Définition du *hostname* :

::

  hostname yak
  echo yak > /etc/hostname

On ajoute les dépôts Debian suivant en plus de l'actuel :

::

  deb http://security.debian.org/ wheezy/updates main
  deb http://debian.easter-eggs.org/debian wheezy main
  deb http://ftp.fr.debian.org/debian wheezy-backports main contrib non-free

On effectue une installation de base :

::

  apt-get update
  apt-get install eeinstall
  eeinstall base

Remarque : Durant l'installation des paquets, laisser les choix par défaut et choisir la locale **en_US.UTF-8**

Configuration de alerte mail :

::

  echo "root: supervision@etalab2.fr" >> /etc/aliases
  newaliases

On install un kernel :

::

  apt-get install linux-image-3.11-0.bpo.2-amd64

On install mdadm & grub :

::

  apt-get install mdadm grub2

Remarque : choisir d'installer grub sur sda et sdb.

On modifie ensuite le paramètre rootdelay du kernel (particularité du 3.11). Pour cela il faut modifier la variable ``GRUB_CMDLINE_LINUX_DEFAULT`` dans le fichier ``/etc/default/grub`` et mettre la valeur ``"rootdelay=8"``. Il faut ensuite lancer la commande :

::

  update-grub

On peut ensuite rebooter la machine

Mise en place de la configuration SSH
-------------------------------------

On modifie l'emplacement de stockage des clés SSH :

::

  sed -i 's/^#AuthorizedKeysFile.*$/AuthorizedKeysFile \/etc\/ssh\/authorized_keys\/%u/' /etc/ssh/sshd_config
  mkdir /etc/ssh/authorized_keys
  /etc/init.d/ssh restart

Ajout d'un utilisateur etalab
------------------------------

::

  adduser etalab

**Remarque :** Pour la connexion SSH via une clé avec cette utilisateur, la clé doit être mise dans le fichier */etc/ssh/authorized_keys/etalab*.


Installation de postfix
=======================

Installer et configurer Postfix ::

  aptitude purge exim4 exim4-base exim4-config exim4-daemon-light postfix+
    General type of mail configuration:
      Internet Site
    System mail name:
      yak.data.gouv.fr
    Root and postmaster mail recipient:
      etalab
    Other destinations to accept mail for (blank for none):
      yak.data.gouv.fr, ns3362306.ovh.net, localhost.ovh.net, localhost
    Force synchronous updates on mail queue?
      No
    Local networks:
      127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
    Mailbox size limit (bytes):
      0
    Local address extension character:
      +
    Internet protocols to use:
      ipv4

Dans ``/etc/posfix/main.cf``, modifier la ligne ::

  myhostname = ns3362306.ovh.net


en ::

  myhostname = yak.data.gouv.fr

Éditer le fichier ``/etc/aliases`` pour y ajouter ::

  axel: axel@haustant.fr
  emmanuel: emmanuel@raviart.com
  etalab: axel,emmanuel

Indexer la base et mettre à jour Postfix ::

  newaliases
  service postfix reload


Installation de fail2ban
========================

::

  apt-get install fail2ban

Le check SSH est activé par défaut avec un ban au bout de 6 erreurs. Ceci peut-être modifié en éditant le fichier */etc/fail2ban/jail.conf* et en modifiant le paramètre *maxretry* de la section *[ssh]*.

Pour faire en sorte que certaine IP ne soit jamais bannies, il faut éditer le paramètre *ignoreip* de la section *[DEFAULT]*. Ce paramètre liste les adresses IP qui ne seront jamais bannies (liste séparée par des espaces).

Etant donné que Fail2ban utilise des règles Netfilter pour bloquer les IP bannies et que nous mettons par ailleurs en place un pare-feu à base de règles Netfilter également, le service Fail2ban ne sera pas démarrer directement mais le sera via le script packetfilter qui manipulera également nos règles de pare-feu. Nous allons donc désactiver le lancement automatique de Fail2ban et faire en sorte que celui-ci ne soit pas réactiver en cas de mise à jour du paquet Debian :

::

  insserv -r -f fail2ban
  echo "#! /bin/sh
  ### BEGIN INIT INFO
  # Provides:          fail2ban
  # Required-Start:    $local_fs $remote_fs
  # Required-Stop:     $local_fs $remote_fs
  # Should-Start:      $time $network $syslog iptables firehol shorewall ipmasq arno-iptables-firewall
  # Should-Stop:       $network $syslog iptables firehol shorewall ipmasq arno-iptables-firewall
  # Default-Start:
  # Default-Stop:      0 1 2 3 4 5 6
  # Short-Description: Start/stop fail2ban
  # Description:       Start/stop fail2ban, a daemon scanning the log files and
  #                    banning potential attackers.
  ### END INIT INFO" > /etc/insserv/overrides/fail2ban
  insserv fail2ban


Installation du pare-feu
========================

Mettre en place les fichiers suivant (commun à tout les hyperviseurs) :

- **packetfilter** dans */etc/init.d/*
- **etalab.conf** dans */etc/*

**Remarque :** les droits de ces fichiers doivent être *0750*.

Il faut ensuite activer le service au démarrage :

::

  insserv packetfilter

Arrêt/démarrage du parefeu
--------------------------

Démarrage :

::

  service packetfilter start

Arrêt :

::

  service packetfilter stop

Status :

::

  service packetfilter status


Installation de Munin
=====================


Installation du serveur central
-------------------------------

Installation du paquet debian

::

  apt-get install munin apache2

On ajoute les hosts a monitorer en ajoutant dans le fichier */etc/munin/munin.conf* :

::

  [ns3362306.ovh.net]
    address ns3362306.ovh.net
    use_node_name yes

  [ns235513.ovh.net]
      address ns235513.ovh.net
      use_node_name yes

  [ns235977.ovh.net]
      address ns235977.ovh.net
      use_node_name yes

  [ns236004.ovh.net]
      address ns236004.ovh.net
      use_node_name yes


Configuration d'Apache
----------------------

::

  ln -s /etc/munin/apache.conf /etc/apache2/conf.d/munin

Créer ensuite le fichier ``/etc/apache2/conf.d/munin-etalab`` ::


  <Location /munin/>
  	Allow from all
  </Location>

  <Location /munin-cgi/munin-cgi-html>
  	Allow from all
  </Location>

  <Location /munin-cgi/munin-cgi-graph>
  	Allow from all
  </Location>

Recharger la configuration d'Apache ::

  service apache2 reload

Munin est accessible à l'adresse http://yak.data.gouv.fr/munin/


Installation du client
----------------------

Installation du paquet debian ::

  apt-get install munin-node

Autoriser les connexions du serveur central en ajoutant dans le fichier ``/etc/munin/munin-node.conf`` ::

  allow ^37\.187\.72\.214$


Mise en place des plugins ceph pour Munin
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

On prépare tout d'abord un utilisateur ceph pour munin (sur **ns235513**) :

::

  ceph auth get-or-create client.munin mon 'allow r' > /etc/ceph/ceph.client.munin.keyring
  chown munin: /etc/ceph/ceph.client.munin.keyring
  chmod 400 /etc/ceph/ceph.client.munin.keyring
  scp /etc/ceph/ceph.client.munin.keyring 192.168.0.2:/etc/ceph
  ssh 192.168.0.2 'chown munin: /etc/ceph/ceph.client.munin.keyring'
  scp /etc/ceph/ceph.client.munin.keyring 192.168.0.3:/etc/ceph
  ssh 192.168.0.3 'chown munin: /etc/ceph/ceph.client.munin.keyring'

On récupère le repos git *contrib* du projet *Munin* :

::

  apt-get install git-core
  cd /usr/local/src
  git clone http://git.zionetrix.net/git/munin-ceph-status

On met en place les plugins :

::

  cd /etc/munin/plugins
  ln -s /usr/local/src/munin-ceph-status/ceph_status ceph_usage
  ln -s /usr/local/src/munin-ceph-status/ceph_status ceph_osd
  ln -s /usr/local/src/munin-ceph-status/ceph_status ceph_mon

On configure le plugin en ajoutant le bloc suivant dans le fichier */etc/munin/plugin-conf.d/munin-node* :

::

  [ceph_*]
  user munin
  env.ceph_keyring /etc/ceph/ceph.client.munin.keyring
  env.ceph_id munin

On redémarre *munin-node* pour qu'il prenne en compte ces nouveaux plugins :

::

  service munin-node restart

Au prochain lancement du cron sur le serveur central, les nouveaux plugins seront détectés et graphés.


Installation de Nagios
======================


Installation du serveur central
-------------------------------

::

  apt-get install icinga
  chmod g+rx /var/lib/icinga/rw
  adduser www-data nagios
  service apache2 stop
  service apache2 start

**Remarque :** Autoriser l'activation des *external commands* durant l'installation, accepter la configuration automatique d'apache et entrer le mot de passe *admin*.


Installation des plugins supplémentaires
----------------------------------------


check_etalab_nrpe
~~~~~~~~~~~~~~~~~

::

  apt-get install nagios-nrpe-plugin

Définition de la commande de check Nagios :

::

  define command {
        command_name    check_etalab_nrpe
        command_line    /usr/lib/nagios/plugins/check_nrpe -t 90 -H $HOSTADDRESS$ -c $ARG1$
  }


check_etalab_ssl_cert
~~~~~~~~~~~~~~~~~~~~~

::

  apt-get install nmap
  cd /usr/local/lib/nagios/plugins

Mettre en place ensuite le script *check_ssl_cert* dans */usr/local/lib/nagios/plugins* avec les droits *0755*.

Définition de la commande de check Nagios :

::

  define command {
        command_name    check_etalab_ssl_cert
        command_line    /usr/local/lib/nagios/plugins/check_ssl_cert -H $HOSTADDRESS$ -p $ARG1$
  }


check_etalab_webinject
~~~~~~~~~~~~~~~~~~~~~~

::

  apt-get install libwebinject-perl nagios-plugins-contrib
  mkdir /etc/webinject
  echo "<useragent>check_http</useragent>
  <timeout>10</timeout>
  <globaltimeout>20</globaltimeout>
  <reporttype>nagios</reporttype>" > /etc/webinject/webinject.xml
  echo "<testcases repeat="1">
  <case
    id="1"
    description1="Searching on www.data.gouv.fr"
    method="get"
    url="http://www.data.gouv.fr/fr/search?q=paris"
    verifypositive=".small.*[1-9]+[0-9]*.r..sultats.*.\/small."
    errormessage="Unable to retrieve a search on www.data.gouv.fr"
  />
  </testcases>" > /etc/webinject/www.data.gouv.fr.xml
  echo "<testcases repeat="1">
  <case
    id="1"
    description1="Page d'accueil wiki.data.gouv.fr"
    method="get"
    url="http://wiki.data.gouv.fr/wiki/Accueil"
    verifypositive=".h1 lang=.fr..*Accueil.*\/h1."
    errormessage="Impossible de recupérer la page d'accueil de wiki.data.gouv.fr"
  />
  </testcases>" > /etc/webinject/wiki.data.gouv.fr.xml

On peut ensuite définir la commande de check Nagios en conséquence :

::

  define command {
        command_name    check_etalab_webinject
        command_line    /usr/lib/nagios/plugins/check_webinject -c /etc/webinject/webinject.xml /etc/webinject/$ARG1$.xml
  }


Installation du plugin de supervision du repos Git
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**A faire sur les trois noeuds Ceph**

Installation du plugin :

::

  git clone http://git.zionetrix.net/git/check_git_config /usr/local/src/check_git_config
  ln -s /usr/local/src/check_git_config/check_git_config /usr/local/lib/nagios/plugins/check_git_config

Installation de la configuration des checks :

::

  echo "nagios  ALL=NOPASSWD:/usr/local/lib/nagios/plugins/check_git_config" > /etc/sudoers.d/nagios-git-config
  chmod 440 /etc/sudoers.d/nagios-git-config
  echo "command[check_git_config]=sudo /usr/local/lib/nagios/plugins/check_git_config /srv/common" > /etc/nagios/nrpe.d/config.cfg
  service nagios-nrpe-server reload


Installation de git
===================

On ajoute un utilisateur *git* :

::

  adduser --home /srv/git --disabled-password git

On met en place les clés SSH autorisées à se connecter au serveur via l'utilisateur *git* en ajoutant dans le fichier */etc/ssh/authorized_keys/git* les clés des utilisateurs *root* des hyperviseurs

Il faut ensuite mettre les données des dépôts git dans */srv/git*. Toutes les dossiers et fichiers se trouvant dans ce dossier doivent appartenir à l'utilisateur *git*.


Installation de Piwik
=====================

::

  aptitude install libapache2-mod-php5
  aptitude install mysql-server
  aptitude install php5-cli
  aptitude install php5-gd
  aptitude install php5-mysql
  aptitude install unzip

En tant qu'etalab ::

  cd
  mkdir repositories
  cd repositories/
  git init --bare data.gouv.fr-certificates.git
  git init --bare stats.data.gouv.fr.git
  cd
  git clone repositories/data.gouv.fr-certificates.git/
  mkdir vhosts
  cd vhosts/
  git clone ../repositories/stats.data.gouv.fr.git
  cd stats.data.gouv.fr/
  wget http://builds.piwik.org/latest.zip
  unzip latest.zip
  rm latest.zip
  rm How\ to\ install\ Piwik.html

En tant que root ::

  chown -R www-data:www-data /home/etalab/vhosts/stats.data.gouv.fr/piwik
  chmod -R 0755 /home/etalab/vhosts/stats.data.gouv.fr/piwik/tmp

  a2enmod ssl

Créer le fichier ``/etc/apache2/conf.d/ssl`` ::

  <IfModule mod_ssl.c>
          NameVirtualHost *:443
          SSLCertificateChainFile /home/etalab/data.gouv.fr-certificates/ca-wildcard-certificate-chain.pem
          SSLCertificateFile /home/etalab/data.gouv.fr-certificates/wildcard.data.gouv.fr-certificate.pem
          SSLCertificateKeyFile /home/etalab/data.gouv.fr-certificates/private-key-raw.pem
  </IfModule>

En tant que root ::

  cd /etc/apache2/sites-available/
  ln -s /home/etalab/vhosts/stats.data.gouv.fr/config/apache2.conf stats.data.gouv.fr.conf
  cd ../sites-enabled/
  rm 000-default
  a2ensite stats.data.gouv.fr.conf
  service apache2 restart


Optimisation de Piwik
---------------------

Dans Piwik "General Settings", mettre :

* Allow Piwik archiving to trigger when reports are viewed from the browser: No
* Reports for today (or any other Date Range including today) will be processed at most every 3600 seconds

Créer le fichier ``/etc/cron.d/etalab`` ::

  MAILTO="supervision@data.gouv.fr"
  # m h dom mon dow user command
  42 * * * * www-data /usr/bin/php5 /home/etalab/vhosts/stats.data.gouv.fr/piwik/misc/cron/archive.php -- url=http://stats.data.gouv.fr/ > /tmp/piwik-archive.log

Puis en tant que root ::

  aptitude install php5-curl

  service cron restart

Éditer le fichier ``/etc/php5/apache2/php.ini`` et mettre ::

  memory_limit = 512M


Activation de la géolocalisation par ville dans Piwik
-----------------------------------------------------

En tant que root ::

  aptitude install libapache2-mod-geoip
  cd /home/etalab/vhosts/stats.data.gouv.fr/piwik/misc/
  wget http://geolite.maxmind.com/download/geoip/database/GeoLiteCity.dat.xz
  xz -d GeoLiteCity.dat.xz
  mv GeoLiteCity.dat GeoIPCity.dat
  chown www-data. GeoIPCity.dat

Éditer le fichier ``/etc/apache2/mods-available/geoip.conf`` et modifier la ligne GeoIPDBFile ::

  <IfModule mod_geoip.c>
    GeoIPEnable On
    #GeoIPDBFile /usr/share/GeoIP/GeoIP.dat
    GeoIPDBFile /home/etalab/vhosts/stats.data.gouv.fr/piwik/misc/GeoIPCity.dat
  </IfModule>

En tant que root ::

  service apache2 restart

Dans Piwik "Geolocation" page :

* Choisir "GeoIP (Apache)".
* Setup automatic updates of GeoIP databases
  * Location Database: http://geolite.maxmind.com/download/geoip/database/GeoLiteCity.dat.gz
  * Update databases every months


Installation de sentry
======================

En tant que root:

.. code-block:: bash

  $ aptitude install build-essential python-dev python-setuptools postgresql-9.1 postgresql-server-dev-9.1 libpq-dev libapache2-mod-wsgi
  $ easy_install pip
  $ pip install virtualenv

  $ su - postgres
  $ createuser -S -D -R -P sentry
    Enter password for new role: XXXX
  $ createdb -O sentry sentry -E utf-8

En tant qu'etalab:

.. code-block:: bash

  $ git init --bare repositories/sentry.data.gouv.fr.git
  $ cd vhosts
  $ git clone ~/repositories/sentry.data.gouv.fr.git
  $ cd sentry.data.gouv.fr
  $ virtualenv .
  $ source bin/activate
  $ pip install sentry[postgres]
  $ sentry init sentry.conf.py
  $ vim sentry.conf.py
  $ sentry --config=sentry.conf.py upgrade
  $ sentry --config=sentry.conf.py createsuperuser
  $ sentry --config=sentry.conf.py repair --owner=axel

En tant que root:

.. code-block:: bash

  $ cd /etc/apache2/sites-available/
  $ ln -s /home/etalab/vhosts/sentry.data.gouv.fr/apache.conf sentry.data.gouv.fr.conf
  $ a2ensite sentry.data.gouv.fr.conf
  $ service apache2 restart
