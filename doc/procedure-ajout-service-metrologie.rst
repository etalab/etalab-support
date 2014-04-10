Installation de l'agent Munin
-----------------------------
Sur la machine à sonder, on installe le packet munin ::

  apt-get install munin-node munin-plugins-extra

On autorise les connexions du serveur centrale en ajoutant dans le fichier */etc/munin/munin-node.conf* ::

  allow ^37\.187\.72\.214$

Puis on relance le service ::

  service munin-node restart

Ajout de check supplémentaire
-----------------------------
Pour ajouter des sondes supplémentaires, on pourra ajouter un lien symbolique comme suit ::

    ln -s /usr/share/munin/plugins/<sonde> /etc/munin/plugins

et recharger la configuration de la sonde munin ::

    service munin-node restart

Ajout du dépot contib
~~~~~~~~~~~~~~~~~~~~~
On installe le dépot contrib de munin ::

    cd /usr/local/src/
    git clone https://github.com/munin-monitoring/contrib.git /usr/local/src/munin-monitoring
    

Tester un plugin
~~~~~~~~~~~~~~~~
::

    cd /etc/munin/plugins
    munin-run <nom plugin>


Configuration du parefeu si il est présent
------------------------------------------
::

    # -------------------------- Munin
    iptables -A INPUT -p tcp -s $MUNIN -i $IFEXT0 --dport 4949 -j ACCEPT



Déclaration sur le serveur central
----------------------------------
::

    vi /etc/munin/munin.conf

    [nom_host]
         address nom_host.fqdn
         use_node_name yes

Au prochain lancement du cron sur le serveur, les nouvelles machines graphées.
