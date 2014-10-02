Commandes usuelles relatives à Pacemaker
========================================

La haute-disponibilité est assurée par le programme Pacemaker. Le service, ou daemon, associé s'appel *corosync*. Pour modifier sa configuration, il faut utiliser l'outil de configuration *crm*.

L'outil *crm* peut être lancé avec en argument l'action à effectuer et ses paramètres, ou bien en mode interactif, en ne passant aucun paramètre.

En mode interactif, la commande *help* permet d'avoir des informations sur les commandes disponibles. La commande *help* peut aussi être utilisée en précisant en paramètre le nom d'une commande afin d'obtenir plus d'informations sur l'utilité et l'utilisation de cette commande.

Le mode interactif est subdivisé en niveau correspondant au type d'action à effectuer ou au type d'objet à manipuler. Les principaux types d'objets manipulés sont :

* les *nodes* : les noeuds du cluster, autrement dit les serveurs prennant part au cluster ;
* les *resources* : les services rendu par le cluster

Les sous-niveaux principalement utilisés sont:

* *ressource* : manipulations de ressources du cluster (arrêt, démarage, migration, ...)
* *configure* : visualisation et édition de la configuration du cluster

Dans un sous-niveau, la commande *help* affiche l'aide des commandes du sous-niveau.

La commande *cd* permet de remonter d'un niveau et la commande *exit* permet de quitter l'outil *crm*.

Afficher l'état actuel du cluster
---------------------------------
Pour afficher le status du cluster on execute la commande suivante ::

    crm status
 
ou, celle-ci dessous pour un affichage en live ::

    crm_mon

Ces commandes affichent l'output suivant ::

	============
	Last updated: Tue Sep 16 11:16:44 2014
	Last change: Thu Jul 31 15:21:27 2014 via crm_attribute on slb-01
	Stack: openais
	Current DC: slb-01 - partition with quorum
	Version: 1.1.7-ee0730e13d124c3d58f00016c3376a1de5323cff
	2 Nodes configured, 2 expected votes
	25 Resources configured.
	============
	
	Online: [ slb-02 slb-01 ]
	
	 Resource Group: g_lan
	     p_vip_lan  (ocf::heartbeat:IPaddr2):   Started slb-02
	     p_vip_lan11    (ocf::heartbeat:IPaddr2):   Started slb-02
	     p_vip_test_lan11   (ocf::heartbeat:IPaddr2):   Started slb-02
	 Resource Group: g_wan
	     p_vip_ext0 (ocf::heartbeat:IPaddr2):   Started slb-02
	     p_ipext_ui (ocf::heartbeat:IPaddr2):   Started slb-02
	     p_ipext_api    (ocf::heartbeat:IPaddr2):   Started slb-02
	     p_ipext_www_datagouvfr (ocf::heartbeat:IPaddr2):   Started slb-02
	     p_ipext_wiki_datagouvfr    (ocf::heartbeat:IPaddr2):   Started slb-02
	     p_ipext_static_datagouvfr  (ocf::heartbeat:IPaddr2):   Started slb-02
	     p_ipext_www_openfiscafr    (ocf::heartbeat:IPaddr2):   Started slb-02
	     p_ipext_wiki_openfiscafr   (ocf::heartbeat:IPaddr2):   Started slb-02
	     p_ipext_uipre  (ocf::heartbeat:IPaddr2):   Started slb-02
	     p_ipext_apipre (ocf::heartbeat:IPaddr2):   Started slb-02
	     p_ipext_mx-01  (ocf::heartbeat:IPaddr2):   Started slb-02
	     p_ipext_www_cada   (ocf::heartbeat:IPaddr2):   Started slb-02
	     p_ipext_www_datagouvfrv3   (ocf::heartbeat:IPaddr2):   Started slb-02
	     p_ipext_mail_datagouvfr    (ocf::heartbeat:IPaddr2):   Started slb-02
	     p_ipext_www_etalabgouvfr   (ocf::heartbeat:IPaddr2):   Started slb-02
	     nginx  (lsb:nginx):    Started slb-02
	     varnish    (lsb:varnish):  Started slb-02
	     haproxy    (ocf::heartbeat:haproxy):   Started slb-02
	     dhcpd  (lsb:isc-dhcp-server):  Started slb-02
	     p_ipext_hetic  (ocf::heartbeat:IPaddr2):   Started slb-02
	     p_ipext_oc_datagouvfr  (ocf::heartbeat:IPaddr2):   Started slb-02

On distingue sur cet output, éléments suivant ::

    - Online                  : Deux nodes, slb-01 et slb-02 font partie du cluster 
                                  et sont fonctionnelles,
    - Resource Group          : Deux groupes de ressources ont été défini g_lan et g_wan,
    - Started slb-02          : Les ressources sont actives sur le serveur slb-02, 
    - ocf::heartbeat:IPaddr2  : Ces ressources sont gérés par des scripts OCF (Open Cluster Framework)
                                  et que le script IPaddr2 est utilisé. Ce sont donc des adresses ip. 
    - lsb:                    : Ces ressources sont gérés via scripts LSB (Linux Standard Base).


Afficher la configuration du cluster
------------------------------------
Pour afficher la configuration du cluster, on execute la commande *crm configure show*

Voici un extrait de cette commande, suivit de quelques explications. Ici on définit les machines qui font partie intégrante du cluster ::

	node slb-01 \
	        attributes standby="off"
	node slb-02 \
	        attributes standby="off"

Là, on définit une ressources qui correspondent à la gestion du service HAProxy ainsi que Nginx. La ressource HAProxy est géré via scripts OCF, alors que Nginx est géré via les scripts LSB. On s'assure du fonctionnement des ressources via  *op monitor* toute les 10 secondes.

::

	primitive haproxy ocf:heartbeat:haproxy \
	        params conffile="/etc/haproxy/haproxy.cfg" \
	        op monitor interval="10s" \
	        meta is-managed="true"
	primitive nginx lsb:nginx \
	        op monitor interval="10s" \
	        meta is-managed="true"

On définit une ressource qui correspond à une IP publique *param ip* ainsi que l'adresse de broadcast, le masque au format CIDR et pour finir l'interface réseau sur laquelle sera stockée l'ip *nic*. Cette adresse ip pourra ensuite être utilisée par differents service comme Nginx, HAProxy ou Varnish par exemple ::

	primitive p_ipext_api ocf:heartbeat:IPaddr2 \
	        params ip="37.59.183.72" broadcast="37.59.183.64" cidr_netmask="27" nic="eth0" \
	        op start interval="0" timeout="60s" \
	        op stop interval="0" timeout="60s" \
	        op monitor interval="15s" timeout="60s"

On définit un groupe de ressource afin de lier certaines resources entre elles. L'ordre de déclaration des ressources est important car il défini ordre de démarrage de celles-ci. 
On veillera à déclarer les services non critique en dernier pour éviter, en cas de problème au démarrage du dit service, de bloquer le démarrage des ressources suivantes ::
 
	group g_lan p_vip_lan p_vip_lan11 p_vip_test_lan11 \
	        meta target-role="Started"
	group g_wan p_vip_ext0 p_ipext_ui p_ipext_api p_ipext_www_datagouvfr p_ipext_wiki_datagouvfr p_ipext_static_datagouvfr p_ipext_www_openfiscafr p_ipext_wiki_openfiscafr p_ipext_uipre p_ipext_apipre p_ipext_mx-01 p_ipext_www_cada p_ipext_www_datagouvfrv3 p_ipext_mail_datagouvfr p_ipext_www_etalabgouvfr nginx varnish haproxy dhcpd p_ipext_hetic p_ipext_oc_datagouvfr \
	        meta target-role="Started"

On définit que les ressources doivent être allouées à slb-02 de manière prioritaire par rapport à slb-01.
On indique à pacemaker un poids(score) supérieure pour la node slb-02 ::

	location l_wan g_wan 50: slb-02

On déclare une contrainte de coloaction afin que les groupes de ressources définies soient liés en cas de bascule d'un des groupes. ::
 
	colocation c_lan_wan inf: g_lan g_wan

On défini l'ordre dans laquelle les groupes de ressources doivent être déplacés en cas de bascule. ::

	order o_lan_wan inf: g_lan g_wan

On définit des options de configuration générales au cluster ::

	property $id="cib-bootstrap-options" \
	        dc-version="1.1.7-ee0730e13d124c3d58f00016c3376a1de5323cff" \
	        cluster-infrastructure="openais" \
	        stonith-enabled="false" \
	        no-quorum-policy="ignore" \
	        expected-quorum-votes="2" \
	        last-lrm-refresh="1400575688"
	rsc_defaults $id="rsc-options" \
	        resource-stickiness="INFINITY" \
	        failure-timeout="60s" \
	        migration-threshold="1"

:: 

    - stonith-enabled       : On ne fait pas de stonith
    - resource-stickiness   : En cas de recovery de la node et/ou ressource qui ont posé problème, 
                              on ne les rebascule pas sur leurs node d'origine. 
    - failure-timeout       : On définit un reset automatique par minutes, des "failcount" précédement contatés par crm  
    - migration-threshold   : On concidère qu'une bascule est nécessaire dès qu'un failcount est constaté.



Modifier la configuration du cluster
------------------------------------
Pour modifier la configuration du cluster on execute la commande *crm configure edit*

Basculer les services d'un hyperviseur à l'autre
------------------------------------------------
Si l'on souhaite basculer tout les services d'une machine à l'autre, il faut déclaré la node active en standby. Cela s'effecute via la commande suivante :

.. note:: Idéalement avant d'executer cette commande on lancera dans un autre terminal la commande visualisation live de crm *crm_mon*

::

    crm node standby


Visualiser les log de pacemaker
-------------------------------
Pour visualiser les logs de pacemaker on afficher le contenu de */var/log/daemon.log*




