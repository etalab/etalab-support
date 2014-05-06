=============================
Installation des serveurs slb
=============================

Les Serveurs Load Balancer (slb), sont des machines placées entre les frontaux web et les IP publiques. Avec le temps, d'autres fonctionnalités ont été ajoutés à ces machines, notamment les fonctions de firewall, routeur.. 

Les détails de chaque services vont être exposés ci-après. 


.. include:: include-installation-serveur-virtuel-lan.rst

Installation de base
====================

Ajout des interfaces réseau relatives aux deux environnements sur les hyperviseurs ::

   virsh edit slb-0x

::

    [..]
    <interface type='bridge'>
      <mac address='00:16:3e:3e:0a:f5'/>
      <source bridge='br1'/>
      <target dev='vm-slb-020'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
    <interface type='bridge'>
      <mac address='00:16:3e:70:b1:57'/>
      <source bridge='br10'/>
      <target dev='vm-slb-0210'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
    </interface>
    <interface type='bridge'>
      <mac address='00:16:3e:70:b1:58'/>
      <source bridge='br11'/>
      <target dev='vm-slb-0211'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x07' function='0x0'/>
    </interface>
    [..]
    <interface type='bridge'>

.. warning:: Afin d'eviter de dupliquer les configurations, les champs ci-dessous en gras sont à modifier.

**<mac address=**
**<source bridge=**
**<target dev=**
<address type= ...  **slot=**


.. note:: interface public = br1, interface lan=br10 , interface api=br11 
.. note:: Les interfaces br10 et br11 repose sur des Vlans utilisant vxlan. Une documentation à été réalisé à cette effet : documentation-vxlan.rst
 

Configuration du systeme
========================
Pare-feu (Netfilter)
--------------------

La configuration intrisèque du firewall se trouve dans le fichier ::

    /etc/init.d/packetfilter

Les variables de paramétrage sont exportées dans un autre fichier ::
 
    /etc/etalab.conf

L'ajout du firewall dans les scripts de démarrage de Debian s'effectue via la commande ::

    insserv packetfilter   

On pense également à activer des modules noyaux nécessaire ::

    echo "nf_conntrack_ftp" >> /etc/modules
    echo "nf_nat_ftp" >> /etc/modules

Redondance
----------
La redondance des informations relatif Netfilter s'effectue avec conntrackd. 

On installe conntrackd ::

    apt-get install conntrackd

On utilise le mode ft-fw ::

    zcat /usr/share/doc/conntrackd/examples/sync/ftfw/conntrackd.conf.gz > /etc/conntrackd/conntrackd.conf

Plus d'information ici : 

  * https://wiki.d2france.fr/doku.php?id=blog:conntrackd
  * http://conntrack-tools.netfilter.org

Synchronisation des OS
~~~~~~~~~~~~~~~~~~~~~~

FIXME : Rédiger une doc relative a l'utilisation de git.

Haute Disponibilité (Pacemaker)
-------------------------------
Les SLB sont élevées en tant que cluster haut disponibilité grace aux fonctionnalité qu'apporte le projet Pacemaker. 

  * http://clusterlabs.org/

Installation de pacemaker
~~~~~~~~~~~~~~~~~~~~~~~~~

On installe pacemaker ::

    apt-get install pacemaker

On configure corosync qui assure le dialogue entre les machines du cluster

vi /etc/corosync/corosync.conf ::

        secauth: on
        netmtu: 1400
        interface {
            # The following values need to be set based on your environment 
            ringnumber: 0
            bindnetaddr: 10.10.10.xx
            mcastaddr: 226.94.1.1
            mcastport: 5405
        }

On active le démarrage automatique de corosync ::

    sed -i s/START=no/START=yes/ /etc/default/corosync

On génère le fichier authfile pour sécurisé le canal de contrôle de corosync ::

    cd /etc/corosync
    corosync-keygen

Pour verifier que corosync et pacemaker fonctionne correctement, on peut lancer la commande ci-dessous. ::

    crm_mon

Le résultat de cette commande devrait être ::
    
    [...]
    Online: [ slb-02 slb-01 ]

A ce stade pacemaker fonctionne, on peux maintenant créer une configuration avec la commande ::
    
    crm configure edit

Les resources qui seront gérés par pacemaker sont les ip publiques ainsi que les services associés comme haproxy, nginx, varnish..  


Configuration des applications
==============================
Load balancer + Proxy Http Layer 7 (HAProxy)
--------------------------------------------
On installe HAProxy ::

    apt-get install haproxy

On active le démarrage automatique d'HAProxy

    sed -i s/ENABLE=0/ENABLE=1/ /etc/default/haproxy

L'ajout d'HAProxy dans les scripts de démarrage de Debian s'effectue via la commande ::

    insserv haproxy

A ce stade haproxy fonctionne, on peux maintenant définir sa configuration via le fichier ::

    vi /etc/haproxy/haproxy.cfg


.. note :: Les scripts LSB fournis par défaut dans haproxy ne sont pas compatible avec pacemaker. Voici comment nous avons contourné le problème.

Modification des LSB founis par des LSB compatibles avec pacemaker.

Rapatriement des sources tierces ::

    cd /usr/local/src ; git clone https://github.com/russki/cluster-agents.git

On ajoute ce script au resources de pacemaker. De cette façon, il va pouvoir manipuler le service HAProxy sans problème ::

    cp cluster-agents/haproxy /usr/lib/ocf/resource.d/heartbeat/
    chmod 755 /usr/lib/ocf/resource.d/heartbeat/haproxy 


Offload SSL (Nginx)
-------------------
Actuellement (10/04/2014), HAProxy ne gère pas, dans sa version stable, le protocol SSL. Pour utiliser la fonctionnalité d'offload, nous avons donc besoin d'un outil capable d'assurer le déchargement des requêtes HTTPS à destination de notre plateforme.

Le chemin emprûté par les rêquetes des internautes utilisant le HTTPS, se déroule de la manière suivante; l'internaute entre l'adresse du site, Nginx qui écoute sur l'ip publique et sur le port 443, déchiffre le SSL et redirige les requêtes, en HTTP, vers le serveur web.

.. note:: Lorsque c'est utile, les requêtes peuvent-être redirigées vers haproxy pour être loadbalancés vers les serveurs réels. Dans ce cas d'utilisation, on préfèrera utiliser les sockets unix à la place de tcp. 


Installation de Nginx
~~~~~~~~~~~~~~~~~~~~~
On installe Nginx ::

    apt-get install nginx

On configure les vhost de nginx dans les répertoires suivants ::

    /etc/nginx/sites-available/

On active les vhosts ::

    ln -s /etc/nginx/sites-available/pfa.data.gouv.fr 
    service nginx restart


Certificats Ssl
~~~~~~~~~~~~~~~
On installe les certificats SSL ::
  
    cd /etc/ssl/private
    git clone git@git.intra.data.gouv.fr:certificates/data.gouv.fr-certificates.git
    git clone git@git.intra.data.gouv.fr:certificates/openfisca.fr-certificates.git
    chmod -R 640 * && chown -R :ssl-cert *  


Cache HTTP (Varnish)
--------------------
La gestion du cache sur les slb est assurée avec varnish.  

Le chemin emprûté par les rêquetes des internautes utilisant le cache se déroule de la manière suivante; l'internaute renseigne l'adresse du site, varnish qui écoute sur l'ip publique et sur le port 80, traite les requêtes etrevoit à l'internaute les éléments présent dans son cache.
Pour les autres éléments, les requêtes sont redirigées vers le serveur web dédié qui fourniera les éléments manquants et ajoute en même temps ces derniers à son cache. 
 
.. note:: Lorsque c'est utile, les requêtes peuvent-être redirigées vers haproxy pour être loadbalancés vers les serveurs réels.

Mise en place
~~~~~~~~~~~~~
On définit un espace de stockage dédié au cache disque pour varnish ::
  
    mkdir  /var/lib/varnish/cache 
    chown varnish:varnish /var/lib/varnish/cache

On install varnish ::

    apt-get install varnish

On configure le daemon :: 

    /etc/default/varnish
    
    [...]
    DAEMON_OPTS="-a 37.59.183.76:80 \
             -T localhost:6082 \
             -f /etc/varnish/etalab.vcl \
             -S /etc/varnish/secret \
             -s memory=malloc,500M \
             -s sata=file,/var/lib/varnish/cache,70%"
    [...]

On le configure via les fichiers de configuration suivants ::

    /etc/varnish/etalab.vcl
    /etc/varnish/acl.vcl
    /etc/varnish/openfisca.backend.vcl  
    /etc/varnish/openfisca.recv.vcl

On ajoute varnish à pacemaker ::
    
    primitive varnish lsb:varnish \
        op monitor interval="10s" \
        meta is-managed="true" target-role="Started"

Quelques commandes utiles relative à Varnish ::
 
    varnishlog -m TxStatus:500          :  Voir les erreur 500 retournés au clients
    varnishlog -O -i TxURL              :  Voir les resources non cachées (Cache miss or Content not cached)
    varnishlog -i TxURL,TxHeader -b -o  :  Idem mais goupé par ID, avec les headers associés 
    curl -X PURGE [url du site]         :  Purger le cache de varnish


Métrologie (Munin)
------------------
Les plugins ont été installés en se basant sur la procedure suivante. 

.. include:: procedure-ajout-service-metrologie.rst


Mise en place des plugins munin pour HAProxy
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
On installe munin ainsi que certaines dépendances, nécessaire dans l'environnement d'HAProxy ::

    apt-get install munin-node munin-plugins-extra ca-certificates socat

On peut vérifier le fonctionnement du socket utilisé par munin et initié par haproxy ::

    echo "show stat" | socat unix-connect:/var/run/haproxy.stat stdio  

On installe le dépot contrib de munin ::

    cd /usr/local/src/
    git clone https://github.com/munin-monitoring/contrib.git /usr/local/src/munin-monitoring

On met en place les plugins ::

    ln -s /usr/local/src/munin-monitoring/plugins/haproxy/haproxy_rate_frontend /etc/munin/plugins/
    ln -s /usr/local/src/munin-monitoring/plugins/haproxy/haproxy-sessions-by-servers /etc/munin/plugins/
    ln -s /usr/share/munin/plugins/ip_ /etc/munin/plugins/ip_37.59.183.74
    ln -s /usr/share/munin/plugins/ip_ /etc/munin/plugins/ip_37.59.183.75
    ln -s /usr/share/munin/plugins/ip_ /etc/munin/plugins/ip_37.59.183.80
    ln -s /usr/share/munin/plugins/ip_ /etc/munin/plugins/ip_37.59.183.81
    ln -s /usr/share/munin/plugins/ip_ /etc/munin/plugins/ip_37.59.183.70
    ln -s /usr/share/munin/plugins/if_ /etc/munin/plugins/if_eth0
    ln -s /usr/share/munin/plugins/if_ /etc/munin/plugins/if_eth1
    ln -s /usr/share/munin/plugins/if_ /etc/munin/plugins/if_eth2


On configure le plugin en ajoutant le bloc suivant dans le fichier */etc/munin/plugin-conf.d/munin-node* ::
    
    [haproxy*]
    user root
    env.backend WEBUI-DEV API-DEV DATAGOUVFR WIKI-DATAGOUVFR STATIC-DATAGOUVFR
    env.frontend WEBUI-DEV API-DEV DATAGOUVFR WIKI-DATAGOUVFR STATIC-DATAGOUVFR
    env.socket /var/run/haproxy.stat

On redémarre *munin-node* pour qu'il prenne en compte ces nouveaux plugins ::

  service munin-node restart

Au prochain lancement du cron sur le serveur centrale, les nouveaux plugins seront détectés et graphés.

Mise en place des plugins munin pour Varnish
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
On installe le dépôt de basiszwo ::

    cd /usr/local/src/
    git clone git://github.com/basiszwo/munin-varnish.git
    chmod a+x /usr/local/src/munin-varnish/varnish_*

On met en place les plugins ::

    cd /etc/munin/plugins
    ln -s /usr/local/src/munin-varnish/varnish_cachehitratio 
    ln -s /usr/local/src/munin-varnish/varnish_hitrate

On configure les plugins ::

    cat < EOF >> /etc/munin/plugin-conf.d/munin-node
    
    [varnish*]
    user root
    EOF
    
On redémarre *munin-node* pour qu'il prenne en compte ces nouveaux plugins ::

  service munin-node restart

Supervision (Icinga)
--------------------
Mise en place d'un check icinga pour Pacemaker
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Le plugins pour pacemaker est un plugin maison, localisé ici ::

    /usr/lib/nagios/plugins/check_crm_status_change

A l'installation ainsi qu'a chaque modification de pacemaker, il faudra executer la commande d'initialisation suivante ::

    /usr/lib/nagios/plugins/check_crm_status_change init
