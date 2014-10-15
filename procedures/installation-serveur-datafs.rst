***********************************
Installation du serveur de fichiers
***********************************

.. important:: L'installation se fait sur la base d'une installation vierge de Debian Wheezy.


Installation des dépendances NFS server ::

  apt-get install nfs-kernel-server nfs-common


Création et export des répertoires
==================================

Les fichiers seront stockés dans un volume logique formatté en ext4
qu'il faut créer et monter dans ``/srv/nfs/datagouv`` ::

  lvcreate -l 100%FREE -n datagouv vg00
  mkfs.ext4 /dev/vg00/datagouv
  mkdir -p /srv/nfs/datagouv
  mount -t ext4 /dev/vg00/datagouv /srv/nfs/datagouv

Pour que le volume soit automatiquement monté au démarrage, on ajoute dans /etc/fstab::

  /dev/mapper/vg00-datagouv /srv/nfs/datagouv ext4 errors=remount-ro  0    1


Les stockages seront créés avec pour utilisateur et groupe *nobody*::

  chown -R nobody:nogroup /srv/nfs/datagouv
  chmod -R 755 /srv/nfs/datagouv

Il faut ensuite exporter le répertoire en éditant le fichier ``/etc/exports``::

  /srv/nfs/datagouv   10.10.10.0/24(rw,sync,no_subtree_check)

Dans notre cas, nous exportons le répertoire ``/srv/nfs/datagouv`` en lecture-écriture pour tout le réseau ``10.10.10.0/24``

Une fois les modifications effectuées, il faut redémarrer le service ``nfs-kernel-server``::

  service nfs-kernel-server restart


Configuration des clients
=========================

Installation des dépendances NFS client ::

  apt-get install nfs-common

Creation du point de montage ::

  mkdir -p /mnt/datagouv

Pour voir la liste des répretoires exportés par un server::

  showmount -e <server>

Dans notre cas::

  showmount -e datafs

  Export list for datafs:
  /srv/nfs/datagouv 10.10.10.0/24


Chaque répertoire peut être monté manuellement avec la commande ``mount``::

  mount <server>:<export> <mountpoint>

ou automatiquement au démarrage en ajoutant la ligne suivante au fichier ``fstab``::

  <server>:<export>  <mountpoint>   nfs   rw,sync,hard,intr 0 0

puis en executant la commande::

  mount <mountpoint>

Dans notre cas::

  mount datafs:/srv/nfs/datagouv /mnt/datagouv

ou::

  datafs:/srv/nfs/datagouv /mnt/datagouv   nfs   rw,sync,hard,intr 0 0
