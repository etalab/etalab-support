=======================
Manuel de l'utilisateur
======================= 

Comment accéder à la messagerie du SGMAP/Etalab. 

===================================================
Utilisation via des protocoles standards et ouverts
===================================================

Accès au webmail
================
Le webmail est un site internet accessible par tous les utilisateurs authentifiés, pour consulter ses mails à distance. Le protocole utilisé est en https, donc sécurisé. 

Muni de vos identifiants, vous pouvez accéder au webmail via l'adresse :

  * https://webmail.data.gouv.fr

Si vous souhaitez utiliser un client lourd type Thunderbird, vous pouvez suivre les instructions ci-après.

Configuration d'un client de messagerie via le protocol IMAP
============================================================

Thunderbird / Icedove
---------------------
Pour les clients de messagerie de type thunderbird, la configuration des paramètres de comptes est automatique.

Les champs toutefois nécessaire de renseigner sont les suivants: 

    * la description du compte, 
    * l'adresse mail ainsi que 
    * le mot de passe.

Smartphone sous Android
------------------------
Concernant les autres clients de messagerie comme les téléphones par exemple, les champs à renseigner doivent être similaire à ce qui suit:

Accédez aux paramètres des comptes dans "Paramètre" >  Section Compte, "Ajouter un compte" > "IMAP"

Service IMAP (Réception)
~~~~~~~~~~~~~~~~~~~~~~~~
Renseignez les champs suivants ::

  Utilisateur       : felix.defrance@data.gouv.fr (Ajustez à votre mail)
  Password          : *****************
  Serveur IMAP      : imap.data.gouv.fr
  Type de sécurité  : SSL/TLS si disponible
  Port              : 993

Server SMTP (Envoi)
~~~~~~~~~~~~~~~~~~~
Renseignez les champs suivants ::

  Serveur                   : smtp.data.gouv.fr
  Type de sécurité          : STARTTLS
  Port                      : 587
  Type d'authentification   : LOGIN ou PLAIN ou AUTOMATIC
  Utilisateur / Password    : Idem que pour le service IMAP


Smartphone sous Apple (Iphone)
------------------------------
Pour les téléphones d'Apple procédez comme suit ::

    Appuyez sur "Réglages" > "Mails, Contacts, Calendrier" > "Ajouter un compte" > "Autre"
    Appuyez sur "Ajouter un compte Mail"

Puis renseignez les champs suivants ::

    Nom         : Felix Defrance (Ajustez à votre nom)
    Adresse     : felix.defrance@data.gouv.fr (Ajustez à votre mail)
    Mot de passe: **************
    Description : Félix Defrance Etalab
    
    Appuyez sur "Suivant"

Ensuite, appuyez sur "IMAP" ::

	Entrez "webmail.data.gouv.fr" aux différents endroits appelés "Nom d'hôte"
	Entrez votre adresse email complète aux différents endroits appelés "Nom d'utilisateur"
	Entrez le "Mot de passe" de votre adresse email aux différents endroits demandés
	Appuyez sur "Suivant" puis enregistrez ces paramètres

Ensuite on s'assure que le protocol SSL est activé ::

    Appuyez sur "Réglages" > "Mails, Contacts, Calendrier"
    Aller dans le compte que vous venez de configurer, puis dans la section "Reglage de la reception" assurez vous que SSL est activé. 

C'est terminé. 

Configuration d'un client CalDAV et CardDAV
===========================================

ThunderBird / Icedove
---------------------

Pour permettre les fonctionnalités de calendrier et contact avec thunderbird, il faut installer le plugin SOGO conntector disponible ici :

  * http://www.sogo.nu/downloads/frontends.html

Une fois le plugin installé et thunderbird relancé, on configure le carnet d'adresse ::

    Allez dans "Outils" > "Carnet d'adresse"
    Allez dans "Fichier" > "Nouveau" > "Carnet d'adresse distant" 

    Renseignez votre nom ainsi que l'Url :
	URL : https://webmail.data.gouv.fr/SOGo/dav/felix.defrance/Contacts/personal/ (Ajustez à votre nom et prénom)

    Cochez la synchronisation périodique et définissez la fréquence à laquelle vos contacts doivent être synchronisés.
    

De la même manière, on procède pour le calendrier ::

	Allez dans "Fichier" > "Nouveau" > "Agenda"
	Cochez "Sur le réseau"
    Cochez Format : "CalDAV"
    Emplacement : https://webmail.data.gouv.fr/SOGo/dav/felix.defrance/Calendar/personal/ (Ajustez à votre nom et prénom)
    Cliquez sur suivant, puis renseigner votre Nom, une couleur. 
    Sélectionnez le compte de messagerie auquel le calendrier est attaché.

Smartphone sous Android
-----------------------

L'application DAVdroid, permet de synchroniser les contacts ainsi que les calendriers.

Configuration de DAVdroid
~~~~~~~~~~~~~~~~~~~~~~~~~
L'outil est téléchargeable depuis googleplay ou F-droid.

   * https://f-droid.org/repository/browse/?fdfilter=davdroid&fdid=at.bitfire.davdroid
   * https://play.google.com/store/apps/details?id=at.bitfire.davdroid&hl=fr

On lance Davdroid puis on clique sur l'icone de la clé avec un "+", on selectionne DAVdroid en on renseigne comme il suit ::

    https:// webmail.data.gouv.fr/SOGo/dav
    Utilisateur : felix.defrance@data.gouv.fr (Ajustez à votre mail)
    Mot de passe  : **********

Et on décoche l'authentification par "Digest".

Smartphone sous Apple (Iphone)
------------------------------
Apple Calendrier
~~~~~~~~~~~~~~~~
Pour ajouter un calendrier, procédez comme suit :
    
    Appuyez sur "Réglages" > "Mails, Contacts, Calendrier" > "Ajouter un compte" > "Autre"

    Dans la section Calendrier, appuyez sur " Ajouter un compte CalDAV" ::

        Server  : https://webmail.data.gouv.fr/SOGo/dav/felix.defrance/ (Ajustez à votre nom et prenom)
        Nom     : felix.defrance@data.gouv.fr (Ajustez à votre mail)
        Password: **********************
        Description: Félix Defrance Etalab

Apple Contacts
~~~~~~~~~~~~~~
Pour ajouter un carnet d'adresse, procédez comme suit :

    Appuyez sur "Réglages" > "Mails, Contacts, Calendrier" > "Ajouter un compte" > "Autre"

    Dans la section Contacts, appuyez sur " Ajouter un compte CardDAV" ::

        Server  : https://webmail.data.gouv.fr/SOGo/dav/felix.defrance/ (Ajustez à votre nom et prenom)
        Nom     : felix.defrance@data.gouv.fr (Ajustez à votre mail)
        Password: **********************
        Description: Félix Defrance Etalab


C'est terminé. 

======================================
Utilisation via le protocol ActiveSync
======================================

Configuration d'un smartphone Android
=====================================
Aller dans les parametres d'Android. Dans la section "Comptes", aller dans "ajouter un compte" puis "Entreprise" renseigner les champs. 

Ensuite cliquer sur "suivants, sélectionner "Exchange" puis renseigner les champs. 

.. note :: Pour le champs serveur, on renseignera "mobile.data.gouv.fr" et pour type de sécurité, SSL/TLS (accepter tous les certificats)

C'est terminé.  


Configuration smartphone Apple iPhone
=====================================

.. note: Attention le protocol activesync n'est pas complétement fonctionnel actuellement. 

Aller dans les "Réglages" du téléphone. Dans la section "Mail,Contact, Calendrier", aller dans "Ajout" puis "Exchange" et renseigez les champs suivants ::

  Adresse : felix.defrance@data.gouv.fr
  Serveur : mobile.data.gouv.fr
  Domaine : data.gouv.fr

Si ce n'est pas déjà coché, activer la prise en charge du SSL dans les "Réglages Avancés". 

C'est terminé.
