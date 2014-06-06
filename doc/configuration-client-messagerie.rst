=======================
Manuel de l'utilisateur
======================= 
Comment configurer la messagerie du SGMAP/Etalab. 



Configuration d'un client de messagerie via le protocol IMAP
============================================================

Thunderbird / Icedove
---------------------
Pour les clients de messagerie de type thunderbird, la configuration des paramètres de comptes est automatique.

Les champs toutefois nécessaire de renseigner sont, la description du compte, l'adresse mail ainsi que le mot de passe.

Concernant les autres clients de messagerie comme les téléphones par exemple les champs à renseigner doivent être similaire à ce qui suit:

Service IMAP (Réception)
------------------------
::

  Utilisateur : foo@data.gouv.fr
  Password    : foobar
  Serveur IMAP     : imap.data.gouv.fr
  Type de sécurité : SSL/TLS si disponible
  Port             : 993

Server SMTP (Envoi)
-------------------
::

  Serveur : smtp.data.gouv.fr
  Type de sécurité : STARTTLS
  Port             :587
  Type d'authentification : LOGIN ou PLAIN ou AUTOMATIC
  Utilisateur / Password : Idem que pour le service IMAP

Configuration d'un client Android via le protocol ActiveSync
============================================================
Aller dans les parametres d'Android. Dans la section "Comptes", aller dans "ajouter un compte" puis "Entreprise" renseigner les champs. 

Ensuite cliquer sur "suivants, sélectionner "Exchange" puis renseigner les champs. 

.. note :: Pour le champs serveur, on renseignera "mobile.data.gouv.fr" et pour type de sécurité, SSL/TLS (accepter tous les certificats)

C'est fini.  

Configuration d'un client iPhone via le protocol ActiveSync
============================================================
Aller dans les "Réglages" du téléphone. Dans la section "Mail,Contact, Calendrier", aller dans "Ajout" puis "Exchange" et renseigez les champs suivants ::

  Adresse : felix.defrance@data.gouv.fr
  Serveur : mobile.data.gouv.fr
  Domaine : data.gouv.fr

Si ce n'est pas déjà cocher, activer la prise en charge du SSL dans les "Réglages Avancés". 

C'est fini.

Configuration d'un client CalDAV et CardDAV
===========================================

ThunderBird / Icedove
---------------------
::

	Choose Go > Address Book.
	Choose File > New > Remote Address Book.
	Enter a signifcant name for your calendar in the Name feld.
	Type the following URL in the URL feld:
	http://webmail.data.gouv.fr/SOGo/dav/felix.defrance/Contacts/personal/

::
	
	Choose Go > Calendar.
	Choose Calendar > New Calendar.
	Select On the Network and click on Continue.
	Select CalDAV.
	Type the following URL in the URL feld:
	http://webmail.data.gouv.fr/SOGo/dav/felix.defrance/Calendar/personal/

Android (caldav sync adapter)
-----------------------------
Ces protocols sont gérer via l'application caldavsyncadapter. L'outil est téléchargeable depuis googleplay ou F-droid. ::

  https://f-droid.org/repository/browse/?fdfilter=caldav&fdid=org.gege.caldavsyncadapter

ou ::

  https://play.google.com/store/apps/details?id=org.gege.caldavsyncadapter&hl=fr


Les paramètres de configuration sont les mêmes que pour le client Thunderbird. 

Apple iCal
----------
L'url du serveur est le suivant ::
  
  http://webmail.data.gouv.fr/SOGo/dav/felix.defrance/


