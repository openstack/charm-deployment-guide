:orphan:

===============================
Install OpenStack from a bundle
===============================

Charmed OpenStack is often deployed via a charm bundle. A bundle encompasses
multiple charms, their configuration options, and various optional elements
such as hardware and network constraints. See `Charm bundles`_ in the Juju
documentation.

.. tip::

   The `Install OpenStack`_ page shows how to install by deploying,
   configuring, and relating applications on an individual basis using Juju. It
   is the recommended install method for getting a high level view of how
   OpenStack is put together. It also provides an opportunity to gain
   experience with Juju, which will in turn prepare you for post-deployment
   management of the cloud.

To arrive at a truly customised deployment, while taking advantage of an
official base bundle, a secondary (overlay) bundle can be applied to override
and fine-tune certain elements of the original bundle.

The bundle recommended here is the stable release of `openstack-base`_. It is
used throughout Charmed OpenStack documentation as a reference bundle.

The bundle will probably need to be modified to accommodate for factors in the
local environment (e.g. hardware), or as mentioned, overridden with an overlay
bundle.

Deploy the bundle now. Follow the instructions provided in the stable
`openstack-base README`_.

Once deployed, go on to `Configure OpenStack`_.

Finally, once cloud functionality has been verified see the `OpenStack
Administrator Guides`_ for long-term guidance.

.. LINKS
.. _Install OpenStack: install-openstack
.. _Configure OpenStack: configure-openstack.html
.. _Charm bundles: https://juju.is/docs/sdk/bundles
.. _MAAS: https://maas.io
.. _openstack-base: https://github.com/openstack-charmers/openstack-bundles/tree/master/stable/openstack-base
.. _openstack-base README: https://github.com/openstack-charmers/openstack-bundles/blob/master/stable/openstack-base/README.md
.. _OpenStack Administrator Guides: http://docs.openstack.org/admin
