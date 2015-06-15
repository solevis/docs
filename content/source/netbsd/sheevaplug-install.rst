Installation de NetBSD current sur un Sheevaplug
################################################

.. contents::
    :local:
    :backlinks: top

Introduction
============

Le port de NetBSD sur le Sheevaplug est disponible depuis Octobre 2010. Même si je rencontre quelques soucis d'utilisation je me permet de décrire une méthode simple pour installer NetBSD sur un Sheevaplug.

Toutes les manipulations ont été effectués sur MacOS X, mais c'est le même topo sur BSD, GNU/Linux, etc. Il faut toutefois NetBSD pour le partitionnement du support.

Il est tout à fait possible d'utiliser tftp et nfs pour s'amuser avec NetBSD sans avoir à se prendre la tête du choix d'un support physique.

Récupération des sources
========================

Depuis CVS :

.. code-block:: bash
    
    export CVS_RSH="ssh"
    export CVSROOT="anoncvs@anoncvs.NetBSD.org:/cvsroot"
    cvs checkout -A -P src

Pour mettre à jour les sources :

.. code-block:: bash
    
    cvs update -A -Pd

Configuration du kernel
=======================

Par défaut la configuration du kernel est basique. Vous pouvez ajouter les options qui suivent si cela vous interesse.

.. code-block:: bash
    
    cp sys/arch/evbarm/conf/SHEEVAPLUG sys/arch/evbarm/conf/SHEEVAPLUG_CUSTOM
    $EDITOR sys/arch/evbarm/conf/SHEEVAPLUG_CUSTOM

Packet Filter :

.. code-block:: bash
    
    options     PFIL_HOOKS      # pfil(9) packet filter hooks
    pseudo-device       pf              # PF packet filter
    pseudo-device       pflog           # PF log interface

Ne pas devoir choisir la partition root au boot (indispensable pour tout le monde je pense) :

.. code-block:: bash
    
    config netbsd-sd0 root on sd0 type ffs

Choix du device :
    - sd0 correspond au disque en USB.
    - ld0 correspond à la carte SD
    - wd0 correspond au disque connecté au port e-sata (pas encore testé)

Compilation de NetBSD
=====================

.. code-block:: bash
    
    ./build.sh -m evbarm tools
    ./build.sh -u -U -m evbarm release
    ./build.sh -u -U -m evbarm kernel=SHEEVAPLUG

Partionnement
=============

Effacer le disque
+++++++++++++++++

Pour s’assurer qu’un disque est complètement vide, il suffit de remplir de “zéro” le premier secteur du disque via dd. Par exemple pour sd4:

.. code-block:: bash
    
    dd if=/dev/zero of=/dev/sd4d bs=8k count=1

Création des partitions
+++++++++++++++++++++++

Pour utiliser NetBSD sur le Sheevaplug nous avons besoin au minimum de deux partitions. Une première contenant le kernel au format FAT32 et une seconde pour le système en lui même.
La création des partitions passent par l’utilitaire fdisk.

.. code-block:: bash
    
    fdisk -u sd4
    ...
    Do you want to change our idea of what BIOS thinks? [n] n

.. code-block:: bash
    
    Partition table:
    0: <UNUSED>
    1: <UNUSED>
    2: <UNUSED>
    3: <UNUSED>
    Bootselector disabled.
    No active partition.
    Which partition do you want to change?: [none]

Partition FAT32 de 32MB (soit 65536 secteurs). Pour connaître le nombre de secteurs il suffit de faire un simple produit en croix. Par exemple pour 32MB : Je sais que j’ai 488397168 secteurs au total et 238475MB, donc il me faut (488397168 * 32) / 238475 = 65536 secteurs pour faire 32MB.

.. code-block:: bash
    
    Which partition do you want to change?: [none] 0
    The data for partition 0 is:
    <UNUSED>
    sysid: [0..255 default: 169] 11
    start: [0..30401dcyl default: 63, 0dcyl, 0MB]
    size: [0..30401dcyl default: 488397105, 30401dcyl, 238475MB] 65536
    bootmenu: []

Partition FFS sur le reste du disque (garder les paramètres par défaut) :

.. code-block:: bash
    
    Which partition do you want to change?: [none] 1
    The data for partition 1 is:
    <UNUSED>
    sysid: [0..255 default: 169]
    start: [0..30401dcyl default: 65599, 4dcyl, 32MB]
    size: [0..30397dcyl default: 488331569, 30397dcyl, 238443MB]
    bootmenu: []

Ce qui nous donne au final :

.. code-block:: bash
    
    Partition table:
    0: Primary DOS with 32 bit FAT (sysid 11)
        start 63, size 65536 (32 MB, Cyls 0-4/21/16)
            PBR is not bootable: All bytes are identical (0x00)
    1: NetBSD (sysid 169)
        start 65599, size 488331569 (238443 MB, Cyls 4/21/17-30401/80/63)
            PBR is not bootable: All bytes are identical (0x00)
    2: <UNUSED>
    3: <UNUSED>

Pour prendre en compte les modifications :

.. code-block:: bash
    
    Which partition do you want to change?: [none] none
    Should we write new partition table? [n] y

Création des slices
+++++++++++++++++++

aintenant que le disque est correctement formaté, il faut créer les différents slices de NetBSD. Il faut au minimum un slice pour /. Dans cet exemple j’organise mon disque de cette façon:

    - **kernel/FAT32** : 32MB (65536) [e]
    - **/** : 1GB FFS (2097153) [a]
    - **swap** : 2G (4194307) [b]
    - **/tmp** : 512MB FFS (1048576) [f]
    - **/var** : 2GB FFS (4194307) [g]
    - **/usr** : * FFS (480991596) [h]

Pour calculer la taille de chaque slice en secteurs il faut utiliser la même méthode que pour fdisk : (secteurs_max_disque * taille) / taille_max_disque.
Pour créer les slices on utilise **disklabel**. Pour chaque label (a, b, e, etc) il faut spécifier où celui-ci commence (offset) et sa taille (size). Pour connaitre le premier offset il suffit de lancer **fdisk sd4** et de lire l’offset de notre première partition. Dans mon cas “start 63” donc **63**. Pour les autres offset il suffit d'additionner la taille et l’offset du slice précédent.

.. code-block:: bash
    
    disklabel -e sd4
    ...
    8 partitions:
    #        size    offset     fstype [fsize bsize cpg/sgs]
     a:   2097153     65599     4.2BSD      0     0     0  # (Cyl.     65*-   2145*)
     b:   4194307   2162752     4.2BSD      0     0     0  # (Cyl.   2145*-   6306*)
     c: 488331632        63     unused      0     0        # (Cyl.      0*- 484456*)
     d: 488397168         0     unused      0     0        # (Cyl.      0 - 484520)
     e:     65536        63      MSDOS                     # (Cyl.      0*-     65*)
     f:   1048576   6357059     4.2BSD      0     0     0  # (Cyl.   6306*-   7346*)
     g:   4194307   7405635     4.2BSD      0     0     0  # (Cyl.   7346*-  11507*)
     h: 476797226  11599942     4.2BSD      0     0     0  # (Cyl.  11507*- 484520)

Explications:

    - **c** correspond à l’emplacement de NetBSD sur le disque.
    - **d** correspond à tout le disque
    - **e** est la partition contenant le kernel

Création des filesystem
+++++++++++++++++++++++

Pour la partition FAT32 :

.. code-block:: bash
    
    newfs_msdos /dev/rsd4e

Pour les partitions FFS (NetBSD) :

.. code-block:: bash
    
    newfs /dev/rsd4a
    newfs /dev/rsd4f
    newfs /dev/rsd4g
    newfs /dev/rsd4h

Installation
============

Il faut tout d’abord monter chaque partitions :

.. code-block:: bash
    
    mkdir /mnt/kernel
    mount -t msdosfs /dev/sd4e /mnt/kernel
    mkdir /mnt/sheeva
    mount /dev/sd4a /mnt/sheeva
    mkdir /mnt/sheeva/tmp
    mount /dev/sd4f /mnt/sheeva/tmp
    mkdir /mnt/sheeva/var
    mount /dev/sd4g /mnt/sheeva/var
    mkdir /mnt/sheeva/usr
    mount /dev/sd4h /mnt/sheeva/usr

Ensuite il faut décompresser chaque sets sur le disque :

.. code-block:: bash
    
    cd obj/releasedir/evbarm/binary/sets
    tar xvzf base.tgz -C /mnt/sheeva
    tar xvzf comp.tgz -C /mnt/sheeva
    tar xvzf etc.tgz -C /mnt/sheeva
    tar xvzf games.tgz -C /mnt/sheeva
    tar xvzf man.tgz -C /mnt/sheeva
    tar xvzf misc.tgz -C /mnt/sheeva
    tar xvzf modules.tgz -C /mnt/sheeva
    tar xvzf tests.tgz -C /mnt/sheeva
    tar xvzf text.tgz -C /mnt/sheeva

Puis copier le kernel dans la partition FAT32 :

.. code-block:: bash
    
    cd -
    cd sys/arch/evbarm/compile/obj/SHEEVAPLUG
    cp netbsd.ub /mnt/kernel
    (ou si vous avez configuré la partition root dans le kernel)
    cp netbsd-sd0.ub /mnt/kernel/netbsd.ub
    (remplacer sd0 par le bon device ld0, wd0, etc)

Configuration
=============

Avant de pouvoir brancher le disque et lancer NetBSD il est sage de configurer quelques fichiers.

etc/fstab
+++++++++

.. code-block:: bash
    
    /dev/sd0a       /       ffs     rw      1 1
    /dev/sd0b       none    swap    sw,dp   0 0
    /dev/sd0f       /tmp    ffs     rw      1 1
    /dev/sd0g       /var    ffs     rw      1 1
    /dev/sd0h       /usr    ffs     rw      1 1

etc/rc.conf
+++++++++++

.. code-block:: bash
    
    if [ -r /etc/defaults/rc.conf ]; then
            . /etc/defaults/rc.conf
    fi

    # If this is not set to YES, the system will drop into single-user mode.
    #
    rc_configured=YES

    # Add local overrides below
    #
    hostname=mon_host
    # Daemons
    sshd=YES
    # Horloge
    ntpdate=YES
    ntpdate_hosts="pool.ntp.org"
    # Reseau
    dhclient=YES
    dhclient_flags=mvgbe0
    critical_filesystems_local="/var"

etc/mk.conf
+++++++++++

.. code-block:: bash
    
    NO_X11=YES

Ce ne sont que des exemples, libre à vous de les modifier comme bon vous semble.

Fin de l'installation
=====================

.. code-block:: bash
    
    cd
    sync
    umount /mnt/sheeva/tmp
    umount /mnt/sheeva/var
    umount /mnt/sheeva/usr
    umount /mnt/sheeva/
    umount /mnt/kernel

Configuration du u-boot
=======================
Se connecter via la console série au Sheevaplug.
Pour tester l’installation. Depuis un disque en USB :

.. code-block:: bash
    
    Marvell>> usb start
    Marvell>> fatload usb 0:1 0x2000000 netbsd.ub
    Marvell>> bootm 0x2000000

Pour rendre la configuration permanente :

.. code-block:: bash
    
    Marvell>> setenv bootcmd 'usb start; fatload usb 0:1 0x2000000 netbsd.ub; bootm 2000000'
    Marvell>> saveenv
    Marvell>> reset

Depuis une carte SD :

.. code-block:: bash
    
    Marvell>> mmc init
    Marvell>> fatload mmc 0:1 0x2000000 netbsd.ub
    Marvell>> bootm 0x2000000

.. code-block:: bash
    
    Marvell>> setenv bootcmd 'mmc init; fatload mmc 0:1 0x2000000 netbsd.ub; bootm 2000000'
    Marvell>> saveenv
    Marvell>> reset

Depuis TFTP :

.. code-block:: bash
    
    Marvell>> tftpboot 2000000 netbsd.ub
    Marvell>> bootm 2000000

Depuis un disque en E-SATA:

.. code-block:: bash
    
    Marvell>> ide reset
    Marvell>> fatload ide 0:1 0x2000000 netbsd.ub
    Marvell>> bootm 0x2000000

.. code-block:: bash
    
    Marvell>> setenv bootcmd 'ide reset; fatload ide 0:1 0x2000000 netbsd.ub; bootm 2000000'
    Marvell>> saveenv
    Marvell>> reset

First boot
==========

.. code-block:: bash
    
    login: root
    passwd
    user -m -G wheel foo
    passwd foo

Problèmes
=========

Les bugs que j'ai recontré durant mon utilisation de NetBSD sur mon Sheevaplug :

Console série
+++++++++++++

Si on utilise pendant un trop long moment NetBSD depuis la console série (minicom, screen, etc) celui-ci crash tout bêtement.
Par contre aucun soucis via SSH.

sdmmc0: couldn't enable card
++++++++++++++++++++++++++++

Lors du boot de NetBSD si aucune carte SD n'est inséré on a un beau flood de :

.. code-block:: bash
    
    sdmmc0: couldn't enable card

Le seul moyen d'y remédier pour le moment c'est d'insérer une carte au boot, ou à l'apparition des messages. Ensuite elle peut être enlevé sans aucun soucis.
