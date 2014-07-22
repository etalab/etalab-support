=====
Howto
=====

Backporter un package Debian testing vers Debian stable
=======================================================

On souhaite backporter un package de testing vers la stable. En l'occurance, pour notre exemple on prendra le package python-numpy pour illustrer les manipulations. 

Prérequis
---------
Forger un paquet debian, peut parfois prendre des allures de champs de bataille, surtout lorsque le paquet en question a un certain nombre de dependance. 
Pour cloisonner le travail, je vous propose d'utiliser deboostrap.

On créer un environnement ::

    apt-get install debootstrap
    debootstrap --arch amd64 wheezy ~/dbs-builddeb http://ftp.fr.debian.org/debian/

On se chroot dans le debootstrap ::

    chroot dbs-builddeb

Installer les outils de développement
-------------------------------------
Nous allons avoir besoin de quelques outils de développement, que nous installons ::

    apt-get install devscripts build-essential dh-buildinfo
    echo "export LANG=C" >> ~/.bashrc


Howto par l'exemple
===================
On configure apt dans /etc/apt/source.list, tel que ::

    ## Wheezy
    deb http://ftp.fr.debian.org/debian wheezy main
    deb-src http://ftp.fr.debian.org/debian wheezy main
    # wheezy-backports 
    deb http://ftp.fr.debian.org/debian wheezy-backports  main contrib non-free
    ## Jessie
    #deb http://ftp.fr.debian.org/debian jessie main
    deb-src http://ftp.fr.debian.org/debian jessie main

On update le tout ::

    apt-get update

On récupère les sources ::

    apt-get source python-numpy

On récupère les dependances, que l'on installe ::

    apt-get build-dep python-numpy

On compile le code source ::

    cd python-numpy-1.8.1
    dch -i

::
    python-numpy (1:1.8.1-1~etalabbpo70+1) unstable; urgency=low

    * Non-maintainer upload.
    * Backport to wheezy.

    -- Felix Defrance <felix.defrance@data.gouv.fr>  Thu, 10 Apr 2014 14:22:32 +0000

::

    dpkg-buildpackage -tc

C'est terminé ! On peut voir le package forger dans le répertoire parent. 

::

    python-numpy_1.8.1-1~etalabbpo70+1.debian.tar.gz
    python-numpy_1.8.1-1~etalabbpo70+1_amd64.deb
    python-numpy_1.8.1-1~etalabbpo70+1.dsc
    python-numpy_1.8.1-1~etalabbpo70+1_amd64.changes
    


Installation du package
=======================

Pour une utilisation personnelle un dpkg -i suffira, sinon on ajoutera le package à un dépôt spécifiquement établi pour l'occasion par exemple..
