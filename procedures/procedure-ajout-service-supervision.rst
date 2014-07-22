Installation de etalabnrpe
--------------------------
Sur la machine à superviser, on installe le package responsable de l'environnement NRPE d'Etalab.

::
    apt-get install etalabnrpe

On défini les hosts autorisés ::

    echo "allowed_hosts=yak.intra.data.gouv.fr" >> /etc/nagios/nrpe_local.cfg
    echo "allowed_hosts=yak.data.gouv.fr" >> /etc/nagios/nrpe_local.cfg

On redémarre le service ::

    service nagios-nrpe-server restart

Déclaration sur la machine de supervision
-----------------------------------------

On déclare un nouveau host ::

    vi /etc/icinga/objects/etalab-hosts.cfg
        
        define host{
            use             generic-debian
            host_name       <nom_host>
            address         <fqdn ou ip>
            parents         <host parent si il y'en a un>
        }

On déclare les services à superviser ::

    vi /etc/icinga/objects/etalab-services.cfg

On reload icinga ::

    service icinga reload


Configuration d'Iptables si il est présent
------------------------------------------
::

    # -------------------------- Icinga
    iptables -A INPUT -p tcp -s $ICINGA -i $IFEXT0 --dport 5666 -j ACCEPT
