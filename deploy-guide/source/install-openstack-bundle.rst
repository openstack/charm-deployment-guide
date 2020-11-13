===============================
Install OpenStack from a bundle
===============================

A Juju charm *bundle* is an encapsulation of a multitude of charm deployments,
and includes all the associated relations and configurations that are required
(see `Charm bundles`_ in the Juju documentation). It is possible to therefore
install OpenStack from a bundle.

.. tip::

   The `Install OpenStack`_ page shows how to install by deploying,
   configuring, and relating applications on an individual basis using Juju. It
   is the recommended install method for getting a high level view of how
   OpenStack is put together. It also provides an opportunity to gain
   experience with Juju, which will in turn prepare you for post-deployment
   management of the cloud.

The bundle featured here provides a minimal OpenStack cloud and assumes that
`MAAS`_ is used as a backing cloud to Juju. Due to unknown factors in the local
environment (usually hardware-related) the bundle will most likely need to be
modified prior to deployment. The bundle and its deployment are described in
great detail in its Charm Store entry here: `openstack-base`_.

Once the bundle configuration has been confirmed OpenStack can be deployed:

.. code-block:: none

   juju deploy /path/to/bundle/file

The time required for the install to complete will depend on the hardware
capabilities of the underlying MAAS nodes. Once finished, you should go on to
`Configure OpenStack`_ if not already done.

Finally, once cloud functionality has been verified see the `OpenStack
Administrator Guides`_ for long-term guidance.

.. LINKS
.. _Install OpenStack: install-openstack
.. _Configure OpenStack: configure-openstack.html
.. _Charm bundles: https://jaas.ai/docs/charm-bundles
.. _MAAS: https://maas.io
.. _openstack-base: https://jaas.ai/openstack-base
.. _OpenStack Administrator Guides: http://docs.openstack.org/admin
