***********************************
Installation du serveur virtuel ant
***********************************

Dans la console web OVH, associer l'adresse IP failover 87.98.177.42 à ant et lui donner l'adresse MAC 02:00:00:99:59:b6

Aller sur le serveur physique turtle.data.gouv.fr, en tant que root ::

  qemu-img create -f rbd rbd:libvirt-sata/ant 20G
  create-virtual-machine-failover ant 02:00:00:99:59:b6
    Check name : OK
    Check RBD image : rbd image 'ant':
	    size 20480 MB in 5120 objects
	    order 22 (4096 kB objects)
	    block_name_prefix: rb.0.2615.238e1f29
	    format: 1
    Create VM :
    Domain ant defined from /tmp/tmp.J94nubjZJB

    Id:             -
    Name:           ant
    UUID:           63b1cadb-c7f3-4096-8df0-d2279e1d6b7f
    OS Type:        hvm
    State:          shut off
    CPU(s):         2
    Max memory:     8388608 KiB
    Used memory:    1048576 KiB
    Persistent:     yes
    Autostart:      disable
    Managed save:   no
    Security model: none
    Security DOI:   0

Dans Virtual Machine Manager, lancer ant et lancer l'installation ::

  Language:
    English
  Country, territory or area:
    other
  Continent or region:
    Europe
  Country, territory or area:
    France
  Country to base default locale settings on:
    United States - en_US.UTF-8
  Keymap to use:
    French
  Network configuration method:
    Do not configure the network at this time
  Hostname:
    ant
  Root password:
    123456
  Full name for the new user:
    Etalab
  Username for your account:
    etalab
  Choose a password for the new user:
    123456
  Partitioning method:
    Guided - use entire disk and setup LVM
  Partitioning scheme:
    All files in one partition
    Finish partitioning and write changes to disk
  Continue without a network mirror:
    Yes
  Participate in the package user survey?
    No
  Install the GRUB boot loader to the master boot record?
    Yes

Au redémarrage, se logger en tant que root, puis ::

  nano /etc/network/interfaces

et ajouter l'interface eth0 ::

  auto eth0
  iface eth0 inet static
          address 87.98.177.42
          netmask 255.255.255.255
          broadcast 87.98.177.42
          post-up route add 178.33.236.163 dev eth0
          post-up route add default gw 178.33.236.163 dev eth0
          post-down route del 178.33.236.163 dev eth0
          post-down route del default gw 178.33.236.163

Activer l'interface réseau ``eth0`` ::

  ifup eth0

Créer le fichier ``/etc/resolv.conf`` ::

  nameserver 213.186.33.99

Remplacer le fichier ``/etc/apt/sources.list`` par ::

  deb http://debian.mirrors.ovh.net/debian wheezy main
  # deb-src http://debian.mirrors.ovh.net/debian wheezy main

  deb http://security.debian.org/ wheezy/updates main
  # deb-src http://security.debian.org/ wheezy/updates main

  # wheezy-updates, previously known as 'volatile'
  deb http://debian.mirrors.ovh.net/debian wheezy-updates main
  # deb-src http://debian.mirrors.ovh.net/debian wheezy-updates main

Installer les paquets manquants ::

  aptitude update
  aptitude install task-ssh-server

Quitter maintenant ``virt-manager`` et lancer une connexion ssh ``ssh root@87.98.177.42``, puis ::

Modifier le fichier ``/etc/ssh/sshd_config`` et ajouter les lignes ::

  AuthorizedKeysFile /etc/ssh/authorized_keys/%u
  PasswordAuthentication no

Configurer ssh ::

  mkdir /etc/ssh/authorized_keys

Puis créér les fichiers ``/etc/ssh/authorized_keys/root`` et ``/etc/ssh/authorized_keys/etalab`` en y mettant les clés publiques ssh.

Redémarrer ssh ::

  service ssh restart

Puis tester la connexion ssh en tant que ``root`` et ``etalab``.

Changer le mot de passe de ``root`` et ``etalab`` en quelque chose de sûr ::

  passwd
  passwd etalab

Dans ``/etc/hosts``, modifier la ligne ::
  127.0.1.1       ant

en ::

  127.0.1.1       ant.data.gouv.fr ant

Créer le fichier ``/etc/apt/apt.conf.d/50norecommends`` pour y mettre la ligne ::

  APT::Install-Recommends "false";

Installer les paquets manquants ::

  aptitude install htop
  aptitude install less
  aptitude install molly-guard
  aptitude install ntp
  aptitude install sshguard


Revenir sur le serveur physique, en tant que root, puis ::

  virsh dumpxml ant > /tmp/ant.xml
  scp /tmp/ant.xml 192.168.0.2:/tmp/
  ssh 192.168.0.2 "virsh define '/tmp/ant.xml'"
  scp /tmp/ant.xml 192.168.0.3:/tmp/
  ssh 192.168.0.3 "virsh define '/tmp/ant.xml'"


Installation de postfix
=======================

Installer et configurer Postfix ::

  aptitude purge exim4 exim4-base exim4-config exim4-daemon-light postfix+
    General type of mail configuration:
      Internet Site
    System mail name:
      data.gouv.fr
    Root and postmaster mail recipient:
      etalab
    Other destinations to accept mail for (blank for none):
      data.gouv.fr, ant.data.gouv.fr, localhost.localdomain, localhost
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

  myhostname = ant

en ::

  myhostname = ant.data.gouv.fr

Éditer le fichier ``/etc/aliases`` pour y ajouter ::

  aide-producteur: axel,emmanuel,romain
  axel: axel@haustant.fr
  contact: axel,emmanuel,romain
  emmanuel: emmanuel@raviart.com
  etalab: axel,emmanuel
  romain: romain.tales@pm.gouv.fr
  supervision: axel,emmanuel

Indexer la base et mettre à jour Postfix ::

  newaliases
  service postfix reload

Installer SpamAssassin ::

  aptitude install spamassassin
  aptitude install spamc

Dans /etc/default/spamassassin mettre ::
  ENABLED=1

Puis ::

  service spamassassin restart

Dans ``/etc/postfix/main.cf``, ajouter à la fin ::

  # Added by Etalab. See http://wiki.debian.org/Postfix.

  smtpd_recipient_restrictions = permit_mynetworks,
        reject_invalid_hostname,
        reject_unknown_recipient_domain,
        reject_unauth_destination,
        reject_rbl_client zen.spamhaus.org,
        permit

  smtpd_helo_restrictions = reject_invalid_helo_hostname,
        reject_non_fqdn_helo_hostname
  #       reject_unknown_helo_hostname

  # To uncomment for Postfix 2.10:
  # smtpd_relay_restrictions = permit_mynetworks,
  #      reject_invalid_hostname,
  #       reject_unknown_recipient_domain,
  #       reject_unauth_destination,
  #       reject_rbl_client zen.spamhaus.org,
  #       permit

Dans ``/etc/postfix/master.cf``, rajouter ``-o content_filter=spamassassin`` à la première ligne de la manière suivante ::
  smtp      inet  n       -       -       -       -       smtpd
    -o content_filter=spamassassin

Et en fin de fichier ajouter ::
  # Added by Etalab:
  spamassassin unix -     n       n       -       -       pipe
    user=debian-spamd argv=/usr/bin/spamc -f -e
    /usr/sbin/sendmail -oi -f ${sender} ${recipient}

        <pre>${u'''
/etc/init.d postfix restart
        '''.strip()}</pre>


Installation de wiki.data.gouv.fr
=================================

En tant que root ::

  aptitude install git
  aptitude install libapache2-mod-php5
  aptitude install mysql-server
  aptitude install php-apc
  aptitude install php5-cli
  aptitude install php5-gd
  aptitude install php5-intl
  aptitude install php5-mysql

En tant qu'etalab ::

  cd ~/repositories/
  git init --bare data.gouv.fr-certificates.git
  git init --bare wiki.data.gouv.fr.git
  cd
  git clone repositories/data.gouv.fr-certificates.git/
  mkdir vhosts
  cd vhosts/
  git clone ../repositories/wiki.data.gouv.fr.git
  cd wiki.data.gouv.fr/
  wget http://download.wikimedia.org/mediawiki/1.22/mediawiki-1.22.0.tar.gz
  tar xzf mediawiki-1.22.0.tar.gz
  mv mediawiki-1.22.0 mediawiki

En tant que root ::

  cd /home/etalab/vhosts/wiki.data.gouv.fr/mediawiki/
  chown www-data. images/

  a2enmod ssl

Créer le fichier ``/etc/apache2/conf.d/ssl`` ::

  <IfModule mod_ssl.c>
          NameVirtualHost *:443
          SSLCertificateChainFile /home/etalab/data.gouv.fr-certificates/ca-wildcard-certificate-chain.pem
          SSLCertificateFile /home/etalab/data.gouv.fr-certificates/wildcard.data.gouv.fr-certificate.pem
          SSLCertificateKeyFile /home/etalab/data.gouv.fr-certificates/private-key-raw.pem
  </IfModule>

Éditer le fichier ``/etc/apache2/ports.conf`` et remplacer 80 en 8080 dans les lignes ::

  NameVirtualHost *:8080
  Listen 8080

En tant que root ::

  cd /etc/apache2/sites-available/
  ln -s /home/etalab/vhosts/wiki.data.gouv.fr/config/apache2.conf wiki.data.gouv.fr.conf
  cd ../sites-enabled/
  rm 000-default
  a2ensite wiki.data.gouv.fr.conf
  service apache2 restart

Depuis un navigateur, aller sur http://wiki.data.gouv.fr/

* Your language: en - English
* Wiki language: fr -français
* Database type: MySQL
* Database host: localhost
* Database name: my_wiki
* Database table prefix:
* Database username: root
* Database password: XXXX
* Database account for web access: Décocher "Use the same account as for installation"
* Storage engine: InnoDB
* Database character set: Binary
* Name of wiki: WikiEtalab
* Project namespace: Same as the wiki name: $1
* Administrator account
  * Your name: Emmanuel Raviart
  * Password: XXXX
  * Password again: XXXX
  * Email address: emmanuel@raviart.com
* Ask me more questions.
* User rights profile: Open wiki
* Copyright and license: No license footer
* Enable outbound email
* Return email address: webmaster+wiki@data.gouv.fr
* Disable user talk page notification
* Disable watchlist notification
* Enable email authentication
* Extensions: none
* Enable file uploads
* Logo URL: $wgStylePath/common/images/wiki.png
* Disable Instant Commons
* Settings for object caching: PHP object caching (APC, XCache or WinCache)

Recopier le fichier ``LocalSettings.php`` ainsi généré dans le répertoire ``wiki.data.gouv.fr/config`` du PC local, l'adapter, le commiter; puis le pusher.

Ensuite en tant qu'etalab ::

  cd ~/vhosts/wiki.data.gouv.fr/
  git pull
  cd mediawiki/
  ln -s ../config/LocalSettings.php
  cd skins
  git clone https://github.com/etalab/mediawiki-etalab-skin etalab


Installation de fedmsg
======================

En tant que root ::

  aptitude install python-pip

Regarder les paquets nécessaires pour fedmsg ::

  pip install --no-install fedmsg

En installer le plus possible en utilisant les paquets Debian ::

  aptitude install python-daemon
  aptitude install python-decorator
  aptitude install python-dev
  aptitude install python-pygments
  aptitude install python-requests
  aptitude install python-twisted
  aptitude install python-tz

Installer fedmsg ::

  pip install fedmsg

Modifier le fichier ``/etc/fedmsg.d/base.py`` ::

  environment = 'prod',
  topic_prefix = 'fr.gouv.data',

Dans ``/etc/fedmsg.d/endpoints.py``, commenter tous les endpoints.

Dans ``/etc/fedmsg.d/ssl.py``, supprimer la signature des messages ::

  validate_signatures=False,

Tester que fedmsg fonctionne correctement en lançant dans 3 terminaux différents ::

  fedmsg-relay

  fedmsg-tail --really-pretty

  echo "Hello, world" | fedmsg-logger


Installation de circus-fedmsg
-----------------------------

Regarder les paquets nécessaires pour circus ::

  pip install --no-install circus

Installer circus ::

  pip install circus

En tant qu'etalab ::

  cd
  mkdir repositories
  cd repositories/
  git init --bare circus-fedmsg.ant.data.gouv.fr.git
  cd ..
  git clone repositories/circus-fedmsg.ant.data.gouv.fr.git/
  cd circus-fedmsg.ant.data.gouv.fr
  mkdir ipc

En tant que root ::

  cd /var/log/
  mkdir circus-fedmsg

  cd /etc/logrotate.d/
  ln -s /home/etalab/circus-fedmsg.ant.data.gouv.fr/circus-fedmsg.logrotate circus-fedmsg

  cd /etc/init.d/
  ln -s /home/etalab/circus-fedmsg.ant.data.gouv.fr/circus-fedmsg.init circus-fedmsg
  update-rc.d circus-fedmsg defaults
  service circus-fedmsg restart

Tester que fedmsg fonctionne correctement en lançant dans 2 terminaux différents ::

  fedmsg-tail --really-pretty

  echo "Hello, world" | fedmsg-logger


Installation de fedmsg-emit.php
-------------------------------

Installer les bindings PHP pour zmq ::

  aptitude install make
  aptitude install php5-dev
  aptitude install pkg-config
  aptitude install libzmq-dev

En tant qu'etalab ::

  cd ~
  git clone git://github.com/mkoppanen/php-zmq.git
  cd php-zmq
  phpize
  ./configure
  make

En tant que root ::

  make install
    Installing shared extensions:     /usr/lib/php5/20100525/

Créer le fichier /etc/php5/apache2/conf.d/30-zmq.ini ::

  ; configuration for php 0MQ module
  ; priority=30
  extension=zmq.so

Télécharger et installer `fedmsg-init.php <https://github.com/fedora-infra/fedmsg/blob/develop/extras/mediawiki/fedmsg-emit.php>`_ ::

  cd ~/vhosts/wiki.data.gouv.fr/mediawiki/extensions
  wget https://github.com/fedora-infra/fedmsg/raw/develop/extras/mediawiki/fedmsg-emit.php

Dans le fichier téléchargé, remplacer ::

  $prefix = "org.fedoraproject." . $config['environment'] . ".wiki.";

par ::

  $prefix = "fr.gouv.data." . $config['environment'] . ".wiki.";


Dans ~/vhosts/wiki.data.gouv.fr/config/LocalSettings.php, vérifier la présence des lignes ::

  # fedmsg
  require_once("$IP/extensions/fedmsg-emit.php");


Tester que l'extension fedmsg pour MediaWiki fonctionne correctement en lançant dans 2 terminaux différents ::

  fedmsg-relay

  fedmsg-tail --really-pretty

Et en modifiant une page du Wiki, un message de modification devrait apparaître dans la fenêtre du ``fedmsg-tail``.


Installation du cache HTTP/HTTPS
================================


Mise en place de Varnish pour le cache HTTP
-------------------------------------------

On a connecté le disque SSD */dev/vdb* à la machine pour le stockage du cache Varnish. On doit le préparer et le monter pour son utilisation ::

  pvcreate /dev/vdb
  vgcreate vg_ant_ssd /dev/vdb
  lvcreate -L20G -n varnish vg_ant_ssd
  mkfs.ext4 /dev/vg_ant_ssd/varnish
  tune2fs -i 0 -c 0 /dev/vg_ant_ssd/varnish
  echo "/dev/mapper/vg_ant_ssd-varnish	/var/lib/varnish	ext4	defaults	0	0" >> /etc/fstab
  mkdir /var/lib/varnish
  mount -a

Instalation ::

  aptitude install varnish
  service varnish stop

Mise en place du fichier de configuration : */etc/varnish/etalab.vcl*

Configuration du lancement du daemon *varnish* dans le fichier ``/etc/default/varnish`` ::

  DAEMON_OPTS="-a :80 \
               -T localhost:6082 \
               -f /etc/varnish/etalab.vcl \
               -S /etc/varnish/secret \
               -s memory=malloc,4G \
               -s ssd=file,/var/lib/varnish/cache,70%"

On peut ensuite lancer le service.

.. important:: Vanish ecoutera sur le port 80. Il est indispensable ce port soit disponible sur le serveur et donc qu'Apache tourne sur un autre port. Dans la configuration en place, Apache doit tourner sur le port 8080.

::

  service varnish start

Mise en place du proxying HTTPS
-------------------------------

Il a été décidé de ne pas mettre de cache sur les accès HTTPS. Cependant, le trafic pointant sur le serveur *ant* et devant être redirigé vers la machine *bat* pour le site *www.data.gouv.fr*, un proxy HTTP a été mis en place dans la configuration du serveur Apache de *ant*. Pour cela, il faut :

* Ajouter dans le fichier */etc/hosts* un enregistrement pour *www.data.gouv.fr* :

::

  echo "87.98.183.53	www.data.gouv.fr" >> /etc/hosts

* Activation du module Apache *proxy_http* ::

  a2enmod proxy_http
  service apache2 restart

* Mettre en place le fichier */etc/apache2/sites-available/www.data.gouv.fr.conf* ::

  <VirtualHost *:443>
      ServerName www.data.gouv.fr

      DocumentRoot /var/www/empty

      SSLEngine On

      <Proxy *>
          Order deny,allow
          Allow from all
      </Proxy>

      ProxyRequests Off

      ProxyPass / https://www.data.gouv.fr/ retry=2
      ProxyPassReverse / https://www.data.gouv.fr/

      SSLProxyEngine on

      ErrorLog  /var/log/apache2/www.data.gouv.fr.error.log
      CustomLog /var/log/apache2/www.data.gouv.fr.access.log combined
  </VirtualHost>

* Activer le *VirtualHost* ::

  a2ensite www.data.gouv.fr.conf
  service apache2 reload


Quelques commandes de base de Varnish
-------------------------------------


Purge d'une URL du cache
~~~~~~~~~~~~~~~~~~~~~~~~

::

  curl -X PURGE [url]

**Avec :**
  * **[url] :** l'URL à purger


Forçage du rafraichissement d'une partie du site
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Varnish dispose d'un mécanisme de *ban* permettant cela. Par exemple, pour forcer le rafraichissement de tout le répertoire ``http://www.data.gouv.fr/images/`` , utiliser la commande ::

  varnishadm
      ban req.http.host == www.data.gouv.fr && req.url ~ ^/images/.*$
