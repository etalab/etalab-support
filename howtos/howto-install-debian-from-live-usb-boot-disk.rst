=====================================================
Comment installer GNU/Linux Debian depuis une clé usb
=====================================================

Dans ce how to, nous allons voir comment installer Debian sur un ordinateur classique, à partir d'une clé usb bootable. 

On va utiliser une méthode d'installation via chroot, et rsync pour installer l'OS.

.. note :: Cette clé usb bootable à été créé à partir d'un debootstrap. Une documentation se trouve ici : *howto-make-debian-live-usb-boot-disk.rst*


Une fois que vous avez démarré sur la clé usb, veuilliez suivre les indications ci-après.

Choisir le disque dur de destination
------------------------------------
On choisi le disque dur local à la machine sur lequel on souhaite installer Debian, et on repère le device en question  ::

    fdisk -l

::

	Disk /dev/sda: 250.1 GB, 250059350016 bytes
	255 heads, 63 sectors/track, 30401 cylinders, total 488397168 sectors
	Units = sectors of 1 * 512 = 512 bytes
	Sector size (logical/physical): 512 bytes / 512 bytes
	I/O size (minimum/optimal): 512 bytes / 512 bytes
	Disk identifier: 0x082a1650
	
	   Device Boot      Start         End      Blocks   Id  System
	/dev/sda1            2048   487757823   243877888    7  HPFS/NTFS/exFAT
	/dev/sda2   *   487757824   488372223      307200    7  HPFS/NTFS/exFAT


On delete les partitions existantes
-----------------------------------
::

	Command (m for help): d
	Partition number (1-4): 1
	Command (m for help): d

On créer les nouvelles partitions
---------------------------------

On crée une partition pour la SWAP ::

	Command (m for help): n
	Partition type:
	   p   primary (0 primary, 0 extended, 4 free)
	   e   extended
	Select (default p): p
	Partition number (1-4, default 1): 
	Using default value 1
	First sector (2048-488397167, default 2048): 
	Using default value 2048
	Last sector, +sectors or +size{K,M,G} (2048-488397167, default 488397167): +2G
	
	Command (m for help): t
	Partition number (1-4): 1
	Hex code (type L to list codes): 82
	Changed system type of partition 1 to 82 (Linux swap / Solaris)


Pour le ROOTFS ::

	Command (m for help): n
	Partition type:
	   p   primary (1 primary, 0 extended, 3 free)
	   e   extended
	Select (default p): p
	Partition number (1-4, default 2): 
	Using default value 2
	First sector (4196352-488397167, default 4196352): 
	Using default value 4196352
	Last sector, +sectors or +size{K,M,G} (4196352-488397167, default 488397167): 
	Using default value 488397167

On oublie pas de positionner le flag de boot sur la partition où va se trouver grub ::

    Command (m for help): a
    Partition number (1-4): 1

Le résultat de ces commandes précédentes nous donne un partitionnement de ce type ::

	Disk /dev/sda: 250.1 GB, 250059350016 bytes
	255 heads, 63 sectors/track, 30401 cylinders, total 488397168 sectors
	Units = sectors of 1 * 512 = 512 bytes
	Sector size (logical/physical): 512 bytes / 512 bytes
	I/O size (minimum/optimal): 512 bytes / 512 bytes
	Disk identifier: 0x082a1650
	
	   Device Boot      Start         End      Blocks   Id  System
	/dev/sda1   *        2048     4196351     2097152   82  Linux swap / Solaris
	/dev/sda2         4196352   488397167   242100408   83  Linux

On écrit les modifications dans la table des partitions ::
	
    Command (m for help): w
    The partition table has been altered!
    Command (m for help): quit

On créer le file system pour le SWAP ::

    mkswap /dev/sda1 
    Setting up swapspace version 1, size = 2097148 KiB
    no label, UUID=7901eff5-efd0-43a8-9425-11061ef77f45

Puis pour le ROOTFS ::

    mkfs.ext4 /dev/sda2

Préparation du chroot
=====================

On définit un répertoire dans lequel, nous allons copier les fichiers du système GNU/Linux Debian ::

    mkdir /mnt/target 
    mount /dev/sda2 /mnt/target
    mkdir -p /mnt/target/{sys,dev,proc,tmp,var}
    
Avant de  copier les données depuis la clé usb vers le disque local, on définit des exclusions ::

    vi /root/exclude.list

::

	/proc/*
	/sys/*
	/dev/*
	/etc/fstab
	/etc/mtab
	/etc/hostname
	/etc/lvm/*
	/etc/udev/*
	/etc/network/interfaces
	/etc/lvm*
	/mnt/*
	/media/*


Puis on copie le FS, sans les exclusions ::

    rsync -avz --stats --numeric-ids --progress --delete --exclude-from='/root/exclude.list' / /mnt/target/


On monte ensuite les derniers éléments nécessaire à l'installation finale ::

	mount -t proc proc /mnt/target/proc 
	mount -t sysfs sys /mnt/target/sys 
	mount -o bind /dev /mnt/target/dev

On peut maintenant se "chrooter" dans le futur OS de la machine ::

    chroot /mnt/target

Finalisation de l'installation 
==============================

Pour finaliser l'installation, on à besoin de crée une fstab, de gérer un initrd puis d'installer un boot loader sur le disque local.

On créer une fstab, en éditant le fichier suivant et en définissant son contenu ::

    vi /etc/fstab 

::

	# /etc/fstab: static file system information.
	#
	# Use 'blkid' to print the universally unique identifier for a
	# device; this may be used with UUID= as a more robust way to name devices
	# that works even if disks are added and removed. See fstab(5).
	#
	# <file system> <mount point>   <type>  <options>       <dump>  <pass>
    UUID=3b22cc07-1adb-41d7-861a-e5d725b4c67e   /       ext4    errors=remount-ro 0       1
	UUID=7901eff5-efd0-43a8-9425-11061ef77f45   none    swap    sw              0       0
	/dev/sr0        /media/cdrom0   udf,iso9660 user,noauto     0       0
	
Les UUID de disque sont à lister avec la commande suivante, on récupère les UUID correspondant au device */dev/sda** ::

    blkid

::

    /dev/sdb1: SEC_TYPE="msdos" UUID="EB67-3201" TYPE="vfat" 
	/dev/sdb2: UUID="75ef5823-06fa-4bc3-ac94-e2935c3b6609" TYPE="ext4" 
	/dev/sda1: UUID="7901eff5-efd0-43a8-9425-11061ef77f45" TYPE="swap" 
	/dev/sda2: UUID="3b22cc07-1adb-41d7-861a-e5d725b4c67e" TYPE="ext4" 

Ensuite on reconstruit l'initrd. Ce fichier permet de charger les drivers nécessaire au chargement du système, lors du démarrage de la machine ::

    update-initramfs -u

Puis on reconfigure grub ::

    dpkg-reconfigure grub-pc

Certaines options sont demandées. 
On répondra respectivement utilisant des tabulations et barre d'espace pour manipuler les éléments ::

    Ligne de commande de Linux : rien
    Ligne de commande par défaut de Linux : quiet
    Périphériques où installer GRUB : on selectionne le disque local. 

        [*] /dev/sda (250059 Mo; WDC_WD2500AAKX-603CA0)

A la suite de ces information, vous devriez avoir les outputs suivants ::

	Installation finished. No error reported.                                                                                                   
	Generating grub.cfg ...
	Found background image: /usr/share/images/desktop-base/desktop-grub.png
	Found linux image: /boot/vmlinuz-3.2.0-4-amd64
	Found initrd image: /boot/initrd.img-3.2.0-4-amd64
	done

L'installation est terminée, on peut rebooter la machine et déconnecter la clé usb bootable. 

::

    exit
    umount /mnt/target/proc
	umount /mnt/target/sys
	umount /mnt/target/dev
	umount /mnt/target
    reboot

Done.
