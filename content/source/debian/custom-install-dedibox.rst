Installation de Debian via debootstrap/chroot sur Dedibox
#########################################################

.. contents::
    :local:
    :backlinks: top

Mode Rescue
===========

Il faut simplement se connecter sur la console de online.net, choisir son dédié et puis cliquer
sur le bouton "Secours".

La Dedibox va "rebooter" sur une ubuntu minimal. Pour s'y connecter via SSH, il suffit d'utiliser
les informations données par online.net.

Partitionnement
===============

.. code-block:: bash
    
    sudo cfdisk
    sudo mkfs.ext4 /dev/sdaX

Installation de debootstrap
===========================

.. code-block:: bash
    
    wget http://archive.ubuntu.com/ubuntu/pool/main/d/debootstrap/debootstrap_1.0.42ubuntu1_all.deb
    sudo dpkg --install debootstrap_1.0.42ubuntu1_all.deb

debootstraping
==============

On montre notre partition :

.. code-block:: bash
    
    sudo mkdir /mnt/debian
    sudo mount /dev/sdaX /mnt/debian

On installe le système de base dans la partition :

.. code-block:: bash
    
    sudo debootstrap squeeze /mnt/debian http://ftp2.fr.debian.org/debian/

Montage des partitions
======================

.. code-block:: bash
    
    sudo mount -o bind /proc /mnt/debian/proc
    sudo mount -o bind /sys /mnt/debian/sys/
    sudo mount -o bind /dev /mnt/debian/dev/
    sudo mount -o bind /dev/pts /mnt/debian/dev/pts/

Configuration du réseau
=======================

.. code-block:: bash
    
    sudo cp /etc/hosts /mnt/debian/etc/hosts
    sudo cp /etc/resolv.conf /mnt/debian/etc/resolv.conf

Chroot
======

.. code-block:: bash
    
    chroot /mnt/debian /bin/bash

Configuration
=============

Paritions
+++++++++

.. code-block:: bash
    
    ${EDITOR} /etc/fstab

.. code-block:: bash
    
    /dev/sdaX	/	ext4	defaults	0	1
    proc	/proc	proc	defaults	0	0

.. code-block:: bash
    
    mount -a

Fuseau Horaire
++++++++++++++

.. code-block:: bash
    
    dpkg-reconfigure tzdata

Réseau
++++++

.. code-block:: bash
    
    ${EDITOR} /etc/network/interfaces

.. code-block:: bash
    
    # loopback
    auto lo
    iface lo inet loopback

    # DHCP
    auto eth0
    iface eth0 inet dhcp

Hostname
++++++++

.. code-block:: bash
    
    echo "mon_hostname" > /etc/hostname


Installation des paquets
++++++++++++++++++++++++

On peut choisir ce que l'on installe :

.. code-block:: bash
    
    aptitude install vim less ssh sudo

Ou utiliser directement le groupe de paquets standard de Debian :

.. code-block:: bash
    
    tasksel install standard

Installation du noyau
+++++++++++++++++++++

.. code-block:: bash
    
    aptitude install linux-image-amd64

Programme d'ammorcage
+++++++++++++++++++++

.. code-block:: bash
    
    aptitude install grub

Mot de passe root
+++++++++++++++++

.. code-block:: bash
    
    passwd root

Création d'un compte
++++++++++++++++++++

.. code-block:: bash
    
    useradd -m new_user
    passwd new_user

Remerciements
=============

Comme il le dit si bien lui même, je remercie mon "valeureux compagnon de route" : Llew_

.. _Llew: http://www.llew.me/
