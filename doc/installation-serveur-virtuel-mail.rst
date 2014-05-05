===========================
Installation du server mail
===========================

Introduction
============

Le service Etalab au sein du SGMAP (Sécrétariat Général de la Modernisation de l'Action Publique) à décidé de mettre à disposition du publique un certain nombre de documentation relatif à son Système d'Information. 

Cette documentation technique à pour but de montrer comment le SGMAP assure son service de messagerie.

Tous les services présent dans ce document fonctionnent sur un seul serveur virtuel. On a nommé ce serveur, le serveur "mail".

Les logiciels utilisés pour assurer les fonctionnalités sont les suivants ::

    * Postfix : Pour assurer l'envoi, la réception et le routage des mails (Protocols : Smtp et Smtps)
    * Dovecot : Pour assurer l'accès aux boites mails par les clients finaux (Protocol : Imaps)
    * SOGo    : Pour assurer un accès par webmail pour les clients finaux (Protocol: Https)
    * Mysql   : Pour unifier les comptes des utilisateur de la plateforme dans une base de donnée SQL.
    * Postfixadmin : Pour assurer une administration simplifiée des comptes utilisateurs. (Protocol : Https)


A noter que la réception et l'envoi vis à vis du monde extérieur, sont gérés par un serveur smtp tiers, lui aussi avec postifx.

L'installation et la configuration de chaque service vont être exposées ci-après.


Installation de base
====================

.. note :: Le serveur virtuel se base sur une installation standard Etalab suivant la procédure **include-installation-serveur-virtuel-lan.rst**

Installation des applications
=============================

Service de base de donnée (Mysql)
---------------------------------

On installe le serveur Mysql ::

  apt-get install mysql-server

Dans le soucis d'un mimimum d'optimisation, on définit l'option suivante dans la configuration de mysql. ::

  echo "innodb_file_per_table = 1" >> /etc/mysql/my.cnf


Service d'envoi/récéption (Postfix)
-----------------------------------

Le serveur postfix va assurer les fonctions suivantes :

    - Envoi du courier pour les services présents sur ce serveur, 
    - Réception des emails des utilisateurs,
    - Envoi du courier pour les utilisateur authentifiés,
    - Authentifier les utilisateurs avec SASL bindé sur Dovecot,

On installe le serveur postfix. ::

    apt-get install postfix

On modifie la configuration par défaut avec les informations relatif à l'infrastructure ::

    sed -i 's/inet_interfaces = localhost/inet_interfaces = all/' /etc/postfix/main.cf
    sed -i 's-mynetworks = 127.0.0.0/8-mynetworks = 127.0.0.0/8 10.10.10.7-' /etc/postfix/main.cf
    echo "relayhost = [smtp.intra.data.gouv.fr]" >> /etc/postfix/main.cf

On ajoute la configuration virutal de postfix ::

    cat < EOF >> /etc/postfix/main.cf
    # VIRTUAL DOMAIN
	# Aliases
	virtual_alias_maps = proxy:mysql:$config_directory/mysql_virtual_alias_maps.cf
	# Accounts
	virtual_mailbox_domains = proxy:mysql:$config_directory/mysql_virtual_domains_maps.cf
	virtual_mailbox_maps = proxy:mysql:$config_directory/mysql_virtual_mailbox_maps.cf
    EOF

On déclare un service dovecot pour postfix ::
 
    cat < EOF >> /etc/postfix/master.cf
    dovecot   unix  -       n       n       -       -       pipe
     flags=DRhu user=vmail:mail argv=/usr/lib/dovecot/deliver -d ${recipient}
    EOF
	
On route les mails vers dovecot afin qu'ils soient stockés ::

    cat < EOF >> /etc/postfix/main.cf
	# Transport
	virtual_transport = dovecot
    dovecot_destination_recipient_limit=1
    EOF

.. note :: Les informations contenu dans les fichiers mysql_* sont définis plus loin. 


Activer SASL
~~~~~~~~~~~~

On configure les fonctionnalités SASL de postfix ::

vi /etc/postfix/master.cf ::
submission inet n       -       -       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject

vi /etc/postfix/main.cf
# SASL Configuration
smtpd_sasl_auth_enable = yes
smtpd_sasl_local_domain = $myhostname
smtpd_sasl_security_options = noanonymous
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_tls_auth_only = yes
smtpd_tls_security_level=may


# SSL/TLS Configuration
smtpd_tls_cert_file = /etc/ssl/private/data.gouv.fr-certificates/wildcard.data.gouv.fr-certificate.crt
smtpd_tls_key_file = /etc/ssl/private/data.gouv.fr-certificates/private-key-raw.key
smtpd_tls_CAfile = /etc/ssl/private/data.gouv.fr-certificates/ca-wildcard-certificate-chain.crt
smtpd_use_tls = yes

.. note :: Les certificats ont été préalablement générés via un organisme tiers. 

#
# SMTPd check
#
smtpd_recipient_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_unauth_destination
smtpd_sender_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_non_fqdn_sender, reject_unknown_sender_domain


La gestion de l'authentification des utilisateurs est déléguée à dovecot. Il faut donc activer une socket unix sur le serveur dovecot pour que postfix puisse l'intérroger.

.. warning :: Les paramètres de configuration suivant, sont liés au serveur dovecot. Néanmoins, dans un soucis de compréhension, ils sont définis à cette endroit de la documentation. 

vi /etc/dovecot/conf.d/10-master.conf ::

  unix_listener /var/spool/postfix/private/auth {
    mode = 0666
    user = postfix
    group = postfix
  }

vi /etc/dovecot/conf.d/10-auth.conf ::

     auth_mechanisms = plain login


Service d'administration web des comptes de messagerie (Postfixadmin)
---------------------------------------------------------------------
Installation d'apache2 
~~~~~~~~~~~~~~~~~~~~~~
Un serveur web est nécessaire pour l'interface de postfixadmin

On installe apache ::
    
    apt-get install apache2

On active les modules nécessaire ::

    a2enmod rewrite

La configuration d'apache se trouve ici ::

  /etc/apache2/sites-available/pfa

avec ::

    <VirtualHost *:80>
        ServerName pfa.data.gouv.fr
        DocumentRoot /usr/share/postfixadmin

        ErrorLog  /var/log/apache2/pfa.data.gouv.fr.error.log
        CustomLog /var/log/apache2/pfa.data.gouv.fr.access.log combined_proxy

        ## Force https.
        RewriteEngine On
        RewriteCond %{HTTPS} !on
        RewriteRule (.*) https://pfa.data.gouv.fr$1 [QSA,R=301,L]
    </VirtualHost>

On active le site ::

    a2ensite pfa


Installation de postfixadmin
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
On installe le service Postfixadmin ::

  apt-get install postfixadmin postfix-mysql

Les informations de configurations relative à la base de donnée sont enregistrées dans le fichier de configuration de postfixadmin ``/etc/postfixadmin/dbconfig.inc.php``

On défini les requêtes sql que postfix devra effectuer pour lister les comptes email présents :

vi /etc/postfix/mysql_virtual_domains_maps.cf ::
      user            = postfix
      password        = *****************************
      hosts           = localhost
      dbname          = postfixadmin
      query           = SELECT domain FROM domain WHERE domain='%s' AND backupmx = '0' AND active = '1'

vi /etc/postfix/mysql_virtual_mailbox_maps.cf ::
      user            = postfix
      password        = *****************************
      hosts           = localhost
      dbname          = postfixadmin
      query           = SELECT maildir FROM mailbox WHERE username='%s' AND active = '1'

vi /etc/postfix/mysql_virtual_alias_maps.cf ::
      user            = postfix
      password        = **********************
      hosts           = localhost
      dbname          = postfixadmin
      query           = SELECT goto FROM alias WHERE address='%s' AND active = '1'

L'accès à postfixadmin ce fait via ``https://pfa.data.gouv.fr``

.. note :: Plus de documentation ici -> /usr/share/doc/postfixadmin/DOCUMENTS/POSTFIX_CONF.txt.gz 



Service de gestion des boites mails (Dovecot)
---------------------------------------------

Le service dovecot va assurer l'interface entre la base de mail au format MailDir et les clients de messagerie des utilisateurs finaux. Le protocol servie pour ce faire sera uniquement l'IMAPS.

En association avec managesieve, dovecot permettra également aux utilisateur de gérer des filtres de message.

L'authentification des utilisateurs se fait sur la base de donnée Mysql. 


Installation de dovecot et des services associés
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
On installe les services relatifs au fonctionnement de dovecot ::
   
  apt-get install dovecot-common dovecot-mysql dovecot-imapd dovecot-managesieved dovecot-sieve


Ajout des certificats SSL
~~~~~~~~~~~~~~~~~~~~~~~~~
Les certificats d'Etalab sont stockés sur un serveur Git interne. 

::
  
    cd /etc/ssl/private/
    git clone git@git.intra.data.gouv.fr:certificates/data.gouv.fr-certificates.git
    git clone git@git.intra.data.gouv.fr:certificates/openfisca.fr-certificates.git
    chmod -R 640 * && chown -R :ssl-cert *

Préparation du filesystem
~~~~~~~~~~~~~~~~~~~~~~~~~
On défini un volume dédié au stockage des mails afin d'éviter le blocage du système en cas de remplissage complet du file system. ::

    lvcreate -L 20g -n vmail vg00
    mkfs.ext4 /dev/vg00/vmail
    mkdir /srv/vmail
    echo "/dev/mapper/vg00-vmail /srv/vmail     ext4    defaults        0   2" >> /etc/fstab
    mkdir /srv/vmail/users

Création d'un utilisateur pour dovecot
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::
    useradd -m -s /bin/false -d /srv/vmail vmail
    chown -R vmail:mail /srv/vmail

Configuration du service imap
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
On défini les parametres du daemon dovecot.

.. note:: D'autres valeurs sont définies par défaut et on les laisse telles quelles. Néanmoins on commente pop3 qui ne sera pas utilisé ici. 

vi /etc/dovecot/conf.d/10-master.conf :: 

    service imap-login {
        inet_listener imap {
        #port = 143
        }
    inet_listener imaps {
        #port = 993
        #ssl = yes
        }
    process_limit = 512
    }

    [...]
    service imap {
        process_limit = 1024
    }
    

On défini les parametres relatifs à la configuration des fichiers stockant les boites mails. Leurs emplacements, leurs types. Pour ce faire on edite le fichier **10-mail.conf**

vi /etc/dovecot/conf.d/10-mail.conf ::

    mail_location = maildir:~/Maildir
    namespace inbox {
        type = private
        separator = .
        inbox = yes
    }
    
    [...]
    
    mail_uid = vmail
    mail_gid = mail



On modifie le processus d'autentification de dovecot, en modifiant les valeurs ci-dessous dans le fichier **10-auth.conf** ::

vi /etc/dovecot/conf.d/10-auth.conf ::

    disable_plaintext_auth = no
    auth_mechanisms = plain
    !include auth-sql.conf.ext



On renseigne les informations concernant les certificats ssl à utiliser dans le fichier **10-ssl.conf** ::

vi /etc/dovecot/conf.d/10-ssl.conf ::

    ssl = yes
    ssl_cert = </etc/ssl/private/data.gouv.fr-certificates/wildcard.data.gouv.fr-certificate.crt
    ssl_key = </etc/ssl/private/data.gouv.fr-certificates/private-key-raw.key


On crée le fichier de configuration necessaire à la connexion à mysql et on positionne les droits correctement :: 

    chmod 0600 dovecot-sql.conf.ext

On edite **dovecot-sql.conf.ext** et on renseigne les informations suivantes ::

vi /etc/dovecot/dovecot-sql.conf.ext ::

	driver = mysql
    [...]
	connect = host=smtp.intra.data.gouv.fr dbname=postfixadmin user=foobar password=foobar
    [...]
    default_pass_scheme = MD5
	[...]
	password_query = SELECT username AS user, password \
	                 FROM mailbox \
	                 WHERE username = '%u' AND active = '1' ;

	user_query = SELECT concat('/srv/vmail/users/', maildir) AS home, \
	                    concat('maildir:/srv/vmail/users/', maildir) AS mail, \
	                    1000 AS uid, 8 AS gid \
	             FROM mailbox \
	             WHERE username = '%u' AND active = '1';


On donne les droits de lecture à dovecot sur la base de données de postfixadmin et plus précisement sur la table mailbox ::

	GRANT SELECT ON postfixadmin.mailbox TO 'dovecot'@'localhost' IDENTIFIED BY 'foobar_password';



Configuration du service de filtre (ManageSieve)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Le service de filtre est nécessaire pour gérer, par exemple, les mails d'autoréponses que sogo va générer dans le cas d'une période de vacances pour l'utilisateur.

On active donc sieve via les fichiers suivants :

.. note:: D'autres valeurs sont définies par défaut et on les laisse telles quelles.

vi /etc/dovecot/conf.d/20-managesieve.conf ::

    service managesieve-login {
    inet_listener sieve {
        port = 4190
    }
    service_count = 1
    }

vi /etc/dovecot/conf.d/15-lda.conf ::

    protocol lda {
    # Space separated list of plugins to load (default is global mail_plugins).
    mail_plugins = $mail_plugins sieve
    }

vi /etc/dovecot/conf.d/90-sieve.conf ::

    plugin {
        #sieve = ~/.dovecot.sieve
        sieve_dir = ~/sieve
    }

On redemarre le service dovecot ::
    
    service dovecot restart


Service de webmail (SOGo)
-------------------------
Installation de Sogo
~~~~~~~~~~~~~~~~~~~~
Pour l'installation de sogo, nous allons suivre les étapes ci dessous. En plus de sogo lui même, on installera également les dépendances nécessaires.

Ajout du dépôt fourni par l'éditeur Sogo ::

    # Sogo
    deb http://inverse.ca/debian wheezy wheezy
    deb http://ftp.fr.debian.org/debian/ wheezy-backports main contrib non-free

    apt-key adv --keyserver hkp://keys.gnupg.net:80 --recv-key 0x810273C4

On met à jour apt et on installe les packages nécessaires ::

    apt-get update
    apt-get install sogo sope4.9-gdl1-mysql memcached apache2 libapache2-mod-php5

    
    /usr/share/doc/tmpreaper/README.security.gz
    sed -i 's/SHOWWARNING=true/SHOWWARNING=false/' /etc/tmpreaper.conf


Configuration du webmail sogo
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

On edite le fichier **/etc/sogo/sogo.conf**

::
     /* Database configuration (mysql://) */    
	SOGoProfileURL = "mysql://bar:foobar@smtp.intra.data.gouv.fr:3306/sogodb/sogo_user_profile";
	OCSFolderInfoURL = "mysql://bar:foobar@smtp.intra.data.gouv.fr:3306/sogodb/sogo_folder_info";
	OCSSessionsFolderURL = "mysql://bar:foobar@smtp.intra.data.gouv.fr:3306/sogodb/sogo_sessions_folder";

::

	/* Mail */
	SOGoDraftsFolderName = INBOX/Drafts;
	SOGoSentFolderName = INBOX/Sent;
	SOGoTrashFolderName = INBOX/Trash;
	SOGoIMAPServer = imap://127.0.0.1:143;
	SOGoSieveServer = sieve://127.0.0.1:4190;
	SOGoSMTPServer = smtp.intra.data.gouv.fr;
	SOGoMailDomain = data.gouv.fr;
	SOGoForceExternalLoginWithEmail = NO;
	NGImap4ConnectionStringSeparator = ".";

::

	/* SQL authentication Mysql */
	SOGoUserSources = (
	    {
	      type = sql;
	      id = postfixadmin;
	      viewURL = "mysql://bar:foobar@smtp.intra.data.gouv.fr:3306/postfixadmin/sogo_users";
	      canAuthenticate = YES;
	      isAddressBook = YES;
	      userPasswordAlgorithm = "md5-crypt";
	      displayName = "SGMAP/Etalab";
	      DomainFieldName = "domain";
	      IMAPLoginFieldName = "c_name";
	      LoginFieldNames = (
	          "c_uid",
	          "c_name"
	      );
	    }
	  );

::

	/* Web Interface */
	SOGoPageTitle = Webmail-Etalab;
	SOGoVacationEnabled = YES;
	SOGoForwardEnabled = YES;
	SOGoSieveScriptsEnabled = YES;
	SOGoMailMessageCheck = every_minute;
	SOGoSieveScriptsEnable = YES;

:: 

	/* General */
	SOGoLanguage = French;
	SOGoTimeZone = Europe/Paris;
	SOGoMemcachedHost = "127.0.0.1";
	WOPort = 127.0.0.1:20000;


Configuration de sogo pour acceder la db de postfixadmin
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

On créer une vue sur la base de données de postfixadmin ::

    USE postfixadmin;
    
    CREATE VIEW  `sogo_users` AS SELECT local_part AS c_uid, username AS c_name, 
    PASSWORD AS c_password, name AS c_cn, username AS mail, domain
    FROM  `mailbox`;

On peut vérifier cette vue avec les requêtes suivantes ::

    SHOW FULL TABLES IN postfixadmin WHERE TABLE_TYPE LIKE 'VIEW';
    SELECT * FROM sogo_users;

On créer un utilisateur qui sera utilisé par sogo pour acceder à la vue ::

    CREATE USER 'sogo'@'%' IDENTIFIED BY 'fooboar';
    GRANT SELECT ON postfixadmin.sogo_users TO 'sogo'@'%' IDENTIFIED BY 'foobar_password';

Ensuite pour les besoins de sogo, on a besoin d'une base dédiée, que l'on crée :: 

    CREATE DATABASE `sogo` CHARACTER SET='utf8';

Et on y ajoute tous les droits possible ::

    GRANT ALL PRIVILEGES ON `sogo`.* TO 'sogo'@'%' WITH GRANT OPTION;

Pour finir on reload les permissions globales de mysql ::

    FLUSH PRIVILEGES;


Configuration d'apache pour SOGo
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

On active les modules nécessaire au fonctionnement du webmail ::

    a2enmod headers proxy_http proxy rewrite ssl

A des fins d'homogénéité, on lie la configuration du webmail dans /etc/sogo ::

    ln -s /etc/apache2/conf.d/SOgo.conf /etc/sogo/apache.conf

On renseigne les certificats ssl qui seront utilisé par le serveur web ::

    cat < EOF >> /etc/apache2/ssl.conf
    <IfModule mod_ssl.c>
        NameVirtualHost *:443
        SSLCertificateFile /etc/ssl/private/data.gouv.fr-certificates/wildcard.data.gouv.fr-certificate.crt
        SSLCertificateKeyFile /etc/ssl/private/data.gouv.fr-certificates/private-key-raw.key
    </IfModule>
    EOF

On défini un virtual host pour le webmail SOGo ::

    cat < EOF >> /etc/apache2/sites-available/webmail.data.gouv.fr
	<VirtualHost *:80>
	    ServerName webmail.data.gouv.fr
	    ServerAlias mail.data.gouv.fr
	    DocumentRoot /var/www
	
	    RedirectMatch permanent ^/ https://webmail.data.gouv.fr/SOGo
	    RedirectMatch permanent ^/SOGo https://webmail.data.gouv.fr/SOGo
	
	    ErrorLog  /var/log/apache2/webmail.data.gouv.fr.error.log
	    CustomLog /var/log/apache2/webmail.data.gouv.fr.access.log combined
	</VirtualHost>
	
	<VirtualHost *:443>
	    ServerName webmail.data.gouv.fr
	    ServerAlias mail.data.gouv.fr
	    DocumentRoot /var/www
	
	    SSLEngine on
	
	    include /etc/sogo/apache.conf
	
	    ErrorLog  /var/log/apache2/webmail.data.gouv.fr.error-ssl.log
	    CustomLog /var/log/apache2/webmail.data.gouv.fr.access-ssl.log combined
	</VirtualHost>	
	EOF

On active le site  ::

    a2ensite webmail.data.gouv.fr


Autoconfiguration des clients lourds
------------------------------------
Le clients de messagerie que nous recommandons d'utiliser est Mozilla Thunderbird ou son dérivé pour Debian, IceDove. Afin de facilité la configuration de thunderbird pour les utilisateurs finaux, on définit un fichier d'autoconfiguration. Celui-ci sera mis à disponibilité du monde via apache2.

On défini un virtual host pour l'autofiguration ::

vi /etc/apache2/sites-available/autoconfig.data.gouv.fr ::

    <VirtualHost *:80>
    ServerName autoconfig.data.gouv.fr
    DocumentRoot /var/www/autoconfig/public_html

        <Location />
                AddDefaultCharset UTF-8
                php_value magic_quotes_gpc off
                php_value register_globals off
        </Location>

    RedirectMatch ^/$ http://sorry.data.gouv.fr


    ErrorLog  /var/log/apache2/autoconfig.data.gouv.fr.error.log
    CustomLog /var/log/apache2/autoconfig.data.gouv.fr.access.log combined
    </VirtualHost>

On active le site ::

    a2ensite autoconfig.data.gouv.fr

On créer le fichier d'autoconfiguration avec les informations suivantes. ::

    mkdir -p /var/www/autoconfig/public_html/mail

::

    vi /var/www/autoconfig/public_html/mail/config-v1.1.xml
	<clientConfig version="1.1">
	  <emailProvider id="data.gouv.fr">
	    <domain>data.gouv.fr</domain>
	    <displayName>data.gouv.fr - %EMAILLOCALPART%</displayName>
	    <displayShortName>Datagouvfr</displayShortName>
	    <incomingServer type="imap">
	      <hostname>imap.data.gouv.fr</hostname>
	      <port>993</port>
	      <socketType>SSL</socketType>
	      <username>%EMAILADDRESS%</username>
	      <authentication>password-cleartext</authentication>
	    </incomingServer>
	    <outgoingServer type="smtp">
	      <hostname>smtp.data.gouv.fr</hostname>
	      <port>587</port>
	      <socketType>STARTTLS</socketType>
	      <authentication>password-cleartext</authentication>
	      <username>%EMAILADDRESS%</username>
	    </outgoingServer>
	  </emailProvider>
	</clientConfig>

.. note :: Plus d'information https://developer.mozilla.org/en-US/docs/Mozilla/Thunderbird/Autoconfiguration/FileFormat/HowTo


Configuration d'activesync
~~~~~~~~~~~~~~~~~~~~~~~~~~

apt-get install sogo-activesync


