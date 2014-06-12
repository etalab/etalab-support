==========================================
How to installation d'un server mail Libre 
==========================================

Préambule
=========
Le service Etalab au sein du SGMAP (Secrétariat Général de la Modernisation de l'Action Publique) à décidé de mettre à disposition du publique un certain nombre de documentation relatif à son Système d'Information.

Cette documentation technique à pour but de montrer comment le SGMAP assure son service de messagerie.

Introduction
============
Tous les services présents dans ce document fonctionnent avec un seul serveur virtuel comme hôte d'hébergement. On a nommé ce serveur, "mail".

La distribution Linux utilisé est Debian GNU/Linux dans sa dernière version à l'heure actuelle, Wheezy.

Les logiciels utilisés pour assurer les fonctionnalités voulu sont les suivants ::

    * Postfix : Pour assurer l'envoi, la réception et le routage des mails (Protocols : Smtp et Smtps)
    * Mysql   : Pour unifier les comptes des utilisateur de la plateforme dans une base de donnée SQL.
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

    - Envoi du courier pour les services présents sur ce serveur,
    - Réception des emails des utilisateurs,
    - Envoi du courier pour les utilisateur authentifiés,
    - Authentifier les utilisateurs avec SASL via Dovecot,

On installe le serveur postfix. ::

    apt-get install postfix

On modifie la configuration par défaut avec les informations relatif à l'infrastructure ::

    sed -i 's/inet_interfaces = localhost/inet_interfaces = all/' /etc/postfix/main.cf
    sed -i 's-mynetworks = 127.0.0.0/8-mynetworks = 127.0.0.0/8 $YOUR_SERVER_IP-' /etc/postfix/main.cf
    sed -i 's/#recipient_delimiter = +/recipient_delimiter = +/' /etc/postfix/main.cf
    echo "relayhost = [smtp.intra.data.gouv.fr]" >> /etc/postfix/main.cf

On démarre postfix ::

    service postfix restart

.. note:: A ce stade vous pouvez, depuis le serveur, envoyer des mails vers l'extérieur.

Service de base de donnée (Mysql)
---------------------------------

Mysql va nous permettre d'héberger une base de données contenant les utilisateurs virtuels.

On génère un mot de passe via **pygen -y 12** afin renseigner le password root de mysql qui nous seras demandé pendant l'installation.

On installe le serveur Mysql ::

  apt-get install mysql-server

On défini le fichier  **/root/.my.cnf** avec ::
    
    [mysql]
    user=root
    password=FOO_PASSWORD

Dans le soucis d'un mimimum d'optimisation, On définit l'option suivante dans la configuration de mysql. ::

  echo "innodb_file_per_table = 1" >> /etc/mysql/my.cnf


On restart mysql ::

    service mysql restart

.. note:: A ce stade on peut se connecter depuis le shell et via la commande mysql au server mysql.

Certificat SSL
--------------
En fonction de votre besoin, il est possible utiliser plusieurs type de certificats. Je décrit ici deux type, les certificats autosigné et les autres.

Création d'un certificats SSL autosigné
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

On créer une pair de clé au format RSA ::
    
    cd /etc/ssl/private/certificates/foobar.fr
    openssl genrsa -out foobar.key 2048
    
On génère ensuite le certificat autosigné ::

    openssl req -new -x509 -days 3650 -key foobar.key -out foobar.crt


Ajout des certificats SSL
~~~~~~~~~~~~~~~~~~~~~~~~~
Les certificats d'Etalab sont stockés sur un serveur Git interne. Ils sont signé par une autorité de certification tierce.

::
  
    cd /etc/ssl/private/
    git clone git@git.intra.data.gouv.fr:certificates/data.gouv.fr-certificates.git
    git clone git@git.intra.data.gouv.fr:certificates/openfisca.fr-certificates.git
    chmod -R 640 * && chown -R :ssl-cert *



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

On applique quelques optimisations ::

    sed -i 's/ServerTokens OS/ServerTokens Prod/' /etc/apache2/conf.d/security
    sed -i 's/ServerSignature On/ServerSignature Off/' /etc/apache2/conf.d/security

On redémarre apache2 ::

    service apache2 restart



Installation de postfixadmin
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
On installe le service Postfixadmin ::

  apt-get install postfixadmin postfix-mysql

.. note:: On utilisera pas dbconfig pour configurer postfixadmin

::

    Web server to reconfigure automatically =>  no
    Configure database for postfixadmin with dbconfig-common? => no

Une fois l'installation faite, on vérifie les prérequis via le setup.phg ::

    https://pfa.data.gouv.fr/setup.php


On configure la base de donnée ::

    mysql> CREATE DATABASE 'postfixadmin' CHARACTER SET='utf8';
    mysql> GRANT ALL PRIVILEGES ON 'postfixadmin'.* TO 'postfix'@'localhost' IDENTIFIED BY 'foobar';


On configure le fichier de configuration relatif à la base de donnée ::

vi /etc/postfixadmin/dbconfig.inc.php ::

    <?php
    $dbuser='postfix';
    $dbpass='foobar';
    $dbname='postfixadmin';
    $dbserver='localhost';
    $dbport='';
    $dbtype='mysqli';


On créer la base de donnée via le setup.php ::

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


.. note :: Plus de documentation ici -> /usr/share/doc/postfixadmin/DOCUMENTS/POSTFIX_CONF.txt.gz 

On ajoute la configuration relative aux utilisateurs virtuels de postfix ::

    cat < EOF >> /etc/postfix/main.cf
    # VIRTUAL DOMAIN
    # Aliases
    virtual_alias_maps = proxy:mysql:$config_directory/mysql_virtual_alias_maps.cf
    # Accounts
    virtual_mailbox_domains = proxy:mysql:$config_directory/mysql_virtual_domains_maps.cf
    virtual_mailbox_maps = proxy:mysql:$config_directory/mysql_virtual_mailbox_maps.cf
    EOF

.. note:: A ce stade, il est possible de créer des utilisateurs, mais leurs boite mail ne seront pas fonctionnelles. Il faut un service capable de stocker les mails. 


Service de gestion des boites mails (Dovecot)
---------------------------------------------

Le service dovecot va assurer l'interface entre la base de mail au format MailDir et les clients de messagerie des utilisateurs finaux. Le protocol servi pour ce faire sera uniquement l'IMAPS.

En association avec managesieve, dovecot permettra également aux utilisateur de gérer des filtres de message.

L'authentification des utilisateurs se fait sur la base de donnée Mysql.


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



On définit les parametres relatifs à la configuration des fichiers stockant les boites mails. Leurs emplacements, leurs types. Pour ce faire on edite le fichier **10-mail.conf**

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

Verification ::

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

SASL va être utilisé pour authentifier les utilisateurs de notre organisation, afin que seulement ceux-ci puissent envoyer des emails par le biai de notre serveur de mail.

Les fonctionnalités SASL vont être activées uniquement sur le port submission(587) prévu par la rfc6409.

En outre, nous avons choisi d'authentifier nos utilisateurs via dovecot qui lui même s'apuit sur la base mysql comme base de donnée utilisateur. Cette réalisation est trivial et évite les multiples configuration de mysql en backend des serivces postfix & co.

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

La gestion de l'authentification des utilisateurs est déléguée à dovecot. On active une socket unix sur le serveur dovecot pour que postfix puisse l'intérroger.

.. warning :: Les paramètres de configuration suivant, sont liés au serveur dovecot. Néanmoins, dans un soucis de compréhension, ils sont définis à cette endroit de la documentation. 

vi /etc/dovecot/conf.d/10-master.conf ::

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



