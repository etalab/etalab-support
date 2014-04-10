Commandes usuelles
==================

Les commandes ci-après permettent de lister de manière non exhaustives, les actions qui vous seront nécessaire à la gestion des VM sur les hyperviseurs. 

A chaque manipulation, ses commandes spécifiques. 

    * qemu-img
    * virsh
    * rdb
    * ceph


qemu-img
--------
Commandes pour créer des images virtuelles de machines. Dans cette exemple on crée une image au format "rbd" ::

  qemu-img create -f rbd rbd:[pool]/[nom-vm] [taille] 

.. note:: Il existe un script d'automatisation reprenant la création d'une vm. Se référer à la documentation **include-installation-serveur-virtuel.rst**


virsh
-----
Les commandes virsh permettent de contrôler les VM.  C'est l'outil sousjacent à virt-manager. 

:: 

  virsh list                : Affiche les machines virtuelles en fonctionnement
  virsh list --all          : Affiche toutes les machines virtuelles
  virsh start [nom-vm]      : Démarre une machine virtuelle
  virsh shutdown [nom-vm]   : Arrête une machine virtuelle
  virsh destroy [nom-vm]    : Arrêt forcé (=coupure de courant) d'une machine virtuelle
  virsh edit [nom-vm]       : Permet d'editer la configuration d'une vm. 
                              (La plupart des modifcations necessite un reboot pour qu'elles soient prises en compte)
  virsh suspend [nom-vm]    : Mettre une vm en pause
  virsh resume [nom-vm]     : Reactiver une vm en pause

Connexion a la console
**********************
Avec un vnc standard on peut se connecter a une vm. 

1/ On cherche le port vnc sur l'hyperviseur ::

    ps aux | grep [nom-vm] | grep vnc --color

Exemple de resultat  :: 

    [...] -vnc 127.0.0.1:1 [....]
    Le port d'écoute sera 5901.

2/ On etablie un tunnel ssh vers ce port ::
    
    ssh -L 5901:localhost:5901 root@[hyperviseur]

3/ Connexion avec un client vnc ::
    
   vncviewer -compresslevel 9  -bgr233 localhost:5901


Exemple de modification de resource Ram et Cpu
**********************************************
Respectivement, pour modifier la mémoire allouée à une machine virtuelle, on édite sa configuration et on change les valeurs, en octet, pour ** <memory> ** (mémoire maximum) et ** <currentMemory> ** (mémoire allouée, peut être inférieure à <memory>). 
L'augmentation de la mémoire allouée nécessite un redémarrage de la machine virtuelle.

Modifier le nombre de cœur dans ** <vcpu> ** . Un redémarrage de la machine virtuelle est nécessaire pour que cette modification soit prise en compte.

Migration d'une vm d'un hyperviseur à un autre
**********************************************
Cette opération peut affecter les performances réseau du cluster ceph lorsque la taille des VM est importante. Il faudra donc porter une attention particulière lors des migrations.

Avant la migration, il faut executer la commande ci-dessous sur l'OS de la VM :: 

    echo tsc > /sys/devices/system/clocksource/clocksource0/current_clocksource
   
Ensuite, on execute sur l'hyperviseur la commande suivante ::

    virsh migrate --live [nom-vm] qemu+ssh://root@[hostname hyperviseur destination]/system tcp://[IP hyperviseur destination]

Exemple ::
    
    virsh migrate --live api-02 qemu+ssh://root@ns235977/system tcp://192.168.0.2


rbd
---
Les commandes rbd permettent de gérer les images disques issue de Ceph. ::

    rados lspools                                   : afficher les pools existants
    rbd list [pool]                                 : afficher les images contenu dans les pools
    rbd info [pool]/[nom-vm]                        : afficher les information plus détaillées de l'image
    rbd rm [pool]/[nom-vm]                          : supprimer une image de vm.
    rbd resize --size [taille] [pool]/[nom-vm]      : redimensionnement d'une image (la taille est en MB) (Opération à faire à froid!)
    rbd snap list                                   : afficher les snapshots existants
    rbd snap create [pool]/[nom-vm]@[nom snapshot]  : Créer un snapshot de la vm
    rbd snap rm [pool]/[nom-vm]@[nom snapshot]      : Supression du snapshot défini.
    rbd snap rollback [pool]/[nom-vm]@[nom snapshot]: Revenir à un snapshot défini.

Ceph
----
Les commandes ceph qui permettent d'avoir des informations sur l'état du cluster ceph ::

  ceph -w       : affiche l'état du cluster ceph et affiche les evenements en court 

D'autres commandes utiles ::

  ceph -s                          : affiche l'état du cluster ceph
  ceph osd tree                    : affiche l'arboresence les osd en fonction des nodes ceph
  ceph osd repair osd.[numero-osd] : lance une operation de reparation sur l'osd spécifié 

Afficher les informations d'états indépendements ::

  ceph osd stat : affiche l'état des osd.
  ceph mon stat : affiche l'état des monitor et du quorum.
  ceph mds stat : affiche l'état des metadata du cluster ceph
  ceph pg stat  : afficher l'état des placement group du cluster ceph


Démarrer un osd
***************
::

  /etc/init.d/ceph start osd.[numero-osd]


  
Redémarrage d'un hyperviseur
****************************
Il faut commencer par déplacer les VM tournant sur cet hyperviseur sur un autre. Pour cela assurez-vous tout d'abord que l'ensemble des VM de cet hypversiveur pourront tourner sans problème sur le second hyperviseur notamment en terme de mémoire vive disponible. Au besoin, réduiser temporairement la mémoire allouée aux VMs.

Une fois cette vérification faite, suivre les commandes défini plus haut "Migration d'une vm d'un hyperviseur à un autre".

Une fois que l'hyperviseur ne fait plus tourner aucune VM, exectuer la commande suivant pour eviter une resynchronisation inutile durant l'indisponibilité de l'hyperviseur :

::
  
  ceph osd set noout

Vous pouvez maintenant redémmarer la machine. Au reboot repasser le service en mode normal :

::
  
  ceph osd unset noout

