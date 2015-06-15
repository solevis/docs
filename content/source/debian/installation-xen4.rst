Installation de Xen4 sur Debian Wheezy
######################################

.. contents::
    :local:
    :backlinks: top

Installation des paquets
========================

.. code-block:: bash
    
    aptitude install xen-linux-system-amd64 xen-tools xen-utils-4.1 xen-hypervisor-4.1-amd64

Configuration de grub
=====================

Pour que le serveur boot automatiquement sur Xen :

.. code-block:: bash
    
    cd /etc/grub.d/
    ln -s 20_linux_xen 09_linux_xen

Test
====

.. code-block:: bash
    
    xm list

Si tout est correctement installé on devrait retrouver quelque chose d'équivalent à ça :

.. code-block:: bash
    
    root@gemenon:~# xm list
    Name                                        ID   Mem VCPUs      State   Time(s)
    Domain-0                                     0  1865     1     r-----     18.8
