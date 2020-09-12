.. OpenStack documentation master file, created by
   sphinx-quickstart on Fri Jun 30 11:14:11 2017.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

=================================
OpenStack Charms Deployment Guide
=================================

Overview
--------

The main purpose of the OpenStack Charms Deployment Guide is to demonstrate how
to build a multi-node OpenStack cloud with `MAAS`_, `Juju`_, and `OpenStack
Charms`_. For easy adoption the cloud will be minimal. Nevertheless, it will be
capable of both performing some real work and scaling to fit more ambitious
projects. High availability will not be implemented beyond natively HA
applications (Ceph, MySQL8, OVN, Swift, and RabbitMQ).

.. note::

   For OpenStack Charms project information, development guidelines, release
   notes, and release schedules, please refer to the `OpenStack Charm Guide`_.

Requirements
------------

The software versions used in this guide are as follows:

* **Ubuntu 20.04 LTS (Focal)** for the MAAS server, Juju client, Juju
  controller, and all cloud nodes (including containers)
* **MAAS 2.8.2**
* **Juju 2.8.1**
* **OpenStack Ussuri**

Hardware requirements are listed on the :doc:`Install MAAS <install-maas>`
page.

Appendices
----------

The guide also includes a wealth of information in the form of appendices.
These cover a wide variety of subjects, such as an elaboration of a specific
charm feature, how to upgrade an OpenStack cloud, or how to manage power events
in a cloud.

Help improve this guide
-----------------------

To help improve this guide you may `file an issue`_ or `submit a
contribution`_.

Table of contents
-----------------

.. toctree::
   :maxdepth: 2

   install-maas.rst
   install-juju.rst
   install-openstack.rst
   install-openstack-bundle.rst
   config-openstack.rst
   app.rst

* :ref:`search`

.. LINKS
.. _MAAS: https://maas.io
.. _Juju: https://jaas.ai
.. _OpenStack Charms: https://docs.openstack.org/charm-guide
.. _OpenStack Charm Guide: https://docs.openstack.org/charm-guide
.. _file an issue: https://bugs.launchpad.net/charm-deployment-guide/+filebug
.. _submit a contribution: https://opendev.org/openstack/charm-deployment-guide
