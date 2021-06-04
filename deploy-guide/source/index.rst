=================================
OpenStack Charms Deployment Guide
=================================

The OpenStack Charms Deployment Guide is the main source of information for
OpenStack Charms usage. To help improve it you can `file an issue`_ or
`submit a contribution`_.

.. note::

   For project information, development guidelines, release notes, and release
   schedules, please refer to the `OpenStack Charm Guide`_.

.. toctree::
   :caption: Installation
   :maxdepth: 1

   Overview <install-overview>
   install-maas
   install-juju
   install-openstack
   configure-openstack

.. toctree::
   :caption: Upgrades
   :maxdepth: 1

   Overview <upgrade-overview>
   upgrade-charms
   upgrade-openstack
   upgrade-series
   upgrade-special
   upgrade-issues

.. toctree::
   :caption: Security
   :maxdepth: 1

   Overview <security-overview>
   keystone
   app-policy-overrides
   app-certificate-management

.. toctree::
   :caption: Compute
   :maxdepth: 1

   nova-cells
   app-pci-passthrough-gpu

.. toctree::
   :caption: Networking
   :maxdepth: 1

   app-ovn
   app-hardware-offload
   app-octavia
   app-bridge-interface-configuration

.. toctree::
   :caption: High availability
   :maxdepth: 1

   app-ha
   app-masakari

.. toctree::
   :caption: Backup
   :maxdepth: 1

   app-trilio-vault

.. toctree::
   :caption: Operations
   :maxdepth: 1

   app-managing-power-events
   ceph-operations
   deferred-events
   operational-tasks

.. toctree::
   :caption: Storage
   :maxdepth: 1

   app-encryption-at-rest
   app-erasure-coding
   rgw-multisite
   app-ceph-rbd-mirror
   cinder-volume-replication
   app-manila-ganesha
   swift

.. LINKS
.. _file an issue: https://bugs.launchpad.net/charm-deployment-guide/+filebug
.. _submit a contribution: https://opendev.org/openstack/charm-deployment-guide
.. _OpenStack Charm Guide: https://docs.openstack.org/charm-guide
