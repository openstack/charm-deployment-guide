=================================
OpenStack Charms Deployment Guide
=================================

The OpenStack Charms Deployment Guide is the main source of information for
OpenStack Charms usage. A search facility is available as a separate
:ref:`search`.

Also included is a wealth of information in the form of appendices. These
cover a wide variety of subjects, such as an elaboration of a specific charm
feature or instructions for upgrading an OpenStack cloud.

To help improve this guide you may `file an issue`_ or `submit a
contribution`_.

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
   app-upgrade-openstack
   app-series-upgrade
   upgrade-special
   upgrade-issues

.. toctree::
   :caption: Security
   :maxdepth: 1

   security-overview
   keystone
   app-policy-overrides

.. toctree::
   :caption: Appendices
   :maxdepth: 1

   Vault <app-vault>
   Certificate lifecycle management <app-certificate-management>
   Encryption at Rest <app-encryption-at-rest>
   Additional Nova cells <app-nova-cells>
   Octavia LBaaS <app-octavia>
   PCI passthrough <app-pci-passthrough-gpu>
   Ceph erasure coding <app-erasure-coding>
   Ceph RADOS Gateway multisite replication <app-rgw-multisite>
   Ceph RBD mirroring <app-ceph-rbd-mirror>
   Ceph iSCSI <app-ceph-iscsi>
   Masakari <app-masakari>
   OVN <app-ovn>
   Managing power events <app-managing-power-events>
   Manila Ganesha <app-manila-ganesha>
   Swift usage <app-swift>
   NIC hardware offload <app-hardware-offload>
   High availability <app-ha>
   TrilioVault Data Protection <app-trilio-vault>
   Bridge interface configuration <app-bridge-interface-configuration>

.. LINKS
.. _file an issue: https://bugs.launchpad.net/charm-deployment-guide/+filebug
.. _submit a contribution: https://opendev.org/openstack/charm-deployment-guide
.. _OpenStack Charm Guide: https://docs.openstack.org/charm-guide
