Installation de Piwik/Nginx sur NetBSD
######################################

.. contents::
    :local:
    :backlinks: top

Piwik
=====

Piwik est un logiciel téléchargeable d'analyse Web en temps réel open source (sous licence GPL). Il vous fournit des rapports détaillés sur les visiteurs de votre site Web: moteur de recherche et mots clés utilisés, leurs langues, vos pages les plus populaires… et bien plus encore.

Piwik se veut être une alternative open source à Google Analytics.

Source : http://fr.piwik.org/

Dépendances
===========

Pour utiliser Piwik, il faut au minimum installer (via pkgin ou pkgsrc) :


.. code-block:: bash
    
    graphics/php-gd
    databases/php-pdo_mysql
    databases/mysql55-server
    databases/mysql55-client
    converters/php-iconv
    textproc/php-dom
    www/php-fpm
    www/nginx


Installation de Piwik
=====================

On télécharge la dernière archive, et on la décompresse :

.. code-block:: bash
    
    wget http://builds.piwik.org/latest.tar.gz
    tar xvzf latest.tar.gz
    cp -rfv piwik/* /var/www/piwik.solevis.net


Configuration de PHP
====================

Ajouter les extensions installés dans php.ini

.. code-block:: bash
    
    ${EDITOR} /usr/pkg/etc/php.ini

.. code-block:: bash
    
    extension=gd.so
    extension=pdo.so
    extension=dom.so
    extension=pdo_mysql.so
    extension=iconv.so

Configuration de MySQL
======================

On démarre le serveur :

.. code-block:: bash
    
    echo "mysqld=YES" >> /etc/rc.conf
    /etc/rc.d/mysqld start


On utilise l'assistant d'installation pour sécuriser un minimum le serveur :

.. code-block:: bash
    
    mysql_secure_installation


Ensuite on créé un utilisateur spécifique pour le Piwik et une base de données associée :

.. code-block:: bash
    
    mysql -u root -p

.. code-block:: bash
    
    mysql> use mysql;
    mysql> CREATE USER 'piwik'@'localhost' IDENTIFIED BY 'YOUR_PASSWORD';
    mysql> create database piwik;
    mysql> GRANT ALL PRIVILEGES ON piwik.* TO 'piwik'@'localhost' WITH GRANT OPTION;

Configuration de Nginx
======================

Une configuration basique pour Piwik :

.. code-block:: bash
    
    server {
            listen   80; ## listen for ipv4

            server_name  piwik.solevis.net;

            access_log  /var/log/nginx/piwik.access.log;
            error_log /var/log/nginx/piwik.error.log;

            root   /var/www/piwik.solevis.net/;
            index  index.php index.html;

            location / {
                    try_files $uri /index.php;        
            }

            # enable php
            location ~* ^/(?:index|piwik)\.php$ {
                    fastcgi_pass 127.0.0.1:9000; 
                    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                    include fastcgi_params;
            }
    }

Pour une configuration beaucoup plus avancée, on peut se référer à ce projet : https://github.com/perusio/piwik-nginx

Démarrage
=========

.. code-block:: bash
    
    echo "nginx=YES" >> /etc/rc.conf
    echo "php_fpm=YES" >> /etc/rc.conf

.. code-block:: bash
    
    /etc/rc.d/nginx start
    /etc/rc.d/php_fpm start

