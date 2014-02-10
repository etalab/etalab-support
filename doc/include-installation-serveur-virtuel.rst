Installation automatisée via preseed
====================================
Choix de la machine hôte
------------------------
Le choix de l'hyperviseur se fait en fonction des ressources disponibles. Voici quelques commandes pour les visualiser.

Taux d'occupation de l'hyperviseur ::
  virsh list
Resource Cpu, Ram, Load Average :: 
  htop

Ensuite il faut déterminer le stockage à utiliser; rapide ou lent. Respectivement, il faut utiliser un storage basé sur disque SSD ou bien SATA. 

.. note:: Actuellement les performances réseau chez ovh étant très médiocres, cela se ressent sur les performances de CEPH. Un storage sur ssd@ceph équivaut à un disque sata attaqué directement par le système. 

Définition du storage pour la vm
--------------------------------
Une fois connecté en ``ssh`` sur l'hyperviseur défini, on peux exécuter les commandes ci-dessous.

Storage Ceph
************
::

    qemu-img create -f rbd rbd:libvirt-sata/<nom vm> 20G

ou ::

    qemu-img create -f rbd rbd:libvirt-ssd/<nom vm> 20G

exemple ::

    qemu-img create -f rbd rbd:libvirt-sata/git 20G

Storage Raw
***********
FIXME ::

    qemu-img create .....


Déclaration de la vm dans libvirt
---------------------------------
Exemple avec une machine à créer dans le lan::

    create-virtual-machine-ip-lan <nom vm> <storage type> <vxlan number>

exemple::

    create-virtual-machine-ip-lan git sata 10

Replication des données de libvirt
----------------------------------
Lorsque la VM nécessite de pouvoir être migrée d'un hyperviseur à l'autre en cas de problème, il faudra répliquer les informations de libvirt concernant la VM, sur tous les hyperviseurs. Cette manipulation s'effecute suivant les étapes ci-après. Dans le cas inverse, il n'est pas utile de répliquer la configuration. 

Ajout de la configuration au repository git
*******************************************
::

    cd /srv/common
    git add etc/libvirt/qemu/git.xml
    git commit -a
    git push

Réplication des informations
****************************
A cette étape le git local est défini et poussé sur le serveur git central. Il faut maintenant mettre à jour les autres hyperviseurs. Une fois positionné sur chaque hyperviseur, on exécute la commande suivante ::
    
    cd /src/common && git pull 


Déclaration dans libvirt
************************
De la même manière sur chaque hyperviseur, on exécute la commande suivante pour déclarer la nouvelle vm à libvirt. ::
    
    virsh define /etc/libvirt/qemu/git.xml

Pour vérifier les informations qu'a libvirt sur les vm, on peux exécuter ::

    virsh list --all

Installation par la console
---------------------------

On lance ``virt-manager`` depuis un ordinateur afin de démarrer la VM et s'y connecter en mode console. La machine boot via PXE et lance automatiquement une installation de base. 

Pour installer un environnement particulier, on peux renseigner le paramètre de boot qui est affiché. Par exemple **ui** ou **api**. Dans chacun des cas, l'installateur exécutera un script de post installation permettant la création des environnements relatifs à votre choix.



