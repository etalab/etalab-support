Procedure de mise à jour d'Owncloud
===================================

Les fichiers applicatifs d'Owncloud sont localisés dans */var/www/owncloud*. Les données des utilisateurs sont localisé dans */var/www/owncloud/dataoc*. La base de donnée se trouve sur la machine smtp.intra.data.gouv.fr


Pré requis
----------
  * On vérifie que les backups de la machine, et précisement du répertoire /var/www/owncloud se sont correctement déroulés.
  * On vérifie que les dump et backup de la base de donnée de owncloud se cont correctement déroulé également.
  * Eventuellement (mode ceinture et bretelles) on peut créer un snapshot de la VM sur ceph. 


.. note:: Les dumps de mysql sur la machine smtp.intra.data.gouv.fr se trouvent ici *root@smtp:~# ll /var/backups/mysql*

Mise à jour
-----------
On définit un downtime de la machine ainsi que ses services sur icinga.

    http://supervision.data.gouv.fr/icinga

Pour mettre à jour owncloud, on se positionne dans son répertoire ::

    cd /var/www/owncloud

Et on télécharge les sources et on les décompresses ::

    wget https://download.owncloud.org/community/owncloud-7.0.2.tar.bz2 
    tar xvf owncloud-7.0.2.tar.bz2

On arrêt de servire Owncloud ::

    service nginx stop

On copie les fichiers existant vers le répertoire de la futur version d'owncloud ::

    cp -a owncloud-7.0.1 owncloud-7.0.2

Puis on patch le répertoire avec la nouvelle release ::

    rsync -av owncloud/* owncloud-7.0.2

On met à jour le path utilisé par nginx ::

    rm public_html
    ln -s owncloud-7.0.2 public_html

Modification du code d'owncloud
-------------------------------
Owncloud ne prend pas en charge la configuration propre de Nginx. 
On commente donc la structure de controle sur la présence du .htaccess ::

  vi owncloud-7.0.2/settings/admin.php
      //$htaccessworking=OC_Util::isHtaccessWorking();

Dernières vérifications
-----------------------
On vérifie également les droits sur le répertoire doivent être postitionnés à 2750. Si ce n'est pas le cas ::

    chmod -R 2750 owncloud-7.0.2

Relance d'Nginx pour fournir Owncloud
-------------------------------------
::

    service nginx start

Une fois le service démarré, on se loggue rapidement sur l'interface web pour executer les mises à jours de la DB...


Verification des logs 
---------------------
Pour s'assurer que le service fonctionne correctement on peut vérifier les logs en parallèle. Les logs se trouve dans */var/www/owncloud/dataoc/owncloud.log*

Monsieur propre
---------------
Pour finir proprement le travail on fait un peu de nettoyage ::

  rm -rf owncloud-7.0.1 owncloud-7.0.1.tar.bz2 owncloud


