***********************************
Installation du serveur virtuel dog
***********************************

Aller sur le serveur physique turtle.data.gouv.fr, en tant que root ::

  qemu-img create -f rbd rbd:libvirt-sata/dog 20G
  create-virtual-machine-ip-ripe dog
    Check name : OK
    Check RBD image : rbd image 'dog':
	    size 20480 MB in 5120 objects
	    order 22 (4096 kB objects)
	    block_name_prefix: rb.0.1e7fa.238e1f29
	    format: 1
    Create VM :
    Domain dog defined from /tmp/tmp.Cz1LapVRn5

    Id:             -
    Name:           dog
    UUID:           6f7df283-857d-40a4-94b6-2b9463b7b5c7
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

Dans Virtual Machine Manager, lancer dog et lancer l'installation ::

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
    Configure network manually
  IP address:
    178.33.214.177
  Netmask:
    255.255.255.240
  Gateway:
    178.33.214.190
  Name server addresses:
    213.186.33.99
  Hostname:
    dog
  Domain name:
    data.gouv.fr
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
  Debian archive mirror country:
    France
  Debian archive mirror:
    debian.mirrors.ovh.net
  Participate in the package user survey?
    No
  Choose software to install:
    [*] SSH server
    [*] Standard system utilities
  Install the GRUB boot loader to the master boot record?
    Yes

Quitter maintenant ``virt-manager`` et lancer une connexion ssh ``ssh root@dog.data.gouv.fr``, puis ::

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

Créer le fichier ``/etc/apt/preferences`` ::

  # Testing
  Package: *
  Pin: release a=testing
  Pin-Priority: 900

  # Unstable aka Sid
  Package: *
  Pin: release a=unstable
  Pin-Priority: 300

  # Experimental
  Package: *
  Pin: release a=experimental
  Pin-Priority: 200

Éditer le fichier ``/etc/apt/sources.list`` ::

  deb http://debian.mirrors.ovh.net/debian/ testing main
  # deb-src http://debian.mirrors.ovh.net/debian/ testing main

  deb http://security.debian.org/ testing/updates main
  # deb-src http://security.debian.org/ testing/updates main

  deb http://debian.mirrors.ovh.net/debian/ unstable main
  # deb-src http://debian.mirrors.ovh.net/debian/ unstable main

  deb http://debian.mirrors.ovh.net/debian/ experimental main
  # deb-src http://debian.mirrors.ovh.net/debian/ experimental main

Créer le fichier ``/etc/apt/apt.conf.d/50norecommends`` pour y mettre la ligne ::

  APT::Install-Recommends "false";

Installer les paquets manquants ::

  aptitude install apache2
  aptitude install git
  aptitude install htop
  aptitude install molly-guard
  aptitude install ntp
  aptitude install rsync
  aptitude install sshguard


Revenir sur le serveur physique, en tant que root, puis ::

  virsh dumpxml dog > /tmp/dog.xml
  scp /tmp/dog.xml 192.168.0.2:/tmp/
  ssh 192.168.0.2 "virsh define '/tmp/dog.xml'"
  scp /tmp/dog.xml 192.168.0.3:/tmp/
  ssh 192.168.0.3 "virsh define '/tmp/dog.xml'"


Installation de postfix
=======================

Installer et configurer Postfix ::

  aptitude purge exim4 exim4-base exim4-config exim4-daemon-light postfix+
    General type of mail configuration:
      Internet Site
    System mail name:
      dog.data.gouv.fr
    Root and postmaster mail recipient:
      etalab
    Other destinations to accept mail for (blank for none):
      dog.data.gouv.fr, localhost.data.gouv.fr, localhost
    Force synchronous updates on mail queue?
      No
    Local networks:
      127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
    Use procmail for local delivery?
      Yes
    Mailbox size limit (bytes):
      0
    Local address extension character:
      +
    Internet protocols to use:
      ipv4

Éditer le fichier ``/etc/aliases`` pour y ajouter ::

  axel: axel@haustant.fr
  emmanuel: emmanuel@raviart.com
  etalab: axel,emmanuel

Indexer la base et mettre à jour Postfix ::

  newaliases
  service postfix reload


Installation de api.openfisca.fr
================================

::

  aptitude install libapache2-mod-wsgi
  aptitude install python-babel
  aptitude install python-isodate
  aptitude install python-pandas
  aptitude install python-setuptools
  aptitude install python-tz
  aptitude install python-weberror
  aptitude install python-webob

En tant qu'etalab ::

  cd
  git clone https://github.com/etalab/biryani.git
  cd biryani
  python setup.py compile_catalog

En tant que root ::

  cd /home/etalab/biryani
  python setup.py develop --no-deps

En tant qu'etalab ::

  cd
  git clone https://github.com/openfisca/openfisca-core.git
  cd openfisca-core
  ./setup.py compile_catalog

En tant que root ::

  cd /home/etalab/openfisca-core
  python setup.py develop --no-deps

En tant qu'etalab ::

  cd
  git clone https://github.com/openfisca/openfisca-france.git
  # cd openfisca-france
  # ./setup.py compile_catalog

En tant que root ::

  cd /home/etalab/openfisca-france
  python setup.py develop --no-deps

En tant qu'etalab ::

  cd
  git clone https://github.com/openfisca/openfisca-web-api.git
  cd openfisca-web-api
  ./setup.py compile_catalog

En tant que root ::

  cd /home/etalab/openfisca-web-api
  python setup.py develop --no-deps

En tant qu'etalab ::

  cd
  mkdir repositories
  cd repositories/
  git init --bare api.openfisca.fr.git
  cd
  mkdir vhosts
  cd vhosts/
  git clone ../repositories/api.openfisca.fr.git

En tant que root ::

  cd /etc/apache2/sites-available/
  ln -s  /home/etalab/vhosts/api.openfisca.fr/config/apache2.conf api.openfisca.fr.conf
  cd ../sites-enabled/
  rm 000-default
  a2ensite api.openfisca.fr.conf
  service apache2 restart
