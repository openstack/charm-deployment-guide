=================================
OpenStack Charms Deployment Guide
=================================

The OpenStack Charms Deployment Guide demonstrates how to build an OpenStack
cloud by making use of the tools provided by the OpenStack Charms project. Its
aim is to impart a solid understanding of how the cloud is constructed by
deploying and configuring Juju charms manually.

.. note::

   For all other information concerning the OpenStack Charms project please
   refer to the :doc:`cg:index`.

   The Charm Guide also documents an automated cloud deployment method that
   uses a Juju bundle (see the :doc:`cg:getting-started/index` tutorial).

This guide can be :ref:`searched here <search>`. To help improve it you can
`file an issue`_ or `submit a contribution`_.

Software
--------

The software versions used in this guide are as follows:

* **Ubuntu 22.04 LTS (Jammy)** for the Juju client, Juju controller, and all
  cloud nodes (including containers)
* **MAAS 3.3.3**
* **Juju 2.9.43**
* **OpenStack 2023.1 (Antelope)**

Cloud description
-----------------

For easy adoption the cloud will be minimal. Nevertheless, it will be capable
of both performing some real work and scaling to fit more ambitious projects.
High availability will not be implemented beyond natively HA applications
(Ceph, MySQL, OVN, and RabbitMQ).

Procedure
---------

The procedure consists of the following steps:

.. toctree::
   :maxdepth: 1

   install-maas
   install-juju
   install-openstack
   configure-openstack

.. LINKS
.. _file an issue: https://bugs.launchpad.net/charm-deployment-guide/+filebug
.. _submit a contribution: https://opendev.org/openstack/charm-deployment-guide
