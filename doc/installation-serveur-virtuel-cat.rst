***********************************
Installation du serveur virtuel cat
***********************************

Dans la console web OVH, associer l'adresse IP failover 87.98.165.65 à cat et lui donner l'adresse MAC 02:00:00:e4:27:1f 

Aller sur le serveur physique wolf.data.gouv.fr, en tant que root ::

  qemu-img create -f rbd rbd:libvirt-ssd/cat 50G
  create-virtual-machine-ip-failover cat 02:00:00:e4:27:1f ssd
    Check name : OK
    Check RBD image : rbd image 'cat':
	    size 51200 MB in 12800 objects
	    order 22 (4096 kB objects)
	    block_name_prefix: rb.0.43e8.2ae8944a
	    format: 1
    Create VM :
    Domain cat defined from /tmp/tmp.XDk0nPrt7Y

    Id:             -
    Name:           cat
    UUID:           48193767-646b-4a05-bc37-e39c94ce23ba
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

Dans Virtual Machine Manager, lancer cat et lancer l'installation ::

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
    cat
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
          address 87.98.165.65
          netmask 255.255.255.255
          broadcast 87.98.165.65
          post-up route add 178.33.238.28 dev eth0
          post-up route add default gw 178.33.238.28 dev eth0
          post-down route del 178.33.238.28 dev eth0
          post-down route del default gw 178.33.238.28

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

Quitter maintenant ``virt-manager`` et lancer une connexion ssh ``ssh root@87.98.165.65``, puis ::

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
  127.0.1.1       cat

en ::

  127.0.1.1       cat.data.gouv.fr cat

Créer le fichier ``/etc/apt/apt.conf.d/50norecommends`` pour y mettre la ligne ::

  APT::Install-Recommends "false";

Installer les paquets manquants ::

  aptitude install git
  aptitude install htop
  aptitude install less
  aptitude install molly-guard
  aptitude install ntp
  aptitude install sshguard


Revenir sur le serveur physique, en tant que root, puis ::

  virsh dumpxml cat > /tmp/cat.xml
  scp /tmp/cat.xml 192.168.0.1:/tmp/
  ssh 192.168.0.1 "virsh define '/tmp/cat.xml'"
  scp /tmp/cat.xml 192.168.0.2:/tmp/
  ssh 192.168.0.2 "virsh define '/tmp/cat.xml'"


Installation de postfix
=======================

Installer et configurer Postfix ::

  aptitude purge exim4 exim4-base exim4-config exim4-daemon-light postfix+
    General type of mail configuration:
      Internet Site
    System mail name:
      cat.data.gouv.fr
    Root and postmaster mail recipient:
      etalab
    Other destinations to accept mail for (blank for none):
      cat.data.gouv.fr, localhost.localdomain, localhost
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

  myhostname = cat

en ::

  myhostname = cat.data.gouv.fr

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
  aptitude install python-twisted
  aptitude install python-tz

Installer fedmsg ::

  pip install fedmsg
  pip install requests

Modifier le fichier ``/etc/fedmsg.d/base.py`` ::

  environment = 'prod',
  topic_prefix = 'fr.gouv.data',

Dans ``/etc/fedmsg.d/endpoints.py``, commenter les endpoints existants et les remplacer par ::

  "data-gouv-fr-infrastructure": [
      "tcp://ant.data.gouv.fr:9940",
      "tcp://bat.data.gouv.fr:9940",
  ],

Dans ``/etc/fedmsg.d/ssl.py``, supprimer la signature des messages ::

  validate_signatures=False,

Tester que fedmsg fonctionne correctement en lançant dans 3 terminaux différents ::

  fedmsg-relay

  fedmsg-tail --really-pretty

  echo "Hello, world" | fedmsg-logger


Installation de qa.data.gouv.fr
===============================

::

  aptitude install libapache2-mod-wsgi
  aptitude install mongodb-server
  aptitude install python-babel
  aptitude install python-html5lib
  aptitude install python-isodate
  aptitude install python-mako
  aptitude install python-pymongo
  aptitude install python-pymongo-ext
  # aptitude install python-tz
  aptitude install python-yaml
  aptitude install python-weberror
  aptitude install python-webob

  pip install bleach
  pip install cssmin
  pip install markdown
  # pip install requests

  # Install a customized version of webassets that corrects UnicodeDecodeErrors.
  # pip install webassets
  pip install git+https://github.com/noirbizarre/webassets.git@for-weckan#egg=webassets

  npm install -g bower

En tant qu'etalab ::

  cd
  git clone https://github.com/etalab/biryani.git

En tant que root ::

  cd /home/etalab/biryani
  python setup.py develop --no-deps
  python setup.py compile_catalog

En tant qu'etalab ::

  cd
  git clone https://github.com/etalab/ckan-toolbox.git

En tant que root ::

  cd /home/etalab/ckan-toolbox
  python setup.py develop --no-deps

En tant qu'etalab ::

  cd
  git clone https://github.com/etalab/ckan-of-worms.git

En tant que root ::

  cd /home/etalab/ckan-of-worms
  python setup.py develop --no-deps

En tant qu'etalab ::

  cd ~/ckan-of-worms
  python setup.py compile_catalog
  bower install
  ./setup.py build_assets

.. note:: Si la compilation échoue à cause d'une erreur d'encodage, utiliser les fichiers présents sur un PC local où le build_assets a réussi.

En tant qu'etalab ::

  cd
  mkdir repositories
  cd repositories/
  git init --bare qa.data.gouv.fr.git
  cd
  mkdir vhosts
  cd vhosts/
  git clone ../repositories/qa.data.gouv.fr.git

  cd ~/ckan-of-worms
  ./ckanofworms/scripts/setup_app.py -v ~/vhosts/qa.data.gouv.fr/config/paste.ini


En tant que root ::

  cd /home/etalab/vhosts/qa.data.gouv.fr/
  mkdir cache
  chown www-data. cache

  cd /etc/apache2/sites-available/
  ln -s  /home/etalab/vhosts/qa.data.gouv.fr/config/apache2.conf qa.data.gouv.fr.conf
  cd ../sites-enabled/
  rm 000-default
  a2ensite qa.data.gouv.fr.conf
  service apache2 restart


Recopie du contenu de CKAN dans QA
----------------------------------

Avec l'interface web, créer un compte dans ``qa.data.gouv.fr``, le rendre administrateur et récupérer son API key pour la donner à tous les bots (champ ``ckan_of_worms.api_key`` dans ``circus-fedmsg.cat.data.gouv.fr/etalabbot.ini``).

En tant qu'etalab ::

  cd ../ckan-of-worms/
  ./ckanofworms/scripts/harvest_ckan.py -a -v ~/circus-fedmsg.cat.data.gouv.fr/etalabbot.ini 


Installation de ws.data.gouv.fr
===============================

::

  aptitude install python-gevent

Regarder les paquets nécessaires pour chaussette & ws4py ::

  pip install --no-install chaussette ws4py

Installer chaussette & ws4py ::

  pip install chaussette ws4py

En tant qu'etalab ::

  cd
  git clone https://github.com/etalab/dactylo.git

En tant que root ::

  cd /home/etalab/dactylo
  python setup.py develop --no-deps

En tant qu'etalab ::

  cd ~/dactylo
  python setup.py compile_catalog
  bower install
  ./setup.py build_assets

.. note:: Si la compilation échoue à cause d'une erreur d'encodage, utiliser les fichiers présents sur un PC local où le build_assets a réussi.

En tant qu'etalab ::

  cd ~/repositories/
  git init --bare ws.data.gouv.fr.git
  cd
  cd vhosts/
  git clone ../repositories/ws.data.gouv.fr.git
  cd ws.data.gouv.fr/
  mkdir ipc

  cd ~/dactylo/
  ./dactylo/scripts/setup_app.py -v ~/vhosts/ws.data.gouv.fr/config/paste.ini

En tant que root ::

  cd /home/etalab/vhosts/ws.data.gouv.fr/
  mkdir cache
  chown www-data. cache

  cd /etc/apache2/sites-available/
  ln -s  /home/etalab/vhosts/ws.data.gouv.fr/config/apache2.conf ws.data.gouv.fr.conf
  a2ensite ws.data.gouv.fr.conf
  service apache2 restart

  cd /var/log/
  mkdir circus-ws

  cd /etc/logrotate.d/
  ln -s /home/etalab/vhosts/ws.data.gouv.fr/config/circus-ws.logrotate circus-ws

  cd /etc/init.d/
  ln -s /home/etalab/vhosts/ws.data.gouv.fr/config/circus-ws.init circus-ws
  update-rc.d circus-ws defaults
  service circus-ws restart

Avec l'interface web, créer un compte dans ``ws.data.gouv.fr``, le rendre administrateur et récupérer son API key pour la donner à tous les bots (champ ``dactylo.api_key`` dans ``circus-fedmsg.cat.data.gouv.fr/etalabbot.ini``)dans ``circus-fedmsg.cat.data.gouv.fr/etalabbot.ini``).


Installation de CowBots
=======================

En tant que root ::

  pip install python-twitter
  pip install requests-oauthlib

En tant qu'etalab ::

  cd
  git clone https://github.com/etalab/cowbots.git


Installation de circus-fedmsg
=============================

Regarder les paquets nécessaires pour circus ::

  pip install --no-install circus

Installer circus ::

  pip install circus

En tant qu'etalab ::

  cd ~/repositories/
  git init --bare circus-fedmsg.cat.data.gouv.fr.git
  cd ..
  git clone repositories/circus-fedmsg.cat.data.gouv.fr.git/
  cd circus-fedmsg.cat.data.gouv.fr
  mkdir ipc

En tant que root ::

  cd /var/log/
  mkdir circus-fedmsg

  cd /etc/logrotate.d/
  ln -s /home/etalab/circus-fedmsg.cat.data.gouv.fr/circus-fedmsg.logrotate circus-fedmsg

  cd /etc/init.d/
  ln -s /home/etalab/circus-fedmsg.cat.data.gouv.fr/circus-fedmsg.init circus-fedmsg
  update-rc.d circus-fedmsg defaults
  service circus-fedmsg restart

Tester que fedmsg fonctionne correctement en lançant dans 2 terminaux différents ::

  fedmsg-tail --really-pretty

  echo "Hello, world" | fedmsg-logger


Envoi de toutes les données présentes dans QA par fedmsg
--------------------------------------------------------

En tant qu'etalab ::

  cd ~/ckan-of-worms/
  ./ckanofworms/scripts/publish_fedmsg_updates.py -a -v ~/vhosts/qa.data.gouv.fr/config/paste.ini


Envoi de tous les jeux de données de QA par fedmsg
--------------------------------------------------

  cd cowbots
  ./check_datasets.py -v ~/circus-fedmsg.cat.data.gouv.fr/etalabbot.ini


Administration de circus-fedmsg
===============================

En tant que root ::

  circusctl --endpoint ipc:///home/etalab/circus-fedmsg.cat.data.gouv.fr/ipc/circus-endpoint.ipc


Installation de id.data.gouv.fr
===============================

::

  aptitude install postgresql
  aptitude install postgresql-server-dev-all
  aptitude install python-virtualenv
  aptitude install redis-server

  npm install -g less yuglify

  cd /var/log/
  mkdir youckan
  chmod go+wx youckan/

En tant qu'etalab ::

  cd
  tar xzf youckan-0.1.0.dev.4992473.tar.gz
  mv youckan-0.1.0.dev.4992473 youckan
  cd youckan/

  cd ~/repositories/
  git init --bare data.gouv.fr-certificates.git
  git init --bare id.data.gouv.fr.git
  cd
  git clone repositories/data.gouv.fr-certificates.git/
  cd vhosts/
  git clone ../repositories/id.data.gouv.fr.git
  cd id.data.gouv.fr/
  mkdir ipc
  virtualenv .
  source bin/activate
  pip install ../../youckan-0.1.0.dev.c5d867d.tar.gz

En tant que root ::

  cd /home/etalab/vhosts/id.data.gouv.fr/
  mkdir media
  chown www-data. media/

  su - postgres
  createuser youckan -P
    Enter password for new role: 
    Enter it again: 
    Shall the new role be a superuser? (y/n) n
    Shall the new role be allowed to create databases? (y/n) n
    Shall the new role be allowed to create more new roles? (y/n) n
  createdb youckan -O youckan -E UTF8

Revenir en tant qu'etalab ::

  # youckan genconf --ini
  #   Domain [youckan.com]: data.gouv.fr
  #   Public hostname [www.youckan.com]: id.data.gouv.fr
  #   Log directory [/var/log/youckan]: 
  #   Creating apache.conf
  #   Creating nginx.conf
  #   Creating youckan.wsgi
  #   Creating youckan.ini

  youckan init --noinput

En tant que root ::

  addgroup etalab www-data
  chown www-data. /var/log/youckan/id.data.gouv.fr.django.logs

  cd /etc/apache2/sites-available/
  ln -s  /home/etalab/vhosts/id.data.gouv.fr/apache.conf id.data.gouv.fr.conf
  cd ../sites-enabled/
  a2ensite id.data.gouv.fr.conf

  a2enmod ssl

Éditer le fichier ``/etc/apache2/ports.conf`` pour y ajouter la ligne ci-dessous dans chacun des blocs SSL ::

  NameVirtualHost *:443

En tant que root ::

  service apache2 restart

  cd /var/log/
  mkdir circus-id

  cd /etc/logrotate.d/
  ln -s /home/etalab/vhosts/id.data.gouv.fr/circus-id.logrotate circus-id

  cd /etc/init.d/
  ln -s /home/etalab/vhosts/id.data.gouv.fr/circus-id.init circus-id
  update-rc.d circus-id defaults
  service circus-id restart


Mise à jour de YouCKAN
======================

En tant qu'etalab ::

  cd vhosts/id.data.gouv.fr/
  source bin/activate
  pip install ../../youckan-0.1.0.dev.c5d867d.tar.gz
  youckan init --noinput

En tant que root ::

  service apache2 force-reload


Installation de static.data.gouv.fr
===================================

En tant qu'etalab ::

  cd ~/repositories/
  git init --bare static.data.gouv.fr.git
  cd
  cd vhosts/
  git clone ../repositories/static.data.gouv.fr.git

En tant que root ::

  cd /etc/apache2/sites-available/
  ln -s  /home/etalab/vhosts/static.data.gouv.fr/config/apache2.conf static.data.gouv.fr.conf
  a2ensite static.data.gouv.fr.conf
  service apache2 restart


Installation du virtual host apache par défaut
==============================================

En tant qu'etalab ::

  git clone https://github.com/etalab/landing-page.git

  cd ~/repositories/
  git init --bare _default_.data.gouv.fr.git
  cd
  cd vhosts/
  git clone ../repositories/_default_.data.gouv.fr.git

En tant que root ::

  cd /etc/apache2/sites-available/
  ln -s  /home/etalab/vhosts/_default_.data.gouv.fr/config/apache2.conf _default_.data.gouv.fr.conf
  cd ../sites-enabled/
  ln -s ../sites-available/_default_.data.gouv.fr.conf 000-_default_.data.gouv.fr.conf
  service apache2 restart

