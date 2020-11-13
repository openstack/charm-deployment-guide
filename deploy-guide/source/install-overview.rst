========
Overview
========

The purpose of the Installation section is to demonstrate how to build a
multi-node OpenStack cloud with `MAAS`_, `Juju`_, and `OpenStack Charms`_. For
easy adoption the cloud will be minimal. Nevertheless, it will be capable of
both performing some real work and scaling to fit more ambitious projects. High
availability will not be implemented beyond natively HA applications (Ceph,
MySQL8, OVN, Swift, and RabbitMQ).

The software versions used in this guide are as follows:

* **Ubuntu 20.04 LTS (Focal)** for the MAAS server, Juju client, Juju
  controller, and all cloud nodes (including containers)
* **MAAS 2.8.2**
* **Juju 2.8.1**
* **OpenStack Ussuri**

Proceed to the :doc:`Install MAAS <install-maas>` page to begin your
installation journey. Hardware requirements are also listed there.

.. LINKS
.. _MAAS: https://maas.io
.. _Juju: https://juju.is
.. _OpenStack Charms: https://docs.openstack.org/charm-guide
