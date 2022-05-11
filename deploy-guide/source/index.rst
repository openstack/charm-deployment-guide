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
   app-certificate-management

.. toctree::
   :caption: Compute
   :maxdepth: 1

   ironic
   nova-cells
   pci-passthrough
   NFV <nfv>

.. toctree::
   :caption: Networking
   :maxdepth: 1

   app-ovn
   app-hardware-offload
   app-octavia
   configure-bridge

.. toctree::
   :caption: High availability
   :maxdepth: 1

   app-masakari

.. toctree::
   :caption: Storage
   :maxdepth: 1

   encryption-at-rest
   ceph-erasure-coding
   rgw-multisite
   app-ceph-rbd-mirror
   cinder-volume-replication
   manila-ganesha
   swift

.. LINKS
.. _file an issue: https://bugs.launchpad.net/charm-deployment-guide/+filebug
.. _submit a contribution: https://opendev.org/openstack/charm-deployment-guide
.. _OpenStack Charm Guide: https://docs.openstack.org/charm-guide
