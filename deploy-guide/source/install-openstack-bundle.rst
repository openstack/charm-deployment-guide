Install OpenStack from a bundle
===============================

:doc:`Stepping through the deployment <install-openstack>` of each OpenStack
application is the best way to understanding how OpenStack and Juju operate, and
how each application relates to one another. But it's a labour intensive
process.

A bundle allows you to accomplish the same deployment with a single command:

.. code:: bash

    juju deploy openstack.bundle

A `bundle <https://jujucharms.com/docs/stable/charms-bundles>`__, as used above,
encapsulates the entire deployment process, including all applications, their
configuration parameters and any relations that need to be made. Generally, you
can use a local file, as above, or deploy a curated bundle from the `charm
store <https://jujucharms.com/store>`__.

For our project, `download
<https://api.jujucharms.com/charmstore/v5/openstack-base/archive>`__ the
`OpenStack <https://jujucharms.com/openstack-base/>`__ and deploy OpenStack
using the above command.

.. note::

    You will probably need to edit the bundle information to match the actual
    hardware that you have.

The speed of the deployment depends on your hardware, but may take some time.
Monitor the output of ``juju status`` to see when everything is ready.

Next steps
----------

See the :ref:`Install OpenStack <test_openstack>`
documentation for details on testing your OpenStack deployment, or jump directly
to :doc:`Configure OpenStack <config-openstack>` to start using OpenStack
productively as quickly as possible.

.. raw:: html

   <!-- LINKS -->
