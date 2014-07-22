Procédure d'uniformisation des hyperviseurs
===========================================

Afin de assurer de l'uniformisation de certains éléments du système sur chaque hypervisieur, nous avons mis en place un prcessus de publication des configurations à l'aide de l'outil git.

Une documentation de l'outil est consultable ici :

.. include:: documentation-uniformisation-des-hyperviseurs.rst

Cas d'utilisation
-----------------
Vous avez ajouté ou modifié la configuration d'une machine virtuelle. 

Déclaration de la modification dans git
---------------------------------------
Sur l'hyperviseur si l'on a créé une VM ::

    cd /srv/common
    git status
    git add etc/libvirt/qemu/foobarvm.xml
    git commit -a  

Duplication de la configuration
-------------------------------
Puis sur chaque hyperviseur du cluster :: 
  
    cd /srv/common
    git pull

Declaration de la VM dans libvirt
---------------------------------
Une fois le fichier de configuration de la machine virtuelle en place, il faut indiquer à libvirt la prise en compte de cette modification . Cette étape s'effectue via la commande suivante::

    virsh define /etc/libvirt/qemu/foobarvm.xml


