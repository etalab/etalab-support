Installation automatisée via preseed
====================================

A la fin de cette procédure vous aurez installés l'un des environement suivants :

    * Une VM de base avec debian (base)
    * Une VM de base avec debian + l'environnement de l'UI (ui)
    * Une VM de base avec debian + l'environnement de l'UI de test (ui-test)
    * Une VM de base avec debian + l'environnement de l'API (api)
    * Une VM de base avec debian + l'environnement de l'APi de test (api-test)

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

    qemu-img create -f rbd rbd:<pool>/<nom vm> <taille>

exemple ::

    qemu-img create -f rbd rbd:libvirt-sata/git 20G

**Avec :**
  - **<pool> :** Le pool Ceph a utiliser : *libvirt-ssd* pour un disque sur stockage *SSD* ou *libvirt-sata* pour un disque sur stockage *SATA*.
  - **<nom-vm> :** Nom de la VM sans espace, uniquement des caractères ascii (exemple : *test*)
  - **<taille> :** Taille du disque (Exemple : *20G*)

Storage Raw
***********
Storage utilisé dans les cas de figure ou les performances doivent optimisées au maximum. Nous n'utilisons pas ceph comme backend de stockage, ovh nous limitant en terme de performance réseau.
FIXME ::
 
   qemu-img create .....


Cas spécifique du FailOver chez ovh
***********************************
Dans le cas d'une installation avec IP FailOver (spécificité Ovh), il faut ensuite créer une adresse MAC virtuelle dans l'interface OVH. Cette adresse MAC doit être associée à l'IP Failover qui sera associé à la VM. Pour cela, il faut d'abord associer l'IP Failover au serveur physique hébergeant la VM (Dans *Accueil > Serveurs dédiés  > Services > IP Fail-Over*), puis créer une MAC virtuelle pour cette adresse IP de type *ovh* (Dans *Accueil > Serveurs dédiés > Services > Mac Virtuelle pour VPS*).

:: 

     create-virtual-machine-failover [nom-vm] [mac] [ssd]


Déclaration de la vm dans libvirt
---------------------------------
Pour faciliter les opérations, un script à été créer. 
Exemple avec une machine à créer dans le lan::

    create-virtual-machine-ip-lan <nom vm> <storage type> <vxlan number>

Help ::

    create-virtual-machine-ip-lan -h

exemple::
    
    create-virtual-machine-ip-lan git sata 10

Replication des données de libvirt
----------------------------------
Lorsque la VM nécessite de pouvoir être migrée d'un hyperviseur à l'autre en cas de problème, il faudra répliquer les informations de libvirt concernant la VM, sur tous les hyperviseurs. Cette manipulation s'effecute suivant les étapes ci-après. Dans le cas inverse, il n'est pas utile de répliquer la configuration. 

Ajout de la configuration au repository git
*******************************************
::

    cd /srv/common
    git add etc/libvirt/qemu/<nom vm>.xml
    git commit -a
    git push

Réplication des informations
****************************
A cette étape le git local est défini et poussé sur le serveur git central. Il faut maintenant mettre à jour les autres hyperviseurs. Une fois positionné sur chaque hyperviseur, on exécute la commande suivante ::
    
    cd /srv/common && git pull 


Déclaration dans libvirt
************************
De la même manière sur chaque hyperviseur, on exécute la commande suivante pour déclarer la nouvelle vm à libvirt. ::
    
    virsh define /etc/libvirt/qemu/<nom vm>.xml

Pour vérifier les informations qu'a libvirt sur les vm, on peux exécuter ::

    virsh list --all

Installation par la console
---------------------------

On lance ``virt-manager`` depuis un ordinateur afin de démarrer la VM et s'y connecter en mode console. La machine boot via PXE et lance automatiquement après 10 minutes, une installation de base. 

Pour installer un environnement particulier, on peux renseigner le paramètre de boot qui est affiché. Par exemple **ui** ou **api**. Dans chacun des cas, l'installateur exécutera un script de post installation permettant la création des environnements relatifs à votre choix.

