Installation de l'OS de base sur une  machine physique chez Ovh
===============================================================

Le but de cette procédure est d'exposer une méthode d'installation from scratch d'une machine dédié physique livrée par Ovh. On utilisera pas les outils d'ovh pour installer cette machine. La distribution choisie est Debian. 

Cette documentation prend pour exemple l'installation de la machine de backup d'etalab; un serveur de la gamme EG-32 d'ovh.

Installation à partir de la console rescue-pro d'OVH

.. important:: La documentation ci-dessous est adaptée au serveur *ns404150*. Modifiez les noms, adresses IP et autres paramètres au serveur que vous installez.

On vérifie qu'il n'y a pas de configuration d'un raid existant sur cette machine :

::
    mdadm -D /dev/md0

Si une configuration raid est présente, on la supprime.

::

  mdadm --stop /dev/md0
  mdadm --zero-superblock /dev/sda1
  mdadm --zero-superblock /dev/sda2

Si les disques ne sont pas configurés pour l'utilisation du raid, on défini les paramétrages nécessaire pour utiliser l'espace disque complet :

::

    fdisk /dev/sda
    fdisk /dev/sdb

Pour chaque disque on effectue les changements suivants :

::
    Command (m for help): n
    Select (default p): p
    Partition number (1-4, default 1): 
    Command (m for help): t
    Hex code (type L to list codes): fd
    Command (m for help): w


On créér un nouveau RAID 1 sur sda1 & sdb1 :

::

  mdadm --create /dev/md0 -l 1 -n 2 /dev/sda1 /dev/sdb1  

On configure LVM sur /dev/md0 :

::

  pvcreate /dev/md0
  vgcreate vg_ns404150 /dev/md0
  lvcreate -L4G -n root vg_ns404150
  lvcreate -L4G -n var vg_ns404150
  lvcreate -L1G -n tmp vg_ns404150
  lvcreate -L2G -n swap vg_ns404150
  mkfs.ext4 /dev/vg_ns404150/root 
  mkfs.ext2 /dev/vg_ns404150/tmp
  mkfs.ext4 /dev/vg_ns404150/var
  tune2fs -i 0 -c 0 /dev/vg_ns404150/tmp
  tune2fs -i 0 -c 0 /dev/vg_ns404150/var
  mkswap /dev/vg_ns404150/swap

On monte la nouvelle arboressence du système dans /mnt/ :

::

  mount /dev/vg_ns404150/root /mnt
  mkdir /mnt/var
  mkdir /mnt/tmp
  mount /dev/vg_ns404150/var /mnt/var/
  mount /dev/vg_ns404150/tmp /mnt/tmp

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
  /dev/mapper/vg_ns404150-root  /       ext4    errors=remount-ro,relatime      0       1
  /dev/mapper/vg_ns404150-var   /var    ext4    defaults                        0       0
  /dev/mapper/vg_ns404150-tmp   /tmp    ext2    defaults                        0       0
  /dev/mapper/vg_ns404150-swap  none    swap    defaults                        0       0

On monte le /proc et /sys :

::

  mount /proc
  mount /sys

On met en place la configuration réseau :

- /etc/network/interfaces :

::

  # This file describes the network interfaces available on your system
  # and how to activate them. For more information, see interfaces(5).
  
  # The loopback network interface
  auto lo
  iface lo inet loopback
  
  auto eth0
  iface eth0 inet static
      address 37.59.24.183
      netmask 255.255.255.0
      network 37.59.24.0
      broadcast 37.59.24.255
      gateway 37.59.24.254

- /etc/resolv.conf :

::

  nameserver 213.186.33.99
  search ovh.net

- Définition du *hostname* :

::

  echo [nom machine] > /etc/hostname

On ajoute les dépôts Debian suivant en plus de l'actuel :

::

  deb http://security.debian.org/ wheezy/updates main
  deb http://debian.data.gouv.fr/debian wheezy main
  deb http://debian.mirrors.ovh.net/debian/ wheezy-backports main

On effectue une installation de base :

::

  apt-get update
  apt-get install etalabinstall
  etalabinstall base

Remarque : Durant l'installation des paquets, laisser les choix par défaut et choisir la locale **en_US.UTF-8**

On installe un kernel livré par les backports wheezy, pour bénéfier du support de vxlan :

::

  apt-get install linux-image-3.12-0.bpo.1-amd64

Configuration de alerte mail :

::
  
  echo "root: supervision@data.gouv.fr" >> /etc/aliases
  newaliases

On installe mdadm & grub :

::

  apt-get install mdadm grub2

.. note:: choisir d'installer grub sur sda et sdb.

::

    GRUB install devices:
        [*] /dev/sda (3000592 MB; HGST_HUS724030ALA640)
        [*] /dev/sdb (3000592 MB; HGST_HUS724030ALA640)
        [ ] /dev/dm-0 (4294 MB; vg_ns404150-root)

.. note:: On modifie ensuite le paramètre rootdelay du kernel (particularité du 3.11). Pour cela il faut modifier la varaible //GRUB_CMDLINE_LINUX_DEFAULT// dans le fichier ///etc/default/grub// et mettre la valeur //"rootdelay=8"//. Il faut ensuite lancer la commande :

::

  update-grub

On reboot.
