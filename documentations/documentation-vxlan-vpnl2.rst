======================================================================
Mise en place d'une interconnexion logique entre plusieurs datacenters 
======================================================================

Introduction
============

Notre architecture comprend des serveurs physiques et virtuels localisés dans des datacenters différents. 
Les machines physiques qui la compose sont hébergés par des prestataires ne permettant pas la maîtrise des équipements réseaux qui interconnecte nos serveurs.

Ayant besoin de cloîsoner par VLAN nos différentes machines, nous avons du trouver une solution pour résoudre se problème.

La solution à été de mettre en place VxLAN (Virtual eXtensible LAN). Ce format d'encapsulation réseau permet de surcharger les trames réseau de bas niveau afin d'y ajouter des informations relatives au VLAN.

En outre,avec l'utilisation d'une interconnection VPN de niveau 2 (Layer 2 ou L2), nous avons pu faire transiter entre nos trois datacenters les VLAN dont nous avions besoin.


Les informations qui suivents décrivent comment nous avons mis en place VxLAN dans notre architecture ainsi que les VPN L2.

Reading :

  * http://www.randco.fr/blog/2013/vxlan/
  * https://tools.ietf.org/html/draft-mahalingam-dutt-dcops-vxlan-02


Prérequis
---------
Quelques pré-requis ont été nécessaires.

Iproute2
~~~~~~~~
Le projet ip route dans sa version 2 est nécessaire pour utiliser Vxlan. Malheureusement, l'actuelle Debian Stable (wheezy), ne propose pas iproute en version 2, il n'est disponible qu'en version 1. 

Profitant de la faible dépendance avec d'autres librairies, nous avons récupérer le binaire de **/bin/ip** compilé depuis une Debian Jessie, afin de l'utiliser sur des liens réseau nécessitant Vxlan.


Modules Vxlan du noyau linux
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Nous avons besoin des modules vxlan présent dans le kernel. Une petite recherche nous permet de savoir si c'est le cas ::

  find /lib/modules/3.11-0.bpo.2-amd64/ -name '*vx*'

Et on ajoute le module souhaité ::

  modprobe vxlan


Autres
~~~~~~
Quelques notions de bridging : 

  * http://www.linuxfoundation.org/collaborate/workgroups/networking/bridge

Et un peu de lecture : 

  * https://tools.ietf.org/html/draft-mahalingam-dutt-dcops-vxlan-09


Schéma d'architecture de VxLAN
==============================

Sur une machine physique classique
----------------------------------

L'empilement des interfaces réseau suit le schéma ci dessous ::
  
  eth0 --> vxlan10 --> br10 --> rbx

::

  eth0    : est l'interface connecté au monde
  vxlan10 : l'interface surchargeant les trames réseau la traversant
  br10    : le bridge bindé sur vxlan10
  rbx     : le tunnel vpn de niveau 2


Sur un hyperviseur
------------------

L'empilement des interfaces réseau suit le schéma ci dessous ::

  eth1 --> br1 --> vxlan10 --> br10 --> rbx
                                    \-> vm-xx
				    \-> vm-xy

::

  eth1    : est l'interface connecté au monde
  br1     : le bridge bindé sur eth1
  vxlan10 : l'interface surchargeant les trames réseau la traversant
  br10    : le bridge bindé sur vxlan10
  rbx     : le tunnel vpn de niveau 2
  vm-xx   : est l'interface qui fournis le réseau à la VM guest.


Configuration de Vxlan
======================

Sur une machine classique
-------------------------

Methode manuelle
~~~~~~~~~~~~~~~~

Pour tester la connectivité des liens via vxlan, il toujours intéressant de savoir comment procéder manuellement. 

On ajoute une interface vxlan10 reposant sur l'interface physique eth0 ::

	/opt/iproute2/ip link add vxlan10 type vxlan id 10 group 239.0.0.10 ttl 4 dev eth0

On démarre l'interface ::

	/opt/iproute2/ip link set dev vxlan10 up

On déclare un bridge br10 puis on déclare qu'il doit reposer sur vxlan10 ::

	brctl addbr br10
	brctl addif br10 vxlan10

On ajoute l'interface du vpn l2 au réseau vxlan ::
 
	brctl addif br10 rbx

On démarre les interfaces restantes ::

	ifup br10
	ip link set rbx up


Automatisation sur le server Yak
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Pour automatiser la configuration, on ajoute ce parametrage dans les fichiers de configuration de l'OS. 

On edite le fichier nécessaire :

vi /etc/network/interfaces ::

    auto vxlan10
    iface vxlan10 inet manual
          pre-up /opt/iproute2/ip link add vxlan10 type vxlan id 10 group 239.0.0.10 ttl 4 dev eth0
          post-down /opt/iproute2/ip link del vxlan10

    auto rbx
    iface rbx inet manual
          pre-up /etc/init.d/openvpn start
          post-down /etc/init.d/openvpn stop

    auto br10
    iface br10 inet static
          address 10.10.10.4
          netmask 255.255.255.0
          network 10.10.10.0
          broadcast 10.10.10.255
          bridge_ports vxlan10 rbx
          bridge_maxwait 0
          bridge_stp off
          bridge_fd 0


Sur un hypersviseur
-------------------
On configure vxlan pour deux vlans, identifiés par les numéro 10 et 11, et on test le focntionnement de vxlan entre les deux interfaces. Les requêtes de ping sont nécessaire, néanmoins on peut charger les liens avec des tests iperfs..


Methode manuelle ::

  /opt/iproute2/ip link add vxlan10 type vxlan id 10 group 239.0.0.10 ttl 4 dev br1
  /opt/iproute2/ip addr add 10.10.10.1/24 broadcast 10.10.10.255 dev vxlan10
  /opt/iproute2/ip link set dev vxlan10 up
    
  /opt/iproute2/ip link add vxlan11 type vxlan id 11 group 239.0.0.11 ttl 4 dev br1
  /opt/iproute2/ip addr add 10.10.11.2/24 broadcast 10.10.11.255 dev vxlan11
  /opt/iproute2/ip link set dev vxlan11 up



::

  ifup vxlan10
  ifup br10

  ifup vxlan11
  ifup br11


Puis de manière, automatique pour un démarrage au boot de la machine

vi /etc/network/interfaces ::

    auto vxlan10
    iface vxlan10 inet manual
            pre-up /opt/iproute2/ip link add vxlan10 type vxlan id 10 group 239.0.0.10 ttl 4 dev br1
            post-down /opt/iproute2/ip link del vxlan10
    
    auto br10
    iface br10 inet static
            address 10.10.10.1
            netmask 255.255.255.0
            network 10.10.10.0
            broadcast 10.10.10.255
            bridge_ports vxlan10
            bridge_maxwait 0
            bridge_stp off
            bridge_fd 0

On ajuste cette configuration sur les autres machines en fonction du nomage des interface ainsi que des ip. 

VPN LAYER 2
===========

La liaison vpn-l2, permet d'interconnecter les réseaux vxlan entre les machines des différents datacenters. Dans notre cas, se sont les vlans LAN et API qui sont concernés. 
Les datacenters étant Graveline(GRA), Roubaix(RBX) et Strasbourg(SBG), les interfaces vpn sont nommées par leurs trigrames. 


Afin d'assuer une redondance sur ces liens vpn, plusieurs serveur openvpn sont démarrés sur des machines physiques différentes. Par exemple le schéma d'architecture entre GRA et RBX ce présent comme suit ::

              /----> host2 (rbx)
  host1(gra)-[ ----> host3 (rbx)
              \----> host4 (rbx)

Lors d'une perte de connectivité sur l'un des hosts de rbx, la bascule du vpn s'effectue dans un laps de temps n'excedant pas 120 secondes.

Liaison VPN-L2 entre GRA<->RBX
------------------------------
Pour illustrer la mise en place typique d'un vpn l2 entre deux machines, j'ai pris pour exemple une machine ayant la fonction d'hyperviseur, sur le site de Rbx.

On installe openvpn et on génère une clé :: 

    apt-get install openvpn
    openvpn --genkey --secret /etc/openvpn/static.key

Le fichier static.key joue le rôle de jeton d'identification, en chriffrant les liens d'interconnectivités vpn l2. Ceux-ci doivent être les mêmes à chaque endpoint d'une liaison.

Les droits sur ce fichier sont donc sensible. On prendra soit d'appliquer des droits restrictifs comme il suit ::

    chmod go-rwx /etc/openvpn/static.key


On définit la configuration d'openvpn :

vi /etc/openvpn/gra.conf ::

        dev gra
        dev-type tap
        secret static.key
        keepalive 10 60
        persist-tun
        persist-key
        mssfix 1300
        script-security 2
        /etc/openvpn/addbr.sh
	status /var/log/openvpn/status-gra.log
	log /var/log/openvpn/openvpn-gra.log


On définit un script permettant de lancer l'interface sur le bridge utilisé :

vi /etc/openvpn/addbr.sh ::

	#/bin/bash
        /sbin/brctl addif br10 gra
        /sbin/ip link set gra up

On ouvre les flux openvpn pour type de lien dans iptables ::

    # -- Vpn L2 RBX <> GRA
        iptables -A INPUT -p udp -i $IFEXT -s $host -d $IPEXT --dport 1194 -j ACCEPT
    
    # -- Vpn L2 RBX <> GRA
        iptables -A FORWARD -i $IFLAN10 -m physdev --physdev-in $IFVXLAN10 --physdev-out $OVPN_GRA -j ACCEPT
        iptables -A FORWARD -i $IFLAN10 -m physdev --physdev-in $OVPN_GRA -j ACCEPT

::

    IFEXT    :  Correspond à l'interface connecté au monde
    IPEXT    :  Correspond à l'ip public
    IFVXLAN10:  Correspond à l'interface vxlan10
    IFLAN10  :  Correspond à l'interface du lan10
    OVPN_GRA :  Correspond à l'interface d'openvpn


On démarre le service openvpn ::

    service openvpn start


Installation d'openvpn client sur GRA
-------------------------------------
Maintenant que les serveurs openvpn sont en place, on peut configurer les clients. Voici le cas typique d'une machine sur le site de GRA. 

L'installation se fait sur le serveur Yak.

On installe openvpn et on rapatrie la clé static via un canal sécurisé, ssh ::

    apt-get install openvpn
    scp root@wolf:/etc/openvpn/static.key /etc/openvpn/static.key

On définit la configuration d'openvpn :
    
vi /etc/openvpn/rbx.conf ::

        dev rbx
        dev-type tap
        secret static.key
        keepalive 10 60
        persist-tun
        persist-key
        mssfix 1300
        remote wolf.data.gouv.fr 1194
        remote turtle.data.gouv.fr 1194
        remote viper.data.gouv.fr 1194
        script-security 2
        up "/etc/openvpn/addbr.sh"
	status /var/log/openvpn/status-rbx.log
	log /var/log/openvpn/openvpn-rbx.log

.. note:: On définit plusieurs remote. En cas de perte de connectivité sur l'un, on passe à l'autre. 

On définit un script permettant de lancer l'interface sur le bridge utilisé :

vi /etc/openvpn/addbr.sh ::

	#!/bin/bash 
	/sbin/brctl addif br10 rbx
	/sbin/ip link set rbx up


Liaison VPN-L2 entre SBG<->RBX
------------------------------
Cette liaison doit être différente de la première et ne pas interférer avec elle, ou d'autres.. On définit une seconde instance d'openvpn sur un port différent et on génère une autre clé static.

On génère une nouvelle clé :: 
    openvpn --genkey --secret /etc/openvpn/static-sbg.key

Dans la configuration des clients et serveurs d'openvpn, on définiera le port spécifique à utiliser via la variable "port"

Pour le serveur :
vi /etc/openvpn/sbg.conf ::

	[..]
        port 1195
	[..]

Pour le client :
vi /etc/openvpn/rbx.conf ::
	 
	[..]
	remote wolf.data.gouv.fr 1195
	remote turtle.data.gouv.fr 1195
	remote viper.data.gouv.fr 1195
	[..]


