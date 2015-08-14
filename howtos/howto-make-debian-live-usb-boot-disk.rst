Comment installer une clé usb bootable GNU/Linux Debian "from scratch"
======================================================================

Dans ce howto, je décris comment fabriquer une clé usb bootable à partir de Debian. 
La méthode d'installation s'articule autour de l'utilitaire de boot *syslinux* et de l'installeur Debian *debootstrap*.

Pré-requis
----------

Vous avez besoin d'installer les outils suivants ::

    apt-get install syslinux debootstrap

Ensuite nous avons besoin d'indentifier le device relatif à votre clé usb ::

    grep -Rli usb /sys/block/*/device/model |  awk -F "/" '{print $4}'

Dans ce howto, la clé usb est détecté par le kernel linux comme étant le device */dev/sdc*. On se connecte dessus avec fdisk ::

    fdisk /dev/sdc

::

	Commande (m pour l'aide): p
	
	Disque /dev/sdc : 16.0 Go, 16030629888 octets
	81 têtes, 32 secteurs/piste, 12079 cylindres, total 31309824 secteurs
	Unités = secteurs de 1 * 512 = 512 octets
	Taille de secteur (logique / physique) : 512 octets / 512 octets
	taille d'E/S (minimale / optimale) : 512 octets / 512 octets
	Identifiant de disque : 0x222b4ffd
	
	Périphérique Amorce  Début        Fin      Blocs     Id  Système
	/dev/sdc1   *        2048       70000       33976+   6  FAT16
	/dev/sdc2           70001    31309823    15619911+  83  Linux


:: 

    - /dev/sdc1 est la partition de boot de la clé
    - /dev/sdc2 est la partition utilisable pour stocker des fichiers.

Installation du système sur la clé usb
--------------------------------------
Afin de se chrooter dans le futur système, on prépare le chroot.
On créer le file system et on monte les partitions ::

    mkdosfs /dev/sdc1
    mkfs.ext4 /dev/sdc2
    mkdir /mnt/usb; mkdir /mnt/usb-boot
    mount /dev/sdc1 /mnt/usb-boot
    mount /dev/sdc2 /mnt/usb

On installe Debian via debootstrap ::

	debootstrap --arch amd64 wheezy /mnt/usb http://ftp.fr.debian.org/debian/

On finalise le chroot ::
	
	mount -t devtmpfs dev /mnt/usb/dev/
	mount -t devpts devpts /mnt/usb/dev/pts
	mount -t sysfs sys /mnt/usb/sys
	mount -t proc proc /mnt/usb/proc

Puis on se chroot ::

	chroot /mnt/usb

On installe un kernel ainsi qu'un bootloader ::
    
    apt-get install linux-image-amd64 grub2

.. note:: Au moment de l'installation de grub2, l'installeur demande de selectionner le disque de boot. On prendra soin de coche le device de la clé usb. Dans mon cas celui-ci sera "/dev/sdc".

On ajoute les dépôts génériques de debian ::

    echo "deb http://ftp.fr.debian.org/debian wheezy main contrib non-free" > /etc/apt/sources.list
    echo "deb http://security.debian.org/ wheezy/updates main contrib non-free" >> /etc/apt/sources.list

On installe quelques packages nécessaires  ::

    apt-get update; apt-get install console-data keyboard-configuration vim htop locales pciutils binutils bind9-host openssh-server firmware-linux-nonfree

On définit un mot de passe root ::

    passwd

On sort du chroot ::
  
    exit

On copie le kernel et l'initrd dans la partition de boot de la clé usb ::

    cp /mnt/usb/boot/vmlinuz-3.2.0-4-amd64 /mnt/usb-boot/vmlinuz
    cp /mnt/usb/boot/initrd.img-3.2.0-4-amd64 /mnt/usb-boot/initrd.gz

C'est terminé on démonte les partitions proprement ::

    umount /mnt/usb/dev/pts
    umount /mnt/usb/dev/
    umount /mnt/usb/sys
    umount /mnt/usb/proc
    umount /mnt/usb
    umount /mnt/usb-boot

Done.
