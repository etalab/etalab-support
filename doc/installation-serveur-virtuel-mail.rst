===========================
Installation du server mail
===========================


Les détails de chaque services vont être explosés ci-après.


.. include:: include-installation-serveur-virtuel-lan.rst

Installation de base
====================

Installation des applications
=============================

Service de gestion des boites mails 
-----------------------------------

L'imap permet de..

Installation de dovecot et des services associés
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::
   
  apt-get install dovecot-common dovecot-mysql dovecot-imapd dovecot-managesieved dovecot-sieve


Configuration du service imap
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Configuration du service de filtre
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


Service de webmail
------------------

Installation du webmail sogo
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
On ajoute les dépôts pour Sogo ::

    # Sogo
    deb http://inverse.ca/debian wheezy wheezy
    deb http://ftp.fr.debian.org/debian/ wheezy-backports main contrib non-free

    apt-key adv --keyserver hkp://keys.gnupg.net:80 --recv-key 0x810273C4

    apt-get update
    apt-get install sogo


Configuration du webmail sogo
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


