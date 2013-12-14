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
