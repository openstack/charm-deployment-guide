:orphan:

.. _cloud_topology_ha:

==================================
Cloud topology for HA applications
==================================

.. note::

   The information on this page is associated with the topic of :ref:`OpenStack
   high availability <ha>`. See that page for background information.

**The below is to be edited. Based on a cloud from Solutions QA.**

This page contains the analysis of cloud machines. The ideal is to do this for
every machine in a cloud in order to determine the *cloud topology*. Six
machines are featured here. They represent a good cross-section of an *Ubuntu
OpenStack* cloud. See :ref:`Reference cloud for HA applications
<reference_cloud_ha>` for the cloud upon which this exercise is based.

Generally speaking, the cloud nodes are hyperconverged and this is the case for
three of the chosen machines, numbered **17**, **18**, and **20**. Yet this
analysis also looks at a trio of nodes dedicated to the `Landscape project`_:
machines **3**, **11**, and **12**, each of which are not hyperconverged.

.. note::

   Juju applications can be given custom names at deployement time (see
   `Application groups`_ in the Juju documentation). This document will call
   out these `named applications` wherever they occur.

.. LINKS
.. _Application groups: https://jaas.ai/docs/application-groups
.. _Landscape project: https://landscape.canonical.com

