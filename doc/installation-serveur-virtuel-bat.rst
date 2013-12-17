***********************************
Installation du serveur virtuel bat
***********************************

Dans la console web OVH, associer l'adresse IP failover 87.98.183.53 à bat et lui donner l'adresse MAC 02:00:00:59:96:9e

Aller sur le serveur physique viper.data.gouv.fr, en tant que root ::

  qemu-img create -f rbd rbd:libvirt-ssd/bat 50G
  create-virtual-machine-failover bat 02:00:00:59:96:9e ssd
    Check name : OK
    Check RBD image : rbd image 'bat':
	    size 51200 MB in 12800 objects
	    order 22 (4096 kB objects)
	    block_name_prefix: rb.0.3a50.238e1f29
	    format: 1
    Create VM :
    Domain bat defined from /tmp/tmp.ftfok4wZAA

    Id:             -
    Name:           bat
    UUID:           abbec1e6-5ea9-464b-a069-ec924d3dca03
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

Dans Virtual Machine Manager, lancer bat et lancer l'installation ::

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
    bat
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
          address 87.98.183.53
          netmask 255.255.255.255
          broadcast 87.98.183.53
          post-up route add 178.33.237.236 dev eth0
          post-up route add default gw 178.33.237.236 dev eth0
          post-down route del 178.33.237.236 dev eth0
          post-down route del default gw 178.33.237.236

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

Quitter maintenant ``virt-manager`` et lancer une connexion ssh ``ssh root@87.98.183.53``, puis ::

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
  127.0.1.1       bat

en ::

  127.0.1.1       bat.data.gouv.fr bat

Créer le fichier ``/etc/apt/apt.conf.d/50norecommends`` pour y mettre la ligne ::

  APT::Install-Recommends "false";

Installer les paquets manquants ::

  aptitude install htop
  aptitude install less
  aptitude install molly-guard
  aptitude install ntp
  aptitude install sshguard


Revenir sur le serveur physique, en tant que root, puis ::

  virsh dumpxml bat > /tmp/bat.xml
  scp /tmp/bat.xml 192.168.0.1:/tmp/
  ssh 192.168.0.1 "virsh define '/tmp/bat.xml'"
  scp /tmp/bat.xml 192.168.0.3:/tmp/
  ssh 192.168.0.3 "virsh define '/tmp/bat.xml'"


Installation de postfix
=======================

Installer et configurer Postfix ::

  aptitude purge exim4 exim4-base exim4-config exim4-daemon-light postfix+
    General type of mail configuration:
      Internet Site
    System mail name:
      bat.data.gouv.fr
    Root and postmaster mail recipient:
      etalab
    Other destinations to accept mail for (blank for none):
      bat.data.gouv.fr, localhost.localdomain, localhost
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

  myhostname = bat

en ::

  myhostname = bat.data.gouv.fr

Éditer le fichier ``/etc/aliases`` pour y ajouter ::

  axel: axel@haustant.fr
  emmanuel: emmanuel@raviart.com
  etalab: axel,emmanuel

Indexer la base et mettre à jour Postfix ::

  newaliases
  service postfix reload


Installation de node.js
=======================

::

  aptitude install build-essential
  aptitude install checkinstall

En tant qu'etalab ::

  cd
  mkdir node.js
  cd node.js
  mdir src
  cd src
  wget -N http://nodejs.org/dist/node-latest.tar.gz
  tar xzvf node-latest.tar.gz && cd node-v*
  ./configure
  checkinstall #(remove the "v" in front of the version number in the dialog)

En tant que root ::

  dpkg -i /home/etalab/node.js/node-v0.10.23/node_*.deb
  npm install -g bower less uglify-js


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
  git init --bare circus-fedmsg.bat.data.gouv.fr.git
  cd ..
  git clone repositories/circus-fedmsg.bat.data.gouv.fr.git/
  cd circus-fedmsg.bat.data.gouv.fr
  mkdir ipc

En tant que root ::

  cd /var/log/
  mkdir circus-fedmsg

  cd /etc/logrotate.d/
  ln -s /home/etalab/circus-fedmsg.bat.data.gouv.fr/circus-fedmsg.logrotate circus-fedmsg

  cd /etc/init.d/
  ln -s /home/etalab/circus-fedmsg.bat.data.gouv.fr/circus-fedmsg.init circus-fedmsg
  update-rc.d circus-fedmsg defaults
  service circus-fedmsg restart

Tester que fedmsg fonctionne correctement en lançant dans 2 terminaux différents ::

  fedmsg-tail --really-pretty

  echo "Hello, world" | fedmsg-logger


Installation de CKAN
====================

::

  aptitude install bzip2
  aptitude install git
  aptitude install ca-certificates
  aptitude install gcc
  aptitude install g++
  # aptitude install python-dev
  # aptitude install python-pip
  aptitude install python-virtualenv
  aptitude install postgresql-9.1
  aptitude install postgresql-server-dev-9.1
  aptitude install solr-jetty
  aptitude install openjdk-6-jdk
  aptitude install libapache2-mod-wsgi
  aptitude install libzmq-dev

En tant qu'etalab ::

  cd ~
  mkdir -p ckan/default
  virtualenv ckan/default
  source ckan/default/bin/activate

  cd ckan
  pip install -e 'git+https://github.com/etalab/ckan.git@release-v2.1.1#egg=ckan'
  pip install -r default/src/ckan/requirements.txt

Réactive le virtualenv pour s'assurer qu'il est à jour ::

  deactivate
  source default/bin/activate

En tant que root ::

  su - postgres
  createuser -S -D -R -P ckan_default
    Enter password for new role: XXXX
  createdb -O ckan_default ckan_default -E utf-8
  createuser -S -D -R -P -l datastore_default
    Enter password for new role: XXXX
  createdb -O ckan_default datastore_default -E utf-8

  psql -d ckan_default < /home/etalab/ckan_default-20131213.dump
  psql -d datastore_default < /home/etalab/datastore_default-20131213.dump
  CTRL-D

En tant qu'etalab (toujours dans le virtualenv "default") ::

  cd ~

  git clone https://github.com/etalab/biryani.git
  cd biryani
  git checkout biryani1
  python setup.py develop --no-deps
  python setup.py compile_catalog
  cd ..

  git clone https://github.com/etalab/ckanext-etalab.git
  cd ckanext-etalab
  python setup.py develop --no-deps
  cd ..

  git clone https://github.com/etalab/ckanext-fedmsg.git
  cd ckanext-fedmsg
  python setup.py develop --no-deps
  cd ..

  git clone https://github.com/etalab/ckanext-youckan.git
  cd ckanext-youckan
  python setup.py develop --no-deps
  cd ..

  pip install git+https://github.com/noirbizarre/webassets.git@for-weckan#egg=webassets
  pip install bleach
  pip install cssmin
  pip install futures
  pip install PyYAML
  pip install wtforms

  git clone https://github.com/etalab/weckan.git
  cd weckan
  python setup.py develop --no-deps
  python setup.py compile_catalog
  bower install
  ./setup.py build_assets
  cd ..

Réinstaller fedmsg dans le virtualenv ::

  pip install fedmsg

En tant que root, éditer le fichier ``/etc/default/jetty`` pour y mettre les lignes ::

  NO_START=0
  JETTY_HOST=127.0.0.1
  JETTY_PORT=8983

Dans ``/var/lib/jetty/webapps``, corriger le lien de solr, qui est faux ::

  cd /var/lib/jetty/webapps
  rm solr
  ln -s /usr/share/solr/web solr

Changer le schema pour utiliser celui de CKAN ::

  cd /etc/solr/conf/
  mv schema.xml schema.xml.orig
  ln -s /home/etalab/ckanext-etalab/ckanext/etalab/public/schema.xml

Puis lancer jetty ::

  service jetty start

Tester son bon fonctionnement, en tant qu'etalab ::

  wget http://localhost:8983/
  wget http://localhost:8983/solr/

Revenir en tant qu'etalab ::

  cd ~/repositories/
  git init --bare data.gouv.fr-certificates.git
  git init --bare www.data.gouv.fr.git
  cd
  git clone repositories/data.gouv.fr-certificates.git/

Sur le PC personnel, pusher le projet wwww.data.gouv.fr dans ce dépôt, puis ::

  mkdir vhosts
  cd vhosts/
  git clone ../repositories/www.data.gouv.fr.git

En tant que root ::

  cd /home/etalab/vhosts/www.data.gouv.fr/
  mkdir storage
  chown www-data. storage

Éditer le fichier ``/etc/apache2/ports.conf`` pour y ajouter la ligne ci-dessous dans chacun des blocs SSL ::

  NameVirtualHost *:443

En tant que root ::

  a2enmod ssl

  cd /etc/apache2/sites-available/
  ln -s /home/etalab/vhosts/www.data.gouv.fr/config/apache2.conf www.data.gouv.fr.conf
  cd ../sites-enabled/
  a2ensite www.data.gouv.fr.conf
  rm 000-default

Réindexation de solr
--------------------

En tant qu'etalab ::

  cd ~/ckan
  source default/bin/activate
  time paster --plugin=ckan search-index rebuild -r --config=../vhosts/www.data.gouv.fr/config/production.ini

Installation d'etalab-ckan-scripts
==================================

  cd
  git clone https://github.com/etalab/etalab-ckan-scripts.git


Installation de la landing page
===============================

En tant qu'etalab ::

  git clone https://github.com/etalab/landing-page.git

