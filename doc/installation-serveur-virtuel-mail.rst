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


Installation de Sogo
~~~~~~~~~~~~~~~~~~~~
Pour l'installation de sogo, nous allons suivre les étapes ci dessous. En plus de sogo lui même, on installera également les dépendance nécessaire.

Ajout du dépôt fourni par l'éditeur Sogo ::

    # Sogo
    deb http://inverse.ca/debian wheezy wheezy
    deb http://ftp.fr.debian.org/debian/ wheezy-backports main contrib non-free

    apt-key adv --keyserver hkp://keys.gnupg.net:80 --recv-key 0x810273C4

On met à jour apt et on installe les packages nécessaires ::

    apt-get update
    apt-get install sogo sope4.9-gdl1-mysql memcached apache2

    
    /usr/share/doc/tmpreaper/README.security.gz
    sed -i 's/SHOWWARNING=true/SHOWWARNING=false/' /etc/tmpreaper.conf

Configuration du webmail sogo
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Configuration de sogo sur la db de postfixadmin
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::
    USE postfixadmin;
    
    CREATE VIEW  `sogo_users` AS SELECT local_part AS c_uid, username AS c_name, 
    PASSWORD AS c_password, name AS c_cn, username AS mail, domain
    FROM  `mailbox`;

    SHOW FULL TABLES IN postfixadmin WHERE TABLE_TYPE LIKE 'VIEW';
    SELECT * FROM sogo_users;

    CREATE USER 'sogo'@'%' IDENTIFIED BY 'fooboar';
    GRANT SELECT ON postfixadmin.sogo_users TO 'sogo'@'%' IDENTIFIED BY 'foobar_password';

    CREATE DATABASE `sogo` CHARACTER SET='utf8';
    GRANT ALL PRIVILEGES ON `sogo`.* TO 'sogo'@'%' WITH GRANT OPTION;
    FLUSH PRIVILEGES;

Configuration d'apache
~~~~~~~~~~~~~~~~~~~~~~

  a2enmod headers proxy_http proxy

