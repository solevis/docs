Dépôts GIT sur NetBSD
#####################

.. contents::
    :local:
    :backlinks: top

Installation de GIT
===================

.. code-block:: bash
    
    root# pkgin in scmgit

ou

.. code-block:: bash
    
    root# cd /usr/pkgsrc/devel/scmgit
    root# make install clean

Configuration du système
========================

Création d'un utilisateur git :

.. code-block:: bash
    
    root# groupadd git
    root# useradd -m -g git git

Creation d'un dépôt GIT
=======================

Création d'un dépôt:

.. code-block:: bash
    
    user% su - git
    git% mkdir /home/git/repos/
    git% git init --bare /home/git/repos/foo.git

Pour rendre le dépôt public :

.. code-block:: bash
    
    git% touch /home/git/repos/foo.git/git-daemon-export-ok

Daemon GIT
==========

Lancement du serveur :

.. code-block:: bash
    
    git% /usr/pkg/libexec/git-core/git-daemon --base-path=/home/git/repos/ --listen=votre_ip

Si tout marche pour le mieux, on créé un script rcng dans /etc/rc.conf:

.. code-block:: bash
    
    root# $EDITOR /etc/rc.d/gitdaemon

.. code-block:: bash
    
    #!/bin/sh
    #
    # PROVIDE: gitdaemon
    # REQUIRE: DAEMON  

    . /etc/rc.subr

    name="gitdaemon"
    rcvar=$name
    pidfile="/var/run/$name.pid"
    command="/usr/pkg/libexec/git-core/git-daemon"
    command_args="--detach --base-path=/home/git/repos --user=git --group=git --pid-file=$pidfile"

    load_rc_config $name
    run_rc_command $1

On applique les bon droits :

.. code-block:: bash
    
    root# chmod 755 /etc/rc.d/gitdaemon

On l'ajoute dans rc.conf :

.. code-block:: bash
    
    root# echo "gitdaemon=YES" >> /etc/rc.conf

On le lance :

.. code-block:: bash
    
    root# /etc/rc.d/gitdaemon start

Recuperation du dépôt sur un client lambda
==========================================

Tout simplement :

.. code-block:: bash
    
    user% git clone git://mon.serveur/foo.git

Recuperation du dépôt puis push via SSH
=======================================

On ajoute notre clef publique sur le serveur :

.. code-block:: bash
    
    user% su - git
    git% mkdir /home/git/.ssh/
    git% $EDITOR /home/git/.ssh/authorized_keys

On récupère le dépôt via ssh :

.. code-block:: bash
    
    user% git clone ssh://git@mon.serveur:port/home/git/repos/foo.git

On test :

.. code-block:: bash
    
    user% cd foo
    user% touch bar
    user% git add bar
    user% git commit

On positionne la branche :

.. code-block:: bash
    
    user% git push origin master

On push :

.. code-block:: bash
    
    user% git push
