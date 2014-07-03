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

Si vous souhaitez utiliser un client lourd, vous pouvez suivre les instructions ci-après.

Configuration d'un client de messagerie via le protocol IMAP
============================================================

Thunderbird / Icedove
---------------------
Pour les clients de messagerie de type thunderbird, la configuration des paramètres de comptes est automatique.

Les champs toutefois nécessaire de renseigner sont les suivants: 

    * la description du compte, 
    * l'adresse mail ainsi que 
    * le mot de passe.

Concernant les autres clients de messagerie comme les téléphones par exemple, les champs à renseigner doivent être similaire à ce qui suit:

Service IMAP (Réception)
~~~~~~~~~~~~~~~~~~~~~~~~
::

  Utilisateur : foo@data.gouv.fr
  Password    : foobar
  Serveur IMAP     : imap.data.gouv.fr
  Type de sécurité : SSL/TLS si disponible
  Port             : 993

Server SMTP (Envoi)
~~~~~~~~~~~~~~~~~~~
::

  Serveur : smtp.data.gouv.fr
  Type de sécurité : STARTTLS
  Port             :587
  Type d'authentification : LOGIN ou PLAIN ou AUTOMATIC
  Utilisateur / Password : Idem que pour le service IMAP


Configuration d'un client CalDAV et CardDAV
===========================================

ThunderBird / Icedove
---------------------

Pour permettre les fonctionnalités de calendrier et contact avec thunderbird, il faut installer le plugin SOGO conntector disponible ici :

  * http://www.sogo.nu/downloads/frontends.html

Une fois le plugin installé et thunderbird relancé, on configure le carnet d'adresse

.. note: Veuilliez prêter une attention particulière l'url d'accès ou vous devez changer felix.defrance par votre prenom.nom. 

::

	Choose Go > Address Book.
	Choose File > New > Remote Address Book.
	Enter a signifcant name for your calendar in the Name feld.
	Type the following URL in the URL feld:
	https://webmail.data.gouv.fr/SOGo/dav/felix.defrance/Contacts/personal/

De la même manière, on procède pour le calendrier ::
	
	Choose Go > Calendar.
	Choose Calendar > New Calendar.
	Select On the Network and click on Continue.
	Select CalDAV.
	Type the following URL in the URL feld:
	https://webmail.data.gouv.fr/SOGo/dav/felix.defrance/Calendar/personal/


Synchronisation des contacts et calendrier sous Android
-------------------------------------------------------

L'application DAVdroid, permet de synchroniser les contacts ainsi que les calendriers.

Configuration de DAVdroid
~~~~~~~~~~~~~~~~~~~~~~~~~
L'outil est téléchargeable depuis googleplay ou F-droid.

   * https://f-droid.org/repository/browse/?fdfilter=davdroid&fdid=at.bitfire.davdroid
   * https://play.google.com/store/apps/details?id=at.bitfire.davdroid&hl=fr

On lance Davdroid puis on clique sur l'icone de la clé avec un "+", on selectionne DAVdroid en on renseigne comme il suit ::

    https:// webmail.data.gouv.fr/SOGo/dav
    Utilisateur : felix.defrance
    Mot de passe  : **********

Et on décoche l'authentification par "Digest".


Apple iCal
----------
L'url du serveur est le suivant ::
  
  https://webmail.data.gouv.fr/SOGo/dav/felix.defrance/


======================================
Utilisation via le protocol ActiveSync
======================================

Configuration d'un client Android
=================================
Aller dans les parametres d'Android. Dans la section "Comptes", aller dans "ajouter un compte" puis "Entreprise" renseigner les champs. 

Ensuite cliquer sur "suivants, sélectionner "Exchange" puis renseigner les champs. 

.. note :: Pour le champs serveur, on renseignera "mobile.data.gouv.fr" et pour type de sécurité, SSL/TLS (accepter tous les certificats)

C'est fini.  


Configuration d'un client iPhone
================================
Aller dans les "Réglages" du téléphone. Dans la section "Mail,Contact, Calendrier", aller dans "Ajout" puis "Exchange" et renseigez les champs suivants ::

  Adresse : felix.defrance@data.gouv.fr
  Serveur : mobile.data.gouv.fr
  Domaine : data.gouv.fr

Si ce n'est pas déjà coché, activer la prise en charge du SSL dans les "Réglages Avancés". 

C'est fini.

Configuration d'un client Blackberry
====================================

Todo.

