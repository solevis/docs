Installation de FreeBSD sur un Sheevaplug
#########################################

.. contents::
    :local:
    :backlinks: top

Introduction
============

Depuis 8.1 il est possible d'installer et d'utiliser convenablement FreeBSD sur un Sheevaplug.
Ce guide est avant-tout un mémo donc je ne prends pas la peine de détailler toutes les étapes, mais il pourra surement servir à d'autres. La procédure est la
même pour un e-sata Sheevaplug (j'ai rajouté deux trois tips pour ce modèle).

Toutes les manipulations suivantes s'effectuent sur un FreeBSD 8.1 (amd64).

Récupération des sources
========================

.. code-block:: bash
    
    cp /usr/share/examples/cvsup/cvs-supfile ~
    $EDITOR cvs-supfile

.. code-block:: bash
    
    *default host=cvsup8.freebsd.org
    *default base=/var/db
    *default prefix=/usr
    *default release=cvs tag=RELENG_8_1
    *default delete use-rel-suffix
    *default compress
    src-all


.. code-block:: bash
    
    csup ~/cvs-supfile


Patchs
======

bind_segfault.diff_ : Corrige une erreur de segmentation de Bind9

default_root_pass.diff_ : Rajoute simplement le mots de passe 'root' au compte root

force_fsck.diff_ : Ajoute une option dans 'rc.conf' pour forcer le fsck au boot

usb_boot.diff_ : Pour booter via le port USB

Pour appliquer les patchs::

.. code-block:: bash
    
    patch -p < the_patch

.. _bind_segfault.diff: http://blog.solevis.net/patchs/bind_segfault.diff
.. _default_root_pass.diff: http://blog.solevis.net/patchs/default_root_pass.diff
.. _force_fsck.diff: http://blog.solevis.net/patchs/force_fsck.diff
.. _usb_boot.diff: http://blog.solevis.net/patchs/usb_boot.diff

Construction du monde
=====================

.. code-block:: bash
    
    cd /usr/src
    make -j 8 buildworld TARGET_ARCH=arm

Configuration du kernel
=======================

Pour le e-sata Sheevaplug

.. code-block:: bash
    
    cd /usr/src/sys/arm/conf
    cp SHEEVAPLUG SHEEVAPLUG_ESATA

.. code-block:: bash
    
    $EDITOR SHEEVAPLUG_ESATA

.. code-block:: bash
    
    # e-sata support
    device ata
    device atadisk

Pour tous les modèles il faut modifier le paramètre *ROOTDEVNAME* dans la configuration du kernel. Il faut
donner l'emplacement de **/** sur le support de stockage. Par exemple **ad0s2a** dans le cas d'un disque connecté
au port e-sata.

.. code-block:: bash
    
    # Root fs 
    options         ROOTDEVNAME=\"ufs:/dev/ad0s2a\"

Partionnement
=============

Il faut au minimum deux partitions sur votre support, une première pour stocker le kernel et une seconde pour votre
système. (Remplacer da0 par le bon device)

Suppression des partitions existantes

.. code-block:: bash
    
    gpart delete -i 1 da0
    gpart destroy da0

Création d'une partition FAT32

.. code-block:: bash
    
    gpart create -s MBR da0
    gpart add -s 32M -t freebsd da0
    newfs_msdos /dev/da0s1

Partition système (Cas d'un seul slice)

.. code-block:: bash
    
    gpart add -t freebsd da0
    bsdlabel -w /dev/da0s2
    newfs -n da0s2a

Installation du monde
=====================

.. code-block:: bash
    
    mkdir /mnt/sheeva
    mount /dev/da0sa2 /mnt/sheeva
    export DESTDIR=/mnt/sheeva
    make installworld distrib-dirs distribution TARGET_ARCH=arm
    umount /mnt/sheeva

Configuration
=============

rc.conf::

    hostname="foobar"
    ifconfig_mge0="DHCP"
    sshd_enable="YES"

    # Force fsck                                                                    
    fsck_y_enable="YES"                                                             
    background_fsck="NO"                                                            
    force_fsck="YES"                                                                
    force_fsck_list="/"                                                             
                                                                                    
    # Ntp                                                                           
    ntpd_enable="YES"                                                               
    ntpd_flags="-g"

fstab::

    /dev/ad0s2a / ufs rw,noclusterr,noclusterw 0 0

Installation du kernel
======================

.. code-block:: bash
    
    mkdir /mnt/kernel
    mount -t msdosfs /dev/da0s1 /mnt/kernel
    cp /usr/obj/arm/usr/src/sys/SHEEVAPLUG/kernel.bin /mnt/kernel 
    (ou SHEEVAPLUG_ESATA)

Fin de l'installation
=====================

.. code-block:: bash
    
    sync
    umount /mnt/sheeva
    umount /mnt/kernel

On peut maintenant brancher le support au Sheevaplug.

Configuration du Sheevaplug
===========================

Il faut maintenant préciser à l'u-boot où se trouve le kernel et de le charger en RAM.
::

    Marvell>> setenv bootcmd "bootcmd=ide reset;fatload ide 1:1 900000 kernel.bin;go 900000"
    Marvell>> saveenv
    Marvell>> boot

Première connexion

.. code-block:: bash
    
    login : root
    pass : root
    # passwd
    # tzsetup
    # adduser -m -G wheel foobar

Problèmes et solutions
======================

Les bugs que j'ai recontré et comment les résoudre.

xz (Core Dump)
++++++++++++++

.. code-block:: bash
    
    mv /usr/bin/xz /usr/bin/xz-sav
    cd /usr/ports/xz
    $EDITOR MakeFile

.. code-block:: bash
    
    #.if ${OSVERSION} >= 900012 || (${OSVERSION} <>= 800505)
    #IGNORE= is already in the base system
    #.endif

.. code-block:: bash
    
    make install clean

Perl
++++

Il faut utiliser la version **5.8**.
