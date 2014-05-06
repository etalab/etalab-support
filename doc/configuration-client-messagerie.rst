Configuration d'un client de messagerie en IMAP
===============================================

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

Configuration d'un client Android avec ActiveSync
=================================================
Aller dans les parametres d'Android. Dans la section "Comptes", aller dans "ajouter un compte" puis "Entreprise" renseigner les champs. 

Ensuite cliquer sur "suivants, sélectionner "Exchange" puis renseigner les champs. 

.. note :: Pour le champs serveur, on renseignera "mobile.data.gouv.fr" et pour type de sécurité, SSL/TLS (accepter tous les certificats)

C'est fini.  

