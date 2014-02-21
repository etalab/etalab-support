==============================
Installation du serveur bkp-01
==============================

.. include:: installation-os-de-base-serveur-dedie-ovh.rst

Configuration du système
========================

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


Configuration de backuppc
=========================
On défini une partition dédié à backuppc :

::

    lvcreate -L 200G -n backuppc vg_ns404150
    mkfs.ext4 /dev/vg_ns404150/backuppc
    tune2fs -i 0 -c 0 /dev/vg_ns404150/backuppc
    echo "/dev/mapper/vg_ns404150-backuppc /var/lib/backuppc  ext4    defaults          0       0" >> /etc/fstab
    mkdir /var/lib/backuppc
    mount -a 


Installation de **backuppc** et de ses dépendances :

::

  apt-get installl backuppc libfile-rsyncp-perl rrdtool

Pendant l'installation des paquets :

  * Choisir de ne pas configurer automatiquement Apache
  * Inutile de noter le mot de passe généré automatiquement, il sera écrasé.

Mettre en place le fichier de configuration de *BackupPC* ``/etc/backuppc/config.pl``

Supprimer le serveur ``localhost`` (sauvegardé par défaut) ::

  rm /etc/backuppc/localhost.pl
  sed -i 's/^localhost.*$//' /etc/backuppc/hosts
  rm -fr /var/lib/backuppc/pc/localhost/

Définir le mot de passe du l'utilisateur *etalab* pour l'accès à *BackupPC* ::

  htpasswd -c /etc/backuppc/htpasswd etalab

Mise en place du VirtualHost Apache
-----------------------------------

Mettre en place le fichier de configuation du *VirtualHost* : */etc/apache2/sites-available/backup.data.gouv.fr.conf*

Activer le *VirtualHost* :

::

  a2ensite backup.data.gouv.fr.conf


Ajout de la sauvegarde d'un serveur
-----------------------------------

.. important:: Avant de commencer, installer et configurer l'agent sur le serveur à sauvegarder.

Créer le fichier de configuration pour la sauvegarde du serveur ``/etc/backuppc/[fqdn].pl`` ::

  do "/etc/backuppc/config.pl";
  $Conf{XferMethod} = 'rsync';
  $Conf{RsyncShareName} = '/';

Donner les bons droits et propriétaires au fichier créé ::

  chown backuppc:www-data /etc/backuppc/[fqdn].pl
  chmod 0660 /etc/backuppc/[fqdn].pl

Pour ajouter l'exclusion de certain fichiers ou dossiers ajouter ::

    $Conf{BackupFilesExclude} = [
        # Standards Debian
        '/proc/*',
        '/sys/*',
        '/dev/pts/*',
        '/media/*',
        '/cdrom/*',
        '/mnt/*',
        '/floppy/*',
        '/tmp/*',
        # Add others Exclude files
        '/var/lib/mongodb/*',
        ];


Ajouter la déclaration du serveur dans le fichier */etc/backuppc/hosts* :

::

  [fqdn]    0   etalab

Lancer une première connexion SSH au serveur à sauvegarder en tant que l'utilisateur *backuppc* ::

  su - backuppc
  ssh root@[fqdn]
  exit

.. note:: Cette connexion doit retourner une erreur de *rsync*.


Supervision de la machine
=========================

.. include:: procedure-ajout-service-supervision.rst

Métrologie de la machine
========================

.. include:: procedure-ajout-service-metrologie.rst


