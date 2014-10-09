***********************************
Installation du serveur de fichiers
***********************************

.. important:: L'installation se fait sur la base d'une installation vierge de Debian Wheezy.

Installation des dépendances NFS server ::

  apt-get install nfs-kernel-server nfs-common


Création et export des répertoires
==================================

Pour suivre la LSB, les stockages seront créés dans ``/srv/nfs``,
avec pour utilisateur et groupe *nobody*::

  mkdir -p /srv/nfs/data.gouv.fr
  chown nobody:nogroup /srv/nfs/data.gouv.fr
  chmod 755 /srv/nfs/data.gouv.fr

Il faut ensuite exporter le répertoire en éditant le fichier ``/etc/exports``::

  /srv/nfs/data.gouv.fr   10.10.10.0/24(rw,sync,no_subtree_check)

Dans notre cas, nous exportons le répertoire ``/srv/nfs/data.gouv.fr`` en lecture-écriture pour tout le réseau ``10.10.10.0/24``

Une fois les modifications effectués, il faut redémarrer le service ``nfs-kernel-server``::

  service nfs-kernel-server restart


Configuration des clients
=========================

Installation des dépendances NFS client ::

  apt-get install nfs-common

Creation du point de montage ::

  mkdir -p /mnt/data.gouv.fr

Pour voir la liste des répretoires exportés par un server::

  showmount -e <server>

Dans notre cas::

  showmount -e datafs

  Export list for datafs:
  /srv/nfs/data.gouv.fr 10.10.10.0/24


Chaque répertoire peut être monté manuellement avec la commande ``mount``::

  mount <server>:<export> <mountpoint>

ou automatiquement au démarrage en ajoutant la ligne suivante au fichier ``fstab``::

  <server>:<export>  <mountpoint>   nfs   rw,sync,hard,intr 0 0

puis en executant la commande::

  mount <mountpoint>

Dans notre cas::

  mount datafs:/srv/nfs/data.gouv.fr /mnt/data.gouv.fr

ou::

  datafs:/srv/nfs/data.gouv.fr /mnt/data.gouv.fr   nfs   rw,sync,hard,intr 0 0
