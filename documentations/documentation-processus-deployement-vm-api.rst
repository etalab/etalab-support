=======================================================
Deployement de machine virtuelle de l'environnement API
=======================================================

Introduction
============
Le déployement des VM api, sous entendu API openfisca, s'axe autour de technologie simple et basique. 
Le déployement automatise 80% du travail à faire pour mettre en place une nouvelle node API.

Les interventions manuelles sont documentées dans le dépot "etalab-support" sur github. Précisement dans, procedures/installation-os-de-base-serveur-virtuel.rst et dans ce document.

Technologie mise en oeuvre
==========================
Les technologies misent en oeuvre sont les suivantes :

    * Serveur DHCP    : géré sur les machines SLB via le service dhcpd.
    * Serveur PXE     : géré sur la machine NS1 via le service tftpd.
    * Serveur Preseed : scripts preseed et post-install présents sur la machine GIT.

Diagramme de séquence
=====================
::

    0/ Action manuelle : Création du storage de la vm dans CEPH. 
    1/ Requête Dhcp de la VM fraîchement crée, pour récupérer l'adresse du serveur PXE.
    2/ Le serveur Dhcp fourni une adresse IP et met à jour les zones dns interne via Dynamic DNS. 
    3/ Un script lancé via cron, met a jour le futur nom de la VM. 
    4/ Choix de l'environnement à installer. Sans intervention manuelle, après 10 minutes, autoinstallation d'un environnement API.
    5/ Récéption de l'image de boot via TFTP
    6/ Lancement de l'installeur Debian
    7/ Récupération du fichier PRESEED
    8/ Récupération en fin d'installation Preseed, d'un script de post-installation.
    9/ Action manuelle : Déclaration de la node API dans HAProxy (dans le back-end prévu).

::

	+------------------+        +-----------+         +----------+        +-----------+
	|     Vms APi      |        |    Slb    |         |    Ns1   |        |    Git    |
	+--------+---------+        +-----+-----+         +-----+----+        +-----+-----+
	         |           (1)          |                     |                   |      
	         +------------------------>                     |                   |      
	         <------------------------+                     |                   |      
	         |                        |         (2)         |                   |      
	         |                        +--------------------->       (3)         |      
	         |                        |                     <-------------------+      
	         |(4)                     |                     |                   |      
	         |           (5)          |                     |                   |      
	         +---------------------------------------------->                   |      
	         <----------------------------------------------+                   |      
	         |                        |                     |                   |      
	         |(6)                     |                     |                   |      
	         |           (7)          |                     |                   |      
	         +------------------------------------------------------------------>      
	         <------------------------------------------------------------------+      
	         |           (8)          |                     |                   |      
	         +------------------------------------------------------------------>      
	         <------------------------------------------------------------------+      
	         |                        |                     |                   |      
	         +                        +                     +                   +     
	

Précisions techniques
=====================
Serveur Dhcp
------------
Sur le serveur dhcp, qui se trouve sur les SLB, on trouvera les options de configuration suivantes, permettant de booter la machine sur le serveur PXE.
    
vi /etc/dhcp/dhcpd.conf ::

	###serveur tftp install Debian
	option tftp-server-name "10.10.10.10";
	next-server 10.10.10.10;
	filename "/pxelinux.0";

Le service permet également de mettre à jour automatiquement certaines zones dns internes avec l'utilisation de Dynamic DNS.

Serveur PXE
-----------
Le serveur PXE (Preboot eXecution Environnement), publie plusieurs environnements disponibles à l'installation. 

Ce service est installé sur le serveur "NS1". Les configurations se trouvent ici ::

    /srv/tftp/pxelinux.cfg/*

L'OS GNU/Linux Debian en version netboot se trouve ici ::

    /srv/tftp/debian-installer

Pour mettre à jour debian-installer, on trouvera les dépots officiels ici ::
    
    * http://ftp.fr.debian.org/debian/dists/wheezy/main/installer-amd64/current/images/netboot/debian-installer

Service Preseed
---------------
Par convention, on utilise le terme "preseed" pour nommer la méthode utilisée par l'installeur Debian, pour installer le système Debian. 

Pour automatiser l'installation, on pré renseigne certains champs nécessaires au système. Ceux-ci permettent de définir la configuration du système de base que l'on souhaite obtenir au terme de l'execution de l'installeur. Les champs pré renseignés sont stockés dans un fichier de configuration que l'on appel communement "preseed".

Au démarrage de l'installeur Debian, le fichier preseed est rapatrié. L'emplacement de celui-ci à été préallablement définit sur le serveur PXE. Pour information, le fichier preseed ce trouve sur la machine GIT et est mis à disposition via un serveur Web.

Au terme de l'execution des commandes Debian, on execute des scripts de post-install. Ces scripts permettent d'auto-configurer l'environnement API.

Les fichiers preseed et post-install se trouvent ici ::

    /home/debian/public_html/preseed

