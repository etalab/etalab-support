===========================================
How to installation d'un serveur mail Libre 
===========================================

Préambule
=========
Le service Etalab au sein du SGMAP (Secrétariat Général de la Modernisation de l'Action Publique) souhaite, dans le cadre de l'opendata, mettre à disposition du public un certain nombre de documentations relatives à son Système d'Information.

Cette documentation technique a pour but de montrer comment le SGMAP/Etalab assure son service de messagerie de manière autonome.

Les logiciels utilisés sont tous sans exception des Logiciels Libres. Ils assurent les fonctionnalités modernes et nécessaires aux agents d'Etalab en termes de messagerie. (Accès de n'importe où à la messagerie, synchronisation des mails,contacts,agendas avec les smartphones, partage d'agendas,etc..)

Introduction
============
Tous les services présents dans ce document sont installés sous Linux. La distribution utilisée est Debian GNU/Linux dans sa dernière version à l'heure actuelle, Wheezy.

Les logiciels utilisés pour assurer les fonctionnalités voulues sont les suivants ::

    * Postfix : Pour assurer l'envoi, la réception et le routage des mails (Protocols : Smtp et Smtps)
    * Mysql   : Pour unifier les comptes des utilisateurs de la plateforme dans une base de données SQL.
    * Postfixadmin : Pour assurer une administration simplifiée des comptes utilisateurs. (Protocol : Https)
    * Dovecot : Pour assurer l'accès aux boites mails par les clients finaux (Protocol : Imaps)
    * SOGo    : Pour assurer un accès par webmail pour les clients finaux (Protocol: Https)

L'installation et la configuration de chaque service vont être exposées ci-après.

Installation de la base du système
==================================

.. note :: Le serveur virtuel se base sur une installation standard Etalab suivant la procédure **include-installation-serveur-virtuel-lan.rst**

Installation des applications
=============================

Service d'envoi/réception (Postfix)
-----------------------------------

Le serveur postfix va assurer les fonctions suivantes :

    - Envoi du courrier pour les services présents sur ce serveur,
    - Réception des emails des utilisateurs,
    - Envoi du courrier pour les utilisateurs authentifiés,
    - Authentifier les utilisateurs avec SASL via Dovecot,

On installe le serveur postfix. ::

    apt-get install postfix

On modifie la configuration par défaut avec les informations relatives à l'infrastructure ::

    sed -i 's/inet_interfaces = localhost/inet_interfaces = all/' /etc/postfix/main.cf
    sed -i 's-mynetworks = 127.0.0.0/8-mynetworks = 127.0.0.0/8, [::1], $YOUR_SERVER_IP-' /etc/postfix/main.cf
    sed -i 's/#recipient_delimiter = +/recipient_delimiter = +/' /etc/postfix/main.cf
    echo "relayhost = [smtp.intra.data.gouv.fr]" >> /etc/postfix/main.cf

On ajoute un compte de réception pour les mails root ::
   
    echo "root: admin@data.gouv.fr" >> /etc/aliases

On compile aliases.db ::

    newaliases

On démarre postfix ::

    service postfix restart

.. note:: A ce stade vous pouvez, depuis le serveur, envoyer des mails vers l'extérieur.

Service de base de données (Mysql)
---------------------------------

Mysql va nous permettre d'héberger une base de données contenant les utilisateurs virtuels.

On génère un mot de passe via **pygen -y 12** afin renseigner le password root de mysql qui nous seras demandé pendant l'installation.

On installe le serveur Mysql ::

  apt-get install mysql-server

On définit le fichier  **/root/.my.cnf** avec ::
    
    [mysql]
    user=root
    password=FOO_PASSWORD

Dans le souci d'un mimimum d'optimisation, on définit l'option suivante dans la configuration de mysql. ::

  echo "innodb_file_per_table = 1" >> /etc/mysql/my.cnf


On restart mysql ::

    service mysql restart

.. note:: A ce stade on peut se connecter depuis le shell et via la commande mysql au server mysql.

Certificat SSL
--------------
En fonction de votre besoin, il est possible d'utiliser plusieurs types de certificats. J'en décris ici deux types; les certificats autosignés et les certificats validés par une autorité de certifications tierce et payante.

Création d'un certificats SSL autosigné
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

On créé une paire de clés au format RSA ::
    
    cd /etc/ssl/private/certificates/foobar.fr
    openssl genrsa -out foobar.key 2048
    
On génère ensuite le certificat autosigné ::

    openssl req -new -x509 -days 3650 -key foobar.key -out foobar.crt


Ajout d'un certificat proventant d'une autorité de certification tierce
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Les certificats d'Etalab sont stockés sur un serveur Git interne. ::
  
    cd /etc/ssl/private
    git clone git@git.intra.data.gouv.fr:certificates/data.gouv.fr-certificates.git
    git clone git@git.intra.data.gouv.fr:certificates/openfisca.fr-certificates.git
    chmod -R 640 * && chown -R :ssl-cert *


Service d'administration web des comptes de messagerie (Postfixadmin)
---------------------------------------------------------------------
Installation d'apache2 
~~~~~~~~~~~~~~~~~~~~~~
Un serveur web est nécessaire pour l'interface de postfixadmin.

On installe apache ::
    
    apt-get install apache2

On active les modules nécessaires ::

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

On applique quelques modifications à la configuration de base ::

    sed -i 's/ServerTokens OS/ServerTokens Prod/' /etc/apache2/conf.d/security
    sed -i 's/ServerSignature On/ServerSignature Off/' /etc/apache2/conf.d/security

On redémarre apache2 ::

    service apache2 restart



Installation de postfixadmin
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
On installe le service Postfixadmin ::

  apt-get install postfixadmin postfix-mysql

.. note:: On n'utilise pas dbconfig pour configurer postfixadmin

::

    Web server to reconfigure automatically =>  no
    Configure database for postfixadmin with dbconfig-common? => no

Une fois l'installation faite, on vérifie les prérequis via le setup.php ::

    https://pfa.data.gouv.fr/setup.php


On configure la base de données ::

    mysql> CREATE DATABASE 'postfixadmin' CHARACTER SET='utf8';
    mysql> GRANT ALL PRIVILEGES ON `postfixadmin`.* TO 'postfix'@'localhost' IDENTIFIED BY 'foobar';


On configure le fichier de configuration relatif à la base de données:

vi /etc/postfixadmin/dbconfig.inc.php ::

    <?php
    $dbuser='postfix';
    $dbpass='foobar';
    $dbname='postfixadmin';
    $dbserver='localhost';
    $dbport='';
    $dbtype='mysqli';


On créé la base de données via le setup.php ::

    https://pfa.data.gouv.fr/setup.php


.. note:: On peut configurer quelques éléments facultatifs via le fichier de configuration **/etc/postfixadmin/config.inc.php**

Configuration de postfix (virtual)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

On définit les requêtes sql que postfix devra effectuer pour lister les comptes emails présents :

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

vi /etc/postfix/mysql_virtual_alias_domain_maps.cf ::

      user            = postfix
      password        = **********************
      hosts           = localhost
      dbname          = postfixadmin
      query           = SELECT goto FROM alias,alias_domain WHERE alias_domain.alias_domain = '%d' and alias.address = CONCAT('%u', '@', alias_domain.target_domain) AND alias.active = 1 AND alias_domain.active='1'


.. note :: Plus de documentation ici -> /usr/share/doc/postfixadmin/DOCUMENTS/POSTFIX_CONF.txt.gz 

On ajoute la configuration relative aux utilisateurs virtuels de postfix ::

    cat < EOF >> /etc/postfix/main.cf
    # VIRTUAL DOMAIN
    # Aliases
    virtual_alias_maps = proxy:mysql:$config_directory/mysql_virtual_alias_maps.cf,proxy:mysql:$config_directory/mysql_virtual_alias_domain_maps.cf
    # Accounts
    virtual_mailbox_domains = proxy:mysql:$config_directory/mysql_virtual_domains_maps.cf
    virtual_mailbox_maps = proxy:mysql:$config_directory/mysql_virtual_mailbox_maps.cf
    EOF

.. note:: A ce stade, il est possible de créer des utilisateurs, mais leurs boites mail ne seront pas fonctionnelles. Il faut maintenant un service capable de stocker les mails. 


Service de gestion des boites mails (Dovecot)
---------------------------------------------

Le service dovecot va assurer l'interface entre la base de mails au format MailDir et les clients de messagerie des utilisateurs finaux. Le protocole servi pour ce faire, sera uniquement l'IMAPS.

En association avec managesieve, dovecot permettra également aux utilisateurs de gérer des filtres de messages.

L'authentification des utilisateurs se fait sur la base de données Mysql.

Installation de dovecot et des services associés
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
On installe les services relatifs au fonctionnement de dovecot ::
   
  apt-get install dovecot-common dovecot-mysql dovecot-imapd dovecot-managesieved dovecot-sieve


Préparation du filesystem
.........................
On définit un volume dédié au stockage des mails afin d'éviter le blocage du système en cas de remplissage complet du file system. ::

    lvcreate -L 20g -n vmail vg00
    mkfs.ext4 /dev/vg00/vmail
    mkdir /srv/vmail
    echo "/dev/mapper/vg00-vmail /srv/vmail     ext4    defaults        0   2" >> /etc/fstab
    mount -a
    mkdir /srv/vmail/users


Création d'un utilisateur pour dovecot
......................................

::

    useradd -m -s /bin/false -d /srv/vmail vmail
    chown -R vmail:mail /srv/vmail


Configuration du service imap
.............................

On définit les paramètres du daemon dovecot.

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



On définit les paramètres relatifs à la configuration des fichiers stockant les boites mails. Leurs emplacements, leurs types. Pour ce faire, on edite le fichier **10-mail.conf**

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


On modifie le processus d'autentification de dovecot, en modifiant les valeurs ci-dessous dans le fichier **10-auth.conf**.

vi /etc/dovecot/conf.d/10-auth.conf ::

    disable_plaintext_auth = no
    auth_mechanisms = plain
    !include auth-sql.conf.ext


On renseigne les informations concernant les certificats ssl à utiliser dans le fichier **10-ssl.conf**.

vi /etc/dovecot/conf.d/10-ssl.conf ::

    ssl = yes
    ssl_cert = </etc/ssl/private/data.gouv.fr-certificates/wildcard.data.gouv.fr-certificate.crt
    ssl_key = </etc/ssl/private/data.gouv.fr-certificates/private-key-raw.key


On crée le fichier de configuration necessaire à la connexion à mysql et on positionne les droits correctement ::

    chmod 0600 /etc/dovecot/dovecot-sql.conf.ext

On edite **dovecot-sql.conf.ext** et on renseigne les informations suivantes.

vi /etc/dovecot/dovecot-sql.conf.ext ::

    driver = mysql
    [...]
    connect = host=localhost dbname=postfixadmin user=foobar password=foobar
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
    FLUSH PRIVILEGES;

Vérification ::

    mysql -udovecot -p


Déclaration du transport dovecot dans postifx
.............................................

On déclare un service dovecot pour postfix ::

    cat < EOF >> /etc/postfix/master.cf
    dovecot   unix  -       n       n       -       -       pipe
     flags=DRhu user=vmail:mail argv=/usr/lib/dovecot/dovecot-lda -f ${sender} -a ${recipient} -d ${user}@${nexthop}
    EOF

On route les mails vers dovecot afin qu'ils soient stockés ::

    cat < EOF >> /etc/postfix/main.cf
    # Transport
    virtual_transport = dovecot
    dovecot_destination_recipient_limit=1
    EOF


Configuration du service de filtre (ManageSieve)
................................................

Le service de filtre est nécessaire pour gérer, par exemple, les mails d'auto-réponses que sogo va générer dans le cas d'une période de vacances pour l'utilisateur.

On active donc sieve via les fichiers suivants :

.. note:: D'autres valeurs sont définies par défaut et on les laisse telles quelles.

vi /etc/dovecot/conf.d/20-managesieve.conf ::

    service managesieve-login {
    inet_listener sieve {
        port = 4190
    }
    service_count = 1
    }

    service managesieve {
    }

    protocol sieve {
    }

vi /etc/dovecot/conf.d/15-lda.conf ::

    recipient_delimiter = +

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

.. note:: A ce stade, les utilisateurs ayant des boite mails, peuvent stocker leurs mails, mais on testera ce fonctionnement après. 


Service d'envoi de mail authentifié (SMTPS)
-------------------------------------------

SASL va être utilisé pour authentifier les utilisateurs de notre organisation, afin que seulement ceux-ci puissent envoyer des emails par le biais de notre serveur de mails.

Les fonctionnalités SASL vont être activées uniquement sur le port submission(587) prévu par la rfc6409.

En outre, nous avons choisi d'authentifier nos utilisateurs via dovecot qui lui même s'appuie sur la base mysql comme base de données utilisateur. Cette réalisation est triviale et évite les multiples configurations de mysql en backend des services postfix & co.

On configure les fonctionnalités SASL de postfix.

vi /etc/postfix/master.cf ::

    submission inet n       -       -       -       -       smtpd
    -o syslog_name=postfix/submission
    -o smtpd_tls_security_level=encrypt
    -o smtpd_sasl_auth_enable=yes
    -o smtpd_client_restrictions=permit_sasl_authenticated,reject

vi /etc/postfix/main.cf ::

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

::

    #
    # SMTPd check
    #
    smtpd_recipient_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_unauth_destination
    smtpd_sender_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_non_fqdn_sender, reject_unknown_sender_domain


Délégation à dovecot
~~~~~~~~~~~~~~~~~~~~

La gestion de l'authentification des utilisateurs est déléguée à dovecot. On active une socket unix sur le serveur dovecot pour que postfix puisse l'interroger.

.. warning :: Les paramètres de configuration suivants sont liés au serveur dovecot. Néanmoins, dans un souci de compréhension, ils sont définis à cet endroit de la documentation. 

vi /etc/dovecot/conf.d/10-master.conf ::

  service auth {

  # Postfix smtp-auth
  unix_listener /var/spool/postfix/private/auth {
    mode = 0666
    user = postfix
    group = postfix
  }

vi /etc/dovecot/conf.d/10-auth.conf ::

     auth_mechanisms = plain login

On relance les services postfix et dovecot ::

    service postfix restart ; service dovecot restart


Autoconfiguration des clients lourds
------------------------------------
Le client de messagerie que nous recommandons d'utiliser est Mozilla Thunderbird ou son dérivé pour Debian, IceDove. 
Afin de faciliter la configuration de thunderbird pour les utilisateurs finaux, on définit un fichier d'autoconfiguration. Celui-ci sera mis à disponibilité du monde via apache2.

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

vi /var/www/autoconfig/public_html/mail/config-v1.1.xml ::

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

.. note :: Plus d'information https://developer.mozilla.org/en-US/docs/Mozilla/Thunderbird/Autoconfiguration/FileFormat/HowT


Vérification de fonctionnement des services installés à ce stade
================================================================

Connexion à IMAPS
-----------------
On se connecte en imaps via netcat ::

  openssl s_client -connect imap.data.gouv.fr:993

  * OK [CAPABILITY IMAP4rev1 LITERAL+ SASL-IR LOGIN-REFERRALS ID ENABLE IDLE AUTH=PLAIN AUTH=LOGIN] Dovecot ready.

  __a login felix@data.gouv.fr foobar_password

  __a OK [CAPABILITY IMAP4rev1 LITERAL+ SASL-IR LOGIN-REFERRALS ID ENABLE IDLE SORT SORT=DISPLAY THREAD=REFERENCES THREAD=REFS MULTIAPPEND UNSELECT CHILDREN NAMESPACE UIDPLUS LIST-EXTENDED I18NLEVEL=1 CONDSTORE QRESYNC ESEARCH ESORT SEARCHRES WITHIN CONTEXT=SEARCH LIST-STATUS SPECIAL-USE] Logged in

  __a list "" *__
  __a OK List completed.
  __a logout

Augmenter la verbosité de dovecot 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

vi /etc/dovecot/conf.d/10-logging.conf ::

   Check around  ## Logging verbosity and debugging. ;)


Troubleshot
~~~~~~~~~~~

Erreur corrigée en exécutant : newaliases ::

  Jun 17 11:59:50 mail postfix/local[25585]: warning: hash:/etc/aliases is unavailable. open database /etc/aliases.db: No such file or directory

Erreur corrigée en supprimant la résolution dns sur ipv6 dans postfix ::

    mynetworks = 127.0.0.0/8, [::1], $IP1, $IP2.. 

    Jun 17 12:15:53 mail postfix/smtpd[25692]: warning: hostname localhost does not resolve to address ::1: No address associated with hostname

Erreur corrigée en modifiant la requête sql de dovecot.

vi /etc/dovecot/dovecot-sql.conf.ext :: 

    10001 AS uid, 8 AS gid 
    au lieu de 
    1000 AS uid, 8 AS gid

    Jun 17 18:04:01 mail dovecot: imap(felix@data.gouv.fr): Error: user felix@data.gouv.fr: Initialization failed: Namespace '': mkdir(/srv/vmail/users/felix@data.gouv.fr) failed: Permission denied (euid=1000(<unknown>) egid=8(mail) missing +w perm: /srv/vmail/users, dir owned by 10001:8 mode=0755)


Envoi de mail via SMTPS
-----------------------
::
    ./smtpt -H mail.data.gouv.fr -f felix@data.gouv.fr -t felix@data.gouv.fr -T -v -p 587 -U felix@data.gouv.fr -P foobar

Vérifier la présence de nouveau mail dans ::

    /srv/vmail/users/felix@data.gouv.fr/new/


Troubleshot
~~~~~~~~~~~

Erreur résolue en permettant à la machine de résoudre son propre nom de domaine fqdn. Pour le savoir on peut exécuter la commande ::

    hostname -f

    Jun 17 19:03:31 mail postfix/pipe[28998]: BF63F179: to=<felix@data.gouv.fr>, relay=dovecot, delay=2498, delays=2498/0.01/0/0.04, dsn=4.3.0, status=deferred (temporary failure. Command output: lda: Error: user felix@data.gouv.fr: Error reading configuration: Invalid settings: postmaster_address setting not given lda: Fatal: Internal error occurred. Refer to server log for more information. )

Augmenter la verbosité de smtpsubmission
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

    submission inet n       -       -       -       -       smtpd -vv


Connexion a sieve
-----------------
::

    sieve-connect --nosslverify --user felix@data.gouv.fr  mail.data.gouv.fr -p 4190



Installation du webmail SOGo
============================
Pour l'installation de sogo, nous allons suivre les étapes ci-dessous. En plus de sogo lui-même, on installera également les dépendances nécessaires.

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
-----------------------------

On édite le fichier **/etc/sogo/sogo.conf**

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

Configuration de sogo pour accéder à la db de postfixadmin
----------------------------------------------------------
On crée une vue de la base de données de postfixadmin ::

    USE postfixadmin;
    
    CREATE VIEW  `sogo_users` AS SELECT local_part AS c_uid, username AS c_name, 
    PASSWORD AS c_password, name AS c_cn, username AS mail, domain
    FROM  `mailbox`;

On peut vérifier cette vue avec les requêtes suivantes ::

    SHOW FULL TABLES IN postfixadmin WHERE TABLE_TYPE LIKE 'VIEW';
    SELECT * FROM sogo_users;

On crée un utilisateur qui sera utilisé par sogo pour accéder à la vue ::

    CREATE USER 'sogo'@'%' IDENTIFIED BY 'fooboar';
    GRANT SELECT ON postfixadmin.sogo_users TO 'sogo'@'%' IDENTIFIED BY 'foobar_password';

Ensuite pour les besoins de sogo, on a besoin d'une base dédiée, que l'on crée :: 

    CREATE DATABASE `sogo` CHARACTER SET='utf8';

Et on y ajoute tous les droits possibles ::

    GRANT ALL PRIVILEGES ON `sogo`.* TO 'sogo'@'%' WITH GRANT OPTION;

Pour finir on reload les permissions globales de mysql ::

    FLUSH PRIVILEGES;


Configuration d'apache pour SOGo
--------------------------------

On active les modules nécessaires au fonctionnement du webmail ::

    a2enmod headers proxy_http proxy rewrite ssl

A des fins d'homogénéité, on lie la configuration du webmail dans /etc/sogo ::

    ln -s /etc/apache2/conf.d/SOgo.conf /etc/sogo/apache.conf

On renseigne les certificats ssl qui seront utilisés par le serveur web ::

    cat < EOF >> /etc/apache2/ssl.conf
    <IfModule mod_ssl.c>
        NameVirtualHost *:443
        SSLCertificateFile /etc/ssl/private/data.gouv.fr-certificates/wildcard.data.gouv.fr-certificate.crt
        SSLCertificateKeyFile /etc/ssl/private/data.gouv.fr-certificates/private-key-raw.key
        SSLCertificateChainFile /etc/ssl/private/data.gouv.fr-certificates/ca-wildcard-certificate-chain.crt
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


Configuration d'activesync
--------------------------
::
     apt-get install sogo-activesync

Configuration des backups des utilisateurs sogo
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Afin de sauvegarder les données de profil de chaque utilisateur, on doit sauvegarder les utilisateurs sogo via un outil dédié.

On définit l'emplacement des sauvegardes
:: 

    mkdir /var/backups/sogo
    chown sogo: /var/backups/sogo

On définit une fréquence de sauvegarde
vi /etc/cron.d/sogo-backup ::

    23 23 * * * sogo /usr/local/bin/sogo-backup    

On crée le script de sauvegarde suivant :
vi /usr/local/bin/sogo-backup ::

    #!/bin/bash
    VALID_USER=sogo
    BACKUP_DIR=/var/backups/sogo
    NB_DAY_RETENTION=15
    USER=$( getent passwd $( id -u ) |cut -d':' -f 1 )  
    [ "$USER" != "$VALID_USER" ] && echo "This script must be run by user $VALID_USER (current : $USER)" && exit 1
    DATE=$( date +%Y%m%d-%Hh%M )
    DIR=$BACKUP_DIR/$DATE
    LOG=$DIR/backup.log
    [ ! -d "$DIR" ] && mkdir "$DIR"
    /usr/sbin/sogo-tool backup "$DIR" ALL > $LOG 2>&1
    find $BACKUP_DIR/* -type d -ctime +$NB_DAY_RETENTION -exec rm -fr \(\) \;
    cat $LOG|grep -v "Cache cleanup interval set every"|grep -v "Using host(s)"

On applique les droits nécessaires :
::

    chown sogo: /usr/local/bin/sogo-backup
    chmod a+x /usr/local/bin/sogo-backup

C'est fini. 
