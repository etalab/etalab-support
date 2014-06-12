
A noter que la réception et l'envoi vis à vis du monde extérieur, sont gérés par un serveur smtp tiers, lui aussi avec postifx.






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

On définit un virtual host pour le webmail SOGo ::

    cat < EOF >> /etc/apache2/sites-available/webmail.data.gouv.fr
	<VirtualHost *:80>
	    ServerName webmail.data.gouv.fr
	    ServerAlias mail.data.gouv.fr
	    DocumentRoot /var/www
	
	    RedirectMatch permanent ^(.*)$ https://webmail.data.gouv.fr$1
	
	    ErrorLog  /var/log/apache2/webmail.data.gouv.fr.error.log
	    CustomLog /var/log/apache2/webmail.data.gouv.fr.access.log combined
	</VirtualHost>
	
	<VirtualHost *:443>
	    ServerName webmail.data.gouv.fr
	    ServerAlias mail.data.gouv.fr
	    DocumentRoot /var/www

        RedirectMatch ^/$ https://webmail.data.gouv.fr/SOGo
	
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
Le clients de messagerie que nous recommandons d'utiliser est Mozilla Thunderbird ou son dérivé pour Debian, IceDove. 
Afin de facilité la configuration de thunderbird pour les utilisateurs finaux, On définit un fichier d'autoconfiguration. Celui-ci sera mis à disponibilité du monde via apache2.

On définit un virtual host pour l'autofiguration.

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

On crée le fichier d'autoconfiguration avec les informations suivantes. ::

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



Installation de l'antispam et antivirus
---------------------------------------
::

    apt-get install amavisd-new spamassassin re2c clamav clamav-daemon pyzor razor altermime

Configuration d'Amavisd-new
~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. note:: La documentation d'amavis est ici **/usr/share/doc/amavisd-new/RELEASE_NOTES.gz**

/etc/amavis/conf.d/50-user ::

Installation des decoders pour amavis ::

	No decoder for       .F
	No decoder for       .lzo  tried: lzop -d
	No decoder for       .rpm  tried: rpm2cpio.pl, rpm2cpio
	No decoder for       .deb  tried: ar
	No decoder for       .7z   tried: 7zr, 7za, 7z
	No decoder for       .rar  tried: unrar-free
	No decoder for       .arj  tried: arj, unarj
	No decoder for       .arc  tried: nomarch, arc
	No decoder for       .zoo  tried: zoo
	No decoder for       .doc  tried: ripole
	No decoder for       .cab  tried: cabextract
	No decoder for       .tnef
	No decoder for       .exe  tried: unrar-free; arj, unarj


    apt-get install p7zip arj arc zoo ripole cabextract rpm2cpio lzop unrar-free binutils

Configuration de spamassassin
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Spamassassin est appelé par amavis, on s'assure qu'il ne démarre pas en tant que daemon. 

Mise à jour des regles chaque nuit ::

    sed -i s/CRON=0/CRON=1/ /etc/default/spamassassin


On active de l'optimisation via re2c
vi /etc/spamassassin/v320.pre et décommenter cette ligne ::

    loadplugin Mail::SpamAssassin::Plugin::Rule2XSBody

.. note:: http://spamassassin.apache.org/full/3.2.x/doc/sa-compile.html



Configuration de Pyzor & Razor
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Verifier l'activation des modules dans **/etc/spamassassin/v310.pre**

Autoriser les flux via iptables ::

    # -------------------------- Pyzor & Razor
    iptables -A FORWARD -p udp -i $IFLAN -s $SMTP_LAN10  --dport 24441 -o $IFEXT0 -j ACCEPT
    iptables -A FORWARD -p tcp -i $IFLAN -s $SMTP_LAN10  --dport 2703 -o $IFEXT0 -j ACCEPT


Configuration de clamAV
~~~~~~~~~~~~~~~~~~~~~~~
On install clamav en mode daemon, pour qu'il télécharge les updates automatiquement. 
