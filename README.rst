=================================
OpenStack Charms Deployment Guide
=================================

.. image:: http://governance.openstack.org/badges/charm-deployment-guide.svg
   :target: http://governance.openstack.org/reference/tags/index.html

The `OpenStack Charms Deployment Guide`_ is the main source of information for
the usage of the `OpenStack Charms`_.

Building
--------

To build the guide run this simple command:

.. code-block:: none

   tox

The resulting HTML files will be found in the ``deploy-guide/build/html``
directory. These can be opened individually with a web browser or hosted by a
local web server.

Contributing
------------

Documentation issues can be filed on `Launchpad`_.

This repository is under version control and is managed via the `OpenStack
Gerrit system`_ (see the `OpenDev Developer’s Guide`_). For specific guidance
on working with the documentation hosted on https://docs.openstack.org please
read the `OpenStack Documentation Contributor Guide`_.

.. LINKS
.. _OpenStack Charms Deployment Guide: https://docs.openstack.org/project-deploy-guide/charm-deployment-guide
.. _OpenStack Charms: https://launchpad.net/openstack-charms
.. _Launchpad: https://bugs.launchpad.net/charm-deployment-guide/+filebug
.. _OpenStack Gerrit system: https://review.openstack.org
.. _OpenDev Developer’s Guide: https://docs.openstack.org/infra/manual/developers.html
.. _OpenStack Documentation Contributor Guide: https://docs.openstack.org/doc-contrib-guide/index.html
