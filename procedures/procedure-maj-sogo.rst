Procedure de mise à jour de sogo
================================

La machine qui fourni le service SOGo est la machine mail. L'upgrade du package sogo est bloqué manuellement pour éviter les erreurs de manipulations.
 
Afin de mettre à jour SOGo, voici la méthode que j'utilise. 

Pré requis
----------

Avant toute modification, on vérifie les backups de la machine, 

On définit également un downtime sur la machine ainsi que sur les services associés,

Et on se renseigne sur la montée de version. 

Pour obtenir des informations sur le package installer ainsi que celles à venir, on peut executer par exemple ::
 
    apt-get cache policy sogo 

Ensuite on consulte de changelog de l'application

    http://www.sogo.nu/bugs/changelog_page.php


Avant la mise à jour effective, on peut aussi snapshoter la machine sur l'hyperviseur. 

Mise à jour
-----------
Pour effectuer la mise à jour on libère le package sogo ::

    apt-mark unhold sogo 

On installe le package  ::

    apt-get install sogo

Puis on (re)bloque le package ::

    apt-mark hold sogo

On vérifie dans les logs par exemple, que le service fonctionne correctement ::

    tailf /var/log/sogo/sogo.log 
