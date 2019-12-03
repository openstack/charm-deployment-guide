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
Charms`_. For accessibility reasons, the cloud will be minimal, while remaining
capable of both performing some real work and scaling to fit more ambitious
projects.

Software versions used in this guide are as follows:

* **Ubuntu 18.04 LTS** (Bionic) for the MAAS system, its cluster nodes, and all
  cloud nodes
* **MAAS 2.6.2**
* **Juju 2.7.0**

.. note::

   For OpenStack Charms project information, development guidelines, release
   notes, and release schedules, please refer to the `OpenStack Charm Guide`_.

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
