Installation de Owncloud/Nginx sur NetBSD
#########################################

.. contents::
    :local:
    :backlinks: top

Dépendances
===========

Pour utiliser Owncloud, il faut au minimum installer (via pkgin ou pkgsrc) :


.. code-block:: bash
    
    textproc/php-dom
    converters/php-mbstring
    archivers/php-zip
    archivers/php-zlib
    databases/php-pdo_sqlite
    graphics/php-gd
    textproc/php-json
    www/php-fpm

Pour Nginx :

.. code-block:: bash
    
    echo "PKG_OPTIONS.nginx+=     dav" >> /etc/mk.conf

puis :

.. code-block:: bash
    
    www/nginx


Installation de Owncloud
========================

On télécharge la dernière archive, et on la décompresse :

.. code-block:: bash
    
    wget http://mirrors.owncloud.org/releases/owncloud-4.5.0.tar.bz2
    tar xvzf owncloud-4.5.0.tar.bz2
    cp -rfv owncloud/* /var/www/cloud.solevis.net


Configuration de PHP
====================

Ajouter les extensions installés dans php.ini

.. code-block:: bash
    
    ${EDITOR} /usr/pkg/etc/php.ini

.. code-block:: bash
    
    extension=json.so
    extension=zip.so
    extension=zlib.so
    extension=mbstring.so
    extension=gd.so
    extension=pdo.so

Configuration de Nginx
======================

Exemple de configuration d'un vhost tiré du blog de nicolargo_ :

.. code-block:: bash
    
    server {

        listen 80;
        server_name cloud.solevis.net;

        root /var/www/cloud.solevis.net;
        client_max_body_size 1000M;
        index index.php;

        # Webdav configuration
        dav_methods PUT DELETE MKCOL COPY MOVE;
        create_full_put_path on;
        dav_access user:rw group:rw all:r;

        try_files $uri $uri/ @webdav;

        location @webdav {
                fastcgi_split_path_info ^(.+\.php)(/.+)$;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                include fastcgi_params;
                fastcgi_pass 127.0.0.1:9000;
        }

        # PHP-FPM server listening on 127.0.0.1:9000
        location ~ \.php$ {
                try_files $uri =404;
                fastcgi_pass 127.0.0.1:9000;
                fastcgi_index index.php;
                include fastcgi_params;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        }

        # Stuffs
        location = /favicon.ico {
                access_log off;
                return 204;
        }

        # Protect hidden file to read/write access
        location ~ /\. {
                deny all;
        }

    }


.. _nicolargo: http://blog.nicolargo.com/

Démarrage
=========

.. code-block:: bash
    
    echo "nginx=YES" >> /etc/rc.conf
    echo "php_fpm=YES" >> /etc/rc.conf

.. code-block:: bash
    
    /etc/rc.d/nginx start
    /etc/rc.d/php_fpm start

