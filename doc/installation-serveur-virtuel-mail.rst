===========================
Installation du server mail
===========================

Le serveur "mail" est utilisé comme stockage de mail. C'est sur ce serveur que les emails sont enregistrés et accessibles pour les clients finaux. Les services configurés pour ce faire sont les services suivants ::

    * Postfix : Pour gérer l'envoi, la réception et le routage des mails (Smtp Smtps)
    * Dovecot : Pour gérer l'accès aux boites mails par les clients finaux (Imaps)
    * SOGo    : Pour gérer un accès par webmail pour les clients finaux (Https)

La réception et l'envoi vis à vis de l'extérieur sont gérés par un serveur smtp tiers, lui aussi avec postifx.

L'installation et la configuration de chaque service vont être exposées ci-après.


.. include:: include-installation-serveur-virtuel-lan.rst

Installation de base
====================

Installation des applications
=============================

Service d'envoi/récéption (Postfix)
-----------------------------------

::

    sed -i 's/inet_interfaces = localhost/inet_interfaces = all/' /etc/postfix/main.cf
    sed -i 's-mynetworks = 127.0.0.0/8-mynetworks = 127.0.0.0/8 10.10.10.7-' /etc/postfix/main.cf
    echo "relayhost = [smtp.intra.data.gouv.fr]" >> /etc/postfix/main.cf

    cat < EOF >> /etc/postfix/main.cf
    # VIRTUAL DOMAIN
	# Aliases
	virtual_alias_maps = proxy:mysql:$config_directory/mysql_virtual_alias_maps.cf
	# Accounts
	virtual_mailbox_domains = proxy:mysql:$config_directory/mysql_virtual_domains_maps.cf
	virtual_mailbox_maps = proxy:mysql:$config_directory/mysql_virtual_mailbox_maps.cf
	
	# Transport
	virtual_transport = dovecot
    dovecot_destination_recipient_limit=1
    EOF


    cat < EOF >> /etc/postfix/master.cf
    dovecot   unix  -       n       n       -       -       pipe
     flags=DRhu user=vmail:mail argv=/usr/lib/dovecot/deliver -d ${recipient}
    EOF

Activer SASL pour le smtp
~~~~~~~~~~~~~~~~~~~~~~~~~
/etc/postfix/

master.cf
submission inet n       -       -       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject

main.cf
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

#
# SMTPd check
#
smtpd_recipient_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_unauth_destination
smtpd_sender_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_non_fqdn_sender, reject_unknown_sender_domain


/etc/dovecot/
conf.d/10-master.conf
  unix_listener /var/spool/postfix/private/auth {
    mode = 0666
    user = postfix
    group = postfix
  }

conf.d/10-auth.conf
     auth_mechanisms = plain login



Service de gestion des boites mails 
-----------------------------------
Installation de dovecot et des services associés
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::
   
  apt-get install dovecot-common dovecot-mysql dovecot-imapd dovecot-managesieved dovecot-sieve


Ajout des certificats SSL
~~~~~~~~~~~~~~~~~~~~~~~~~
::
  
    cd /etc/ssl/private/
    git clone git@git.intra.data.gouv.fr:certificates/data.gouv.fr-certificates.git
    git clone git@git.intra.data.gouv.fr:certificates/openfisca.fr-certificates.git
    chmod -R 640 * && chown -R :ssl-cert *

Préparation du filesystem
~~~~~~~~~~~~~~~~~~~~~~~~~
::
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
::

On définit les parametres du daemon dovecot dans le fichier **10-master.conf**

.. note:: D'autres valeurs sont définies par défaut et on les laisse telles quelles. Néanmoins on commente pop3 qui ne sera pas utilisé ici. 

:: 

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
    

On définit les parametres relatifs à la configuration des fichiers stockant les boites mails. Leurs emplacements, leurs types. Pour ce faire on edite le fichier **10-mail.conf**

On définit des boites mails au format Maildir ::

    mail_location = maildir:~/Maildir
    namespace inbox {
        type = private
        separator = .
        inbox = yes
    }
    
On définit avec quel utilisateur le daemon dovecot va acceder aux Maildir ::

    mail_uid = vmail
    mail_gid = mail






On modifie le processus d'autentification de dovecot, en modifiant les valeurs ci-dessous dans le fichier **10-auth.conf** ::

    disable_plaintext_auth = no
    auth_mechanisms = plain
    !include auth-sql.conf.ext



On renseigne les informations concernant les certificats ssl à utiliser dans le fichier **10-ssl.conf** ::

    ssl = yes
    ssl_cert = </etc/ssl/private/data.gouv.fr-certificates/wildcard.data.gouv.fr-certificate.crt
    ssl_key = </etc/ssl/private/data.gouv.fr-certificates/private-key-raw.key




On crée le fichier de configuration necessaire à la connexion a mysql et on positionne les droits correctement :: 

    chmod 0600 dovecot-sql.conf.ext

On edite **dovecot-sql.conf.ext** et on renseigne les informations suivantes ::

	driver = mysql
	connect = host=smtp.intra.data.gouv.fr dbname=postfixadmin user=foobar password=foobar
	default_pass_scheme = MD5
	
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



Configuration du service de filtre
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Le service de filtre est nécessaire pour gérer les mails d'autoréponses que sogo va générer dans le cas d'une période de vacances pour l'utilisateur.

On active donc sieve via les fichiers suivants :

Pour **conf.d/20-managesieve.conf** 

.. note:: D'autres valeurs sont définies par défaut et on les laisse telles quelles.

::
    service managesieve-login {
    inet_listener sieve {
        port = 4190
    }
    service_count = 1
    }

Pour **15-lda.conf** ::

    protocol lda {
    # Space separated list of plugins to load (default is global mail_plugins).
    mail_plugins = $mail_plugins sieve
    }

Pour **90-sieve.conf** ::

    plugin {
        #sieve = ~/.dovecot.sieve
        sieve_dir = ~/sieve
    }

On redemarre le service dovecot ::
    
    service dovecot restart

Service de webmail
------------------

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

On crée une vue sur la base de données de postfixadmin ::

    USE postfixadmin;
    
    CREATE VIEW  `sogo_users` AS SELECT local_part AS c_uid, username AS c_name, 
    PASSWORD AS c_password, name AS c_cn, username AS mail, domain
    FROM  `mailbox`;

On peut vérifier cette vue avec les requêtes suivantes ::

    SHOW FULL TABLES IN postfixadmin WHERE TABLE_TYPE LIKE 'VIEW';
    SELECT * FROM sogo_users;

On crée un utilisateur qui sera utilisé par sogo pour acceder à la vue ::

    CREATE USER 'sogo'@'%' IDENTIFIED BY 'fooboar';
    GRANT SELECT ON postfixadmin.sogo_users TO 'sogo'@'%' IDENTIFIED BY 'foobar_password';

Ensuite pour les besoins de sogo, on a besoin d'une base dédiée, que l'on crée :: 

    CREATE DATABASE `sogo` CHARACTER SET='utf8';

Et on y ajoute tous les droits possible ::

    GRANT ALL PRIVILEGES ON `sogo`.* TO 'sogo'@'%' WITH GRANT OPTION;

Pour finir on reload les permissions globales de mysql ::

    FLUSH PRIVILEGES;


Configuration d'apache
~~~~~~~~~~~~~~~~~~~~~~

a2enmod headers proxy_http proxy rewrite ssl

mv /etc/apache2/conf.d/SOgo.conf /etc/sogo/apache.conf

cat < EOF >> /etc/apache2/ssl.conf
<IfModule mod_ssl.c>
    NameVirtualHost *:443
    SSLCertificateFile /etc/ssl/private/data.gouv.fr-certificates/wildcard.data.gouv.fr-certificate.crt
    SSLCertificateKeyFile /etc/ssl/private/data.gouv.fr-certificates/private-key-raw.key
</IfModule>
EOF

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

On edite le fichier **/etc/apache2/sites-available/webmail.data.gouv.fr**

a2ensite webmail.data.gouv.fr


Autoconfiguration des clients lourds
------------------------------------

/etc/apache2/sites-available/autoconfig.data.gouv.fr
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

a2ensite autoconfig.data.gouv.fr


mkdir -p /var/www/autoconfig/public_html/mail

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

https://developer.mozilla.org/en-US/docs/Mozilla/Thunderbird/Autoconfiguration/FileFormat/HowTo


Configuration d'activesync
~~~~~~~~~~~~~~~~~~~~~~~~~~

apt-get install sogo-activesync


