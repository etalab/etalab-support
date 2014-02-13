Installation d'une vm dans le lan
=================================

Les machines présentes dans le Lan sont celles qui par opposition ne se trouvent pas dans la ferme de calcul prévue pour accueillir les vm API.

L'adressage de ce réseau Lan est le suivant ::
    
    10.10.10.0/24

La passerelle de ce réseau est la suivante ::

    10.10.10.254 ou gw.intra.data.gouv.fr

Le dns interne est le suivant ::
    
    10.10.10.10 ou ns1.intra.data.gouv.fr

Machine virtuelle de base
-------------------------

Pour configurer une vm de base. Depuis la console de ``virt-manager`` effectuez les manipulations ci-après.

Supprimer la partition todelete
*******************************
Si la partition suivante est présente, on la supprime. ::
  
    lvremove -f /dev/vg00/todelete

.. note:: Cette partition permet de corriger temporairement un bug relatif à une installation via preseed. 

Modifier le hostname
********************
On édite le fichier ``/etc/hostname`` 
  
Modifier le réseau
******************
Si la machine doit bénéficer d'une IP statique, ce qui est nécessaire lors de la création d'une machine délivrant un service, on définit un adressage particulier comme suit :


On cherche une adresse ip libre et on renseigne le dns interne sur ns1.intra.data.gouv.fr ainsi que le reverse dns ::

    /etc/bind/zones/db.intra.data.gouv.fr
    /etc/bind/zones/reverses/db.10.10.10

.. warning:: Ne pas oublier d'incrémenter le serial de la zone. 

On reload la zone ::

    rndc reload

.. note:: Pour vérifier qu'il n'y a pas d'erreur, on peut consulter le fichier de log de bind dans ``/var/log/bind/bind.log`` 

On édite le fichier ``/etc/network/interfaces``

::
    
    # The primary network interface
    auto eth0
    iface eth0 inet static
        address 10.10.10.xx
        netmask 255.255.255.0
        network 10.10.10.0
        broadcast 10.10.10.255
        gateway 10.10.10.254
        mtu 1400

.. note:: Pour faciliter la saisie, un fichier d'exemple peut être utilisé comme il suit : cat /var/lib/etalabinstall/data/interfaces.template >> /etc/network/interfaces 


Si la machine n'a pas besoin d'adresse ip statique, on ne configure pas le réseau. 

Modifier le serveur de mail
***************************
Afin que la machine puisse communiquer par mail, on paramètre correctement le serveur de mail postfix afin de modifier son hostname ainsi que son serveur d'envoi ::

    sed -i s/debian/<nom vm>/ /etc/postfix/main.cf
    echo "relayhost = smtp.intra.data.gouv.fr" >> /etc/postfix/main.cf

La machine est maintenant finalisée, on peut rebooter et s'y connecter via ssh.
