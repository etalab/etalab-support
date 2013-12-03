***********************************
Installation du serveur virtuel ant
***********************************

Dans la console web OVH, associer l'adresse IP failover 87.98.177.42 à ant et lui donner l'adresse MAC 02:00:00:99:59:b6

Aller sur le serveur physique turtle.data.gouv.fr, en tant que root ::

  qemu-img create -f rbd rbd:libvirt-sata/ant 20G
  create-virtual-machine ant 02:00:00:99:59:b6
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
  deb-src http://security.debian.org/ wheezy/updates main

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

Indexer la base et mettre à jour Postfix ::

  newaliases
  service postfix reload

Installer SpamAssassin ::

  aptitude install spamassassin

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

