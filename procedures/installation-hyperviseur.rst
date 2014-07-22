====================================
Installation d'un hyperviseur Etalab 
====================================

Installation de l'OS d'un hyperviseur à partir de la console rescue-pro d'OVH
-----------------------------------------------------------------------------

.. important:: La documentation ci-dessous est adaptée au serveur *ns235977*. Modifiez les noms, adresses IP et autres paramètres au serveur que vous installez.

On supprime toute trace de l'ancien RAID :

::

  mdadm --stop /dev/md0
  mdadm --stop /dev/md1
  mdadm --zero-superblock /dev/sda1
  mdadm --zero-superblock /dev/sda2
  mdadm --zero-superblock /dev/sdb1
  mdadm --zero-superblock /dev/sdb2

On repartionne sda & sdb :
- 80G en type fd (RAID Logiciel)
- le reste en xfs

On créé un nouveau RAID 1 sur sda1 & sdb1 :

::

  mdadm --create /dev/md0 -l 1 -n 2 /dev/sda1 /dev/sdb1

On met du LVM sur /dev/md0 :

::

  pvcreate /dev/md0
  vgcreate vg_ns235977 /dev/md0
  lvcreate -L4G -n root vg_ns235977
  lvcreate -L5G -n var vg_ns235977
  lvcreate -L1G -n tmp vg_ns235977
  lvcreate -L2G -n swap vg_ns235977
  mkfs.ext4 /dev/vg_ns235977/root 
  mkfs.ext4 /dev/vg_ns235977/tmp
  mkfs.ext4 /dev/vg_ns235977/var
  tune2fs  -i 0 -c 0 /dev/vg_ns235977/root
  tune2fs  -i 0 -c 0 /dev/vg_ns235977/tmp
  tune2fs  -i 0 -c 0 /dev/vg_ns235977/var
  mkswap /dev/vg_ns235977/swap

On monte la nouvelle arboressence du système dans /mnt/ :

::

  mount /dev/vg_ns235977/root /mnt
  mkdir /mnt/var
  mkdir /mnt/tmp
  mount /dev/vg_ns235977/var /mnt/var/
  mount /dev/vg_ns235977/tmp /mnt/tmp

On lance un debbootstrap dans /mnt/ :

::

   debootstrap --arch=amd64 wheezy /mnt/ http://debian.mirrors.ovh.net/debian/

On allimente le /dev du chroot :

::

  rsync -av /dev/ /mnt/dev/

On entre dans le chroot :

::

  chroot /mnt/

On défini le mot de passe root

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
  /dev/mapper/vg_ns235977-root  /       ext4    errors=remount-ro,relatime      0       1
  /dev/mapper/vg_ns235977-var   /var    ext4    defaults                        0       0
  /dev/mapper/vg_ns235977-tmp   /tmp    ext4    defaults                        0       0
  /dev/mapper/vg_ns235977-swap  none    swap    defaults                        0       0

On monte le /proc et /sys :

::

  mount /proc
  mount /sys

On met en place la configuration réseau :

- Ajouter le bloc suivant dans le fichier */etc/hosts* :

::
  
  192.168.0.1	ns235513
  192.168.0.2	ns235977
  192.168.0.3	ns236004

- /etc/network/interfaces :

::

  # This file describes the network interfaces available on your system
  # and how to activate them. For more information, see interfaces(5).
  
  # The loopback network interface
  auto lo
  iface lo inet loopback
  
  auto br0
  iface br0 inet static
  	address 178.33.237.236
  	netmask 255.255.255.0
  	network 178.33.237.0
  	broadcast 178.33.237.255
  	gateway 178.33.237.254
  	bridge_ports eth0
  	bridge_maxwait 0
  	bridge_stp off
  	bridge_fd 0
  
  auto br1
  iface br1 inet static
  	address 192.168.0.2
  	netmask 255.255.255.0
  	network 192.168.0.0
  	broadcast 192.168.0.255
  	bridge_ports eth1
  	bridge_maxwait 0
  	bridge_stp off
  	bridge_fd 0

- /etc/resolv.conf :

::

  nameserver 213.186.33.99
  search ovh.net

- Définition du *hostname* :

::

  hostname ns235977.ovh.net
  echo ns235977.ovh.net > /etc/hostname

On ajoute les dépôts Debian suivant en plus de l'actuel :

::

  deb http://security.debian.org/ wheezy/updates main
  deb http://debian.easter-eggs.org/debian wheezy main libvirt kvm
  deb http://ftp.fr.debian.org/debian wheezy-backports main contrib non-free

On effectue une installation de base :

::

  apt-get update
  apt-get install eeinstall
  eeinstall base

Remarque : Durant l'installation des paquets, laisser les choix par défaut et choisir la locale **en_US.UTF-8**

On install un kernel :

::

  apt-get install linux-image-3.12-0.bpo.1-amd64

Configuration de alerte mail :

::
  
  echo "root: supervision@data.gouv.fr" >> /etc/aliases
  newaliases

On install mdadm & grub :

::

  apt-get install mdadm grub2

Remarque : choisir d'installer grub sur sda et sdb.

On modifie ensuite le paramètre rootdelay du kernel (particularité du 3.11). Pour cela il faut modifier la varaible //GRUB_CMDLINE_LINUX_DEFAULT// dans le fichier ///etc/default/grub// et mettre la valeur //"rootdelay=8"//. Il faut ensuite lancer la commande :

::

  update-grub

Standardisation des hyperviseurs via git
----------------------------------------

On configure une interface temporaire vxlan10 pour récupérer la config git-common ::

  /opt/iproute2/ip link add vxlan10 type vxlan id 10 group 239.0.0.10 ttl 4 dev br1
  /opt/iproute2/ip addr add 10.10.10.x/24 broadcast 10.10.10.255 dev vxlan10
  /opt/iproute2/ip link set dev vxlan10 up

On clone le dépot git hyp-common ::

  git clone git@git.intra.data.gouv.fr:hyp-common /srv/common

On exécute le script qui permet de déplacer la configuration des confs ::

  cd /srv/common
  ./checkOrInstall
  ./checkOrInstall install


Configuration des hyperviseurs une fois l'installation de l'OS fait
===================================================================


Ajout d'un utilisateur etalab
-----------------------------

::
  
  adduser etalab
  adduser etalab libvirt

**Remarque :** Pour la connexion SSH via une clé avec cette utilisateur, la clé doit être mise dans le fichier */etc/ssh/authorized_keys/etalab*.


Suppression de l'authentification par mot de passe dans SSH
-----------------------------------------------------------

Dans le fichier /etc/ssh/sshd_config, ajouter la ligne ::

  PasswordAuthentication no 


Installation du pare-feu
------------------------

Mettre en place les fichiers suivant (commun à tout les hyperviseurs) :

- **packetfilter** dans */etc/init.d/*
- **etalab.conf** dans */etc/*

**Remarque :** les droits de ces fichiers doivent être *0750*.

Il faut ensuite activer le service au démarrage :

::
  
  insserv packetfilter

Arrêt/démarrage du parefeu
~~~~~~~~~~~~~~~~~~~~~~~~~~

Démarrage :

::
  
  service packetfilter start

Arrêt :

::
  
  service packetfilter stop

Status :

::
  
  service packetfilter status

Ajout d'une IP FailOver au parefeu
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Les IPs FailOver doit être incluses autorisés par le pare-feu. Pour cela, il suffit d'éditer le fichier */etc/etalab.conf* et d'ajouter l'IP FailOver dans la variable *IPS_FAILOVER*. Par la suite, il faudra relancer *packetfilter* pour que la modification soit prise en compte.

.. important:: Toutes modifications du parefeu doivent être déployées sur les autres hyperviseurs.


Configuration de l'authentification SSH entre les hyperviseurs
--------------------------------------------------------------

Générer une clé SSH **sans-passphrase** sur chaque hyp : 

::

  ssh-keygen -t rsa

Modifier l'emplacement de stockage des clés SSH :

::

  sed -i 's/^#AuthorizedKeysFile.*$/AuthorizedKeysFile \/etc\/ssh\/authorized_keys\/%u/' /etc/ssh/sshd_config
  mkdir /etc/ssh/authorized_keys

Réunir les clés publique de toutes machines et les mettre dans le fichier ///etc/ssh/authorized_keys/root// (//cat /root/.ssh/id_rsa.pub// pour afficher la clé d'un hyperviseur)

Redémarrer SSH :

::

  /etc/init.d/ssh restart

Connecter une fois sur chaque hyperviseur depuis chaque hyperviseur (y compris eux même) :

::

  ssh root@192.168.0.1
  ssh root@192.168.0.2
  ssh root@192.168.0.3
  ssh root@192.168.0.4


Installation de Ceph
--------------------

.. note:: L'installation d'un hyperviseur, ne necessite pas forcément l'installation de ceph. Dans ce cas, installer ceph-common puis passer a la prochaine étape.

::

  echo "deb http://ceph.com/debian-dumpling/ wheezy main" > /etc/apt/sources.list.d/ceph.list
  gpg --keyserver pgpkeys.mit.edu --recv-key 7EBFDD5D17ED316D
  gpg -a --export 7EBFDD5D17ED316D|apt-key add -
  apt-get update
  lvcreate -nceph -L30G vg_`hostname -s`
  mkfs.xfs -n size=64k /dev/vg_`hostname -s`/ceph
  echo "/dev/mapper/vg_$( hostname -s )-ceph /var/lib/ceph xfs rw,noexec,nodev,noatime,nodiratime,inode64 0 0" >> /etc/fstab
  mount -a
  apt-get install ceph
  mkdir /var/lib/ceph/osd/ceph-0 -p
  mkdir /var/lib/ceph/osd/ceph-1 -p
  mkdir /var/lib/ceph/osd/ceph-2 -p
  mkdir /var/lib/ceph/osd/ceph-3 -p
  mkfs.xfs -f -n size=64k /dev/sda2
  mkfs.xfs -f -n size=64k /dev/sdb2
  mkfs.xfs -f -n size=64k /dev/sdc
  mkfs.xfs -f -n size=64k /dev/sdd
  echo "/dev/sda2 /var/lib/ceph/osd/ceph-0 xfs rw,noexec,nodev,noatime,nodiratime,inode64 0 0" >> /etc/fstab
  echo "/dev/sdb2 /var/lib/ceph/osd/ceph-1 xfs rw,noexec,nodev,noatime,nodiratime,inode64 0 0" >> /etc/fstab
  echo "/dev/sdc  /var/lib/ceph/osd/ceph-2 xfs rw,noexec,nodev,noatime,nodiratime,inode64 0 0" >> /etc/fstab
  echo "/dev/sdd  /var/lib/ceph/osd/ceph-3 xfs rw,noexec,nodev,noatime,nodiratime,inode64 0 0" >> /etc/fstab
  mount -a

Remarques :

 - répondre *yes* a la question de savoir si on accepte le nouvelle clé d'autorité de certification
 - L'ID des OSD doit être unique sur l'ensemble du cluster (ceph-X)

Configuration de Ceph
---------------------

- Mettre en place le fichier */etc/ceph/ceph.conf* sur les 3 serveurs

Configuration des monitors
~~~~~~~~~~~~~~~~~~~~~~~~~~

- Sur **ns235513** :

::

  mkdir -p /var/lib/ceph/mon/ceph-a
  ceph-authtool --create-keyring  /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin
  ceph-authtool --create-keyring /var/lib/ceph/mon/ceph-a/keyring --gen-key -n mon.
  cp -a /var/lib/ceph/mon/ceph-a/keyring /etc/ceph/ceph.mon.a.keyring
  chmod 600 /etc/ceph/ceph.client.admin.keyring
  cat /etc/ceph/ceph.client.admin.keyring >> /var/lib/ceph/mon/ceph-a/keyring
  ceph-authtool /var/lib/ceph/mon/ceph-a/keyring -n client.admin --cap mds 'allow' --cap osd 'allow *' --cap mon 'allow *'
  ceph-mon -i a -f -c /etc/ceph/ceph.conf --mkfs

- Sur **ns235977** :

::

  mkdir -p /var/lib/ceph/mon/ceph-b
  scp 192.168.0.1:/var/lib/ceph/mon/ceph-a/keyring /var/lib/ceph/mon/ceph-b/keyring
  scp 192.168.0.1:/etc/ceph/ceph.mon.a.keyring /etc/ceph/ceph.mon.b.keyring
  scp 192.168.0.1:/etc/ceph/ceph.client.admin.keyring /etc/ceph/ceph.client.admin.keyring
  chmod 600 /etc/ceph/ceph.client.admin.keyring
  ceph-mon -i b -f -c /etc/ceph/ceph.conf --mkfs

- Sur **ns236004** :

::

  mkdir -p /var/lib/ceph/mon/ceph-c
  scp 192.168.0.1:/var/lib/ceph/mon/ceph-a/keyring /var/lib/ceph/mon/ceph-c/keyring
  scp 192.168.0.1:/etc/ceph/ceph.mon.a.keyring /etc/ceph/ceph.mon.c.keyring
  scp 192.168.0.1:/etc/ceph/ceph.client.admin.keyring /etc/ceph/ceph.client.admin.keyring
  chmod 600 /etc/ceph/ceph.client.admin.keyring
  ceph-mon -i c -f -c /etc/ceph/ceph.conf --mkfs

- Sur les trois serveurs : 

::

  /etc/init.d/ceph -a start mon

Configuration des OSDs
~~~~~~~~~~~~~~~~~~~~~~

- Sur **ns235513** :

::

  mkdir /var/lib/ceph/journal/
  ceph osd create
  ceph-osd -i 0 --mkfs --mkkey
  ceph auth add osd.0 osd 'allow *' mon 'allow rwx' -i /var/lib/ceph/osd/ceph-0/keyring
  service ceph -a start osd.0
  ceph osd crush set 0 2.0 root=default datacenter=rbx host=ns235513
  
  ceph osd create
  ceph-osd -i 1 --mkfs --mkkey
  ceph auth add osd.1 osd 'allow *' mon 'allow rwx' -i /var/lib/ceph/osd/ceph-1/keyring
  service ceph -a start osd.1  
  ceph osd crush set 1 2.0 root=default datacenter=rbx host=ns235513

  ceph osd create
  ceph-osd -i 2 --mkfs --mkkey
  ceph auth add osd.2 osd 'allow *' mon 'allow rwx' -i /var/lib/ceph/osd/ceph-2/keyring
  service ceph -a start osd.2
  ceph osd crush set 2 2.0 root=default datacenter=rbx host=ns235513
  
  ceph osd create
  ceph-osd -i 3 --mkfs --mkkey
  ceph auth add osd.3 osd 'allow *' mon 'allow rwx' -i /var/lib/ceph/osd/ceph-3/keyring
  service ceph -a start osd.3  
  ceph osd crush set 3 2.0 root=default datacenter=rbx host=ns235513

- Sur **ns235977** :

::

  mkdir /var/lib/ceph/journal/
  ceph osd create
  ceph-osd -i 4 --mkfs --mkkey
  ceph auth add osd.4 osd 'allow *' mon 'allow rwx' -i /var/lib/ceph/osd/ceph-4/keyring
  service ceph -a start osd.4
  ceph osd crush set 4 2.0 root=default datacenter=rbx host=ns235977
  
  ceph osd create
  ceph-osd -i 5 --mkfs --mkkey
  ceph auth add osd.5 osd 'allow *' mon 'allow rwx' -i /var/lib/ceph/osd/ceph-5/keyring
  service ceph -a start osd.5
  ceph osd crush set 5 2.0 root=default datacenter=rbx host=ns235977

  ceph osd create
  ceph-osd -i 6 --mkfs --mkkey
  ceph auth add osd.6 osd 'allow *' mon 'allow rwx' -i /var/lib/ceph/osd/ceph-6/keyring
  service ceph -a start osd.6
  ceph osd crush set 6 2.0 root=default datacenter=rbx host=ns235977

  ceph osd create
  ceph-osd -i 7 --mkfs --mkkey
  ceph auth add osd.7 osd 'allow *' mon 'allow rwx' -i /var/lib/ceph/osd/ceph-7/keyring
  service ceph -a start osd.7
  ceph osd crush set 7 2.0 root=default datacenter=rbx host=ns235977

- Sur **ns236004** :

::

  mkdir /var/lib/ceph/journal/
  ceph osd create
  ceph-osd -i 8 --mkfs --mkkey
  ceph auth add osd.8 osd 'allow *' mon 'allow rwx' -i /var/lib/ceph/osd/ceph-8/keyring
  service ceph -a start osd.8
  ceph osd crush set 8 2.0 root=default datacenter=rbx host=ns236004
  
  ceph osd create
  ceph-osd -i 9 --mkfs --mkkey
  ceph auth add osd.9 osd 'allow *' mon 'allow rwx' -i /var/lib/ceph/osd/ceph-9/keyring
  service ceph -a start osd.9
  ceph osd crush set 9 2.0 root=default datacenter=rbx host=ns236004

  ceph osd create
  ceph-osd -i 10 --mkfs --mkkey
  ceph auth add osd.10 osd 'allow *' mon 'allow rwx' -i /var/lib/ceph/osd/ceph-10/keyring
  service ceph -a start osd.10
  ceph osd crush set 10 2.0 root=default datacenter=rbx host=ns236004

  ceph osd create
  ceph-osd -i 11 --mkfs --mkkey
  ceph auth add osd.11 osd 'allow *' mon 'allow rwx' -i /var/lib/ceph/osd/ceph-11/keyring
  service ceph -a start osd.11
  ceph osd crush set 11 2.0 root=default datacenter=rbx host=ns236004

Configuration de la CRUSH map
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

On met dans un fichier temporaire (exemple : */tmp/crush.txt*) la CRUSH Map :

::

  # begin crush map
  
  # devices
  device 0 osd.0
  device 1 osd.1
  device 2 osd.2
  device 3 osd.3
  device 4 osd.4
  device 5 osd.5
  device 6 osd.6
  device 7 osd.7
  device 8 osd.8
  device 9 osd.9
  device 10 osd.10
  device 11 osd.11
  
  # types
  type 0 osd
  type 1 host
  type 2 datacenter
  type 3 root
  
  # buckets
  
  host ns235513-ssd {
          id -1
          alg straw
          hash 0
          item osd.0 weight 2.000
          item osd.1 weight 2.000
  }
  host ns235513-sata {
          id -2
          alg straw
          hash 0
          item osd.2 weight 2.000
          item osd.3 weight 2.000
  }
  host ns235977-ssd {
          id -3
          alg straw
          hash 0
          item osd.4 weight 2.000
          item osd.5 weight 2.000
  }
  host ns235977-sata {
          id -4
          alg straw
          hash 0
          item osd.6 weight 2.000
          item osd.7 weight 2.000
  }
  host ns236004-ssd {
          id -5
          alg straw
          hash 0
          item osd.8 weight 2.000
          item osd.9 weight 2.000
  }
  host ns236004-sata {
          id -6
          alg straw
          hash 0
          item osd.10 weight 2.000
          item osd.11 weight 2.000
  }
  root sata {
          id -7           # do not change unnecessarily
          # weight 24.000
          alg straw
          hash 0  # rjenkins1
          item ns235513-sata weight 8.000
          item ns235977-sata weight 8.000
          item ns236004-sata weight 8.000
  }
  root ssd {
          id -8           # do not change unnecessarily
          # weight 24.000
          alg straw
          hash 0  # rjenkins1
          item ns235513-ssd weight 8.000
          item ns235977-ssd weight 8.000
          item ns236004-ssd weight 8.000
  }
  # rules
  rule data {
          ruleset 0
          type replicated
          min_size 1
          max_size 10
          step take sata
          step chooseleaf firstn 0 type host
          step emit
  }
  rule metadata {
          ruleset 1
          type replicated
          min_size 1
          max_size 10
          step take sata
          step chooseleaf firstn 0 type host
          step emit
  }
  rule rbd {
          ruleset 2
          type replicated
          min_size 1
          max_size 10
          step take sata
          step chooseleaf firstn 0 type host
          step emit
  }
  rule ssd {
          ruleset 3
          type replicated
          min_size 1
          max_size 10
          step take ssd
          step chooseleaf firstn 0 type host
          step emit
  }
  # end crush map

On compile ensuite cette CRUSH Map :

::
  
  crushtool -c /tmp/crush.txt -o /tmp/crush

On install ensuite cette nouvelle CRUSH Map :

::
  
  ceph osd setcrushmap -i /tmp/crush

On créé ensuite deux *pool ceph* pour *libvirt*, un pour le stockage *SSD* et un pour le stockage *SATA* :

::

  ceph osd pool create libvirt-ssd 200
  ceph osd pool set libvirt-ssd crush_ruleset 3
  ceph osd pool create libvirt-sata 200

**Remarque :** Le nombre *200* correspond au nombre de *Placement Group* calculé selon la méthode officielle expliquée ici : http://ceph.com/docs/master/rados/operations/placement-groups/

On défini ensuite un niveau de réplication à 3 pour tous les pools :

::

  ceph osd pool set data size 3
  ceph osd pool set metadata size 3
  ceph osd pool set rbd size 3
  ceph osd pool set libvirt-ssd size 3
  ceph osd pool set libvirt-sata size 3


Installation de libvirt
-----------------------

::

  lvcreate -L 5G -n libvirt vg_$( hostname -s )
  mkfs.ext4 /dev/vg_$( hostname -s )/libvirt 
  tune2fs -i0 -c0 /dev/vg_$( hostname -s )/libvirt
  echo "/dev/mapper/vg_$( hostname -s )-libvirt /var/lib/libvirt ext4    defaults             0       0" >> /etc/fstab
  mkdir /var/lib/libvirt
  mount -a
  apt-get install libvirt-bin qemu-kvm netcat-openbsd qemu-utils



Configuration de Libvirt pour utiliser Ceph
-------------------------------------------

- Création d'un utilisateur dédié pour libvirt au niveau de ceph (sur **ns235513**) :

::

  ceph auth get-or-create client.libvirt mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=libvirt-ssd, allow rwx pool=libvirt-sata'


- Configuration d'un *secret* au niveau de libvirt pour stocker les informations d'authentification auprès de ceph (sur **ns235513**) :

::

  echo "<secret ephemeral='no' private='no'><uuid>`uuidgen`</uuid><usage type='ceph'><name>client.libvirt secret</name></usage></secret>" > /tmp/secret.xml
  scp /tmp/secret.xml 192.168.0.2:/tmp/
  scp /tmp/secret.xml 192.168.0.3:/tmp/
  virsh secret-define /tmp/secret.xml
  ssh 192.168.0.2 "virsh secret-define /tmp/secret.xml"
  ssh 192.168.0.3 "virsh secret-define /tmp/secret.xml"

- Définition du *secret* (sur **ns235513**) : On commance par récupèré l'UUID du secret libvirt affiché lors de la création du secret à l'étape précedente :

::

  Secret 9b*******************************27e created


- On récupère la clé de l'utilisateur *ceph* *client.libvirt* au format *base64* :

::

  root@ns235513:~# ceph auth get client.libvirt
  [client.libvirt]
  	key = AQ**********************************0A==
  	caps mon = "allow r"
  	caps osd = "allow class-read object_prefix rbd_children, allow rwx pool=libvirt-ssd, allow rwx pool=libvirt-sata"
 
- On peut maintenant définir à partir des deux informations récupérées :

::

  virsh secret-set-value --secret 9b*******************************27e --base64 'AQ**********************************0A=='
  ssh 192.168.0.2 "virsh secret-set-value --secret 9b*******************************27e --base64 'AQ**********************************0A=='"
  ssh 192.168.0.3 "virsh secret-set-value --secret 9b*******************************27e --base64 'AQ**********************************0A=='"

Mise en place des fichiers locaux
---------------------------------

Installer les scripts suivants :

  - *create-virtual-machine* dans */usr/local/sbin/*
  - *generate_mac* dans */usr/loca/bin/*

Télécharger l'ISO Debian qui sera utilisée pour l'installation des VMs :

::

  wget -O /var/lib/libvirt/images/debian-7.2.0-amd64-netinst.iso http://cdimage.debian.org/debian-cd/7.2.0/amd64/iso-cd/debian-7.2.0-amd64-netinst.iso



Installation de la configuration réseau des vlans
-------------------------------------------------

On fixe les Vxlan dans la configuration de l'hyperviseur ::

    # Lan area
	auto vxlan10
	iface vxlan10 inet manual
	        pre-up /opt/iproute2/ip link add vxlan10 type vxlan id 10 group 239.0.0.10 ttl 4 dev br1
	        post-down /opt/iproute2/ip link del vxlan10
	
	auto br10
	iface br10 inet static
	        address 10.10.10.5
	        netmask 255.255.255.0
	        network 10.10.10.0
	        broadcast 10.10.10.255
	        bridge_ports vxlan10
	        bridge_maxwait 0
	        bridge_stp off
	        bridge_fd 0
	
	# Api area
	auto vxlan11
	iface vxlan11 inet manual
	        pre-up /opt/iproute2/ip link add vxlan11 type vxlan id 11 group 239.0.0.11 ttl 4 dev br1
	        post-down /opt/iproute2/ip link del vxlan11
	
	auto br11
	iface br11 inet static
	        address 10.10.11.5
	        netmask 255.255.255.0
	        network 10.10.11.0
	        broadcast 10.10.11.255
	        bridge_ports vxlan11
	        bridge_maxwait 0
	        bridge_stp off
	        bridge_fd 0


On reboot la machine. 
