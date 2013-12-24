*************************************
Installation du serveur de sauvegarde
*************************************

.. important:: L'installation se fait sur la base d'une installation vierge de Debian Wheezy.

Installation de **backuppc** et de ses dépendances :

::

  apt-get installl backuppc libfile-rsyncp-perl

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

Ajouter un alias mail *etalab* pour les mails de notifications des sauvegardes ::

  echo "etalab: supervision@data.gouv.fr" >> /etc/aliases
  newaliases


Mise en place du VirtualHost Apache
===================================

Mettre en place le fichier de configuation du *VirtualHost* : */etc/apache2/sites-available/backup.data.gouv.fr.conf*

Activer le *VirtualHost* :

::

  a2ensite backup.data.gouv.fr.conf


Ajout de la sauvegarde d'un serveur
===================================

.. important:: Avant de commencer, installer et configurer l'agent sur le serveur à sauvegarder.

Créer le fichier de configuration pour la sauvegarde du serveur ``/etc/backuppc/[fqdn].pl`` ::

  do "/etc/backuppc/config.pl";
  $Conf{XferMethod} = 'rsync';
  $Conf{RsyncShareName} = '/';

Donner les bons droits et propriétaires au fichier créé ::

  chown backuppc:www-data /etc/backuppc/[fqdn].pl
  chmod 0660 /etc/backuppc/[fqdn].pl

Pour ajouter l'exclusion de certain fichiers ou dossiers ajouter ::

  push ($Conf{BackupFilesExclude},'/chemin/a/exclure');

Ajouter la déclaration du serveur dans le fichier */etc/backuppc/hosts* :

::

  [fqdn]	0	etalab

Lancer une première connexion SSH au serveur à sauvegarder en tant que l'utilisateur *backuppc* ::

  su - backuppc
  ssh root@[fqdn]
  exit

.. note:: Cette connexion doit retourner une erreur de *rsync*.
