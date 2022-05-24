:orphan:

=======================
Upgrade: Stein to Train
=======================

This page contains notes specific to the Stein to Train upgrade path. See the
main :doc:`upgrade-openstack` page for full coverage.

New placement charm
-------------------

As of OpenStack Train, the Placement API is managed by the new `placement`_
charm and is no longer managed by the nova-cloud-controller charm. The upgrade
to Train therefore involves some coordination to transition to the new API
endpoints.

Prior to upgrading nova-cloud-controller services to Train, the placement charm
must be deployed for Train and related to the Stein-based nova-cloud-controller
application. It is important that the nova-cloud-controller unit leader is
paused while the API transition occurs (paused prior to adding relations for
the placement charm) as the placement charm will migrate existing placement
tables from the nova_api database to a new placement database. Once the new
placement endpoints are registered, nova-cloud-controller can be resumed.

Here are example commands for the process just described:

.. code-block:: none

   juju deploy --series bionic --config openstack-origin=cloud:bionic-train cs:placement
   juju run-action --wait nova-cloud-controller/leader pause
   juju add-relation placement percona-cluster
   juju add-relation placement keystone
   juju add-relation placement nova-cloud-controller

List endpoints and ensure placement endpoints are now listening on the new
placement IP address. Follow this up by resuming nova-cloud-controller:

.. code-block:: none

   openstack endpoint list
   juju run-action --wait nova-cloud-controller/leader resume

Finally, upgrade the nova-cloud-controller services. Below all units are
upgraded simultaneously but see the `paused-single-unit`_ service upgrade
method for a more controlled approach:

.. code-block:: none

   juju config nova-cloud-controller openstack-origin=cloud:bionic-train

The Compute service (nova-compute) should then be upgraded.

Placement endpoints not updated in Keystone service catalog
-----------------------------------------------------------

When the placement charm is deployed during the upgrade to Train (as described
above) the Keystone service catalog is not updated accordingly. This issue is
tracked in bug `LP #1928992`_, which also includes an explicit workaround
(comment #4).

Neutron LBaaS retired
---------------------

As of Train, support for Neutron LBaaS has been retired. The load-balancing
services are now provided by Octavia LBaaS. There is no automatic migration
path, please review the :doc:`app-octavia` page for more information.

Designate encoding issue
------------------------

When upgrading Designate to Train, there is an encoding issue between the
designate-producer and memcached that causes the designate-producer to crash.
See bug `LP #1828534`_. This can be resolved by restarting the memcached service.

.. code-block:: none

   juju run --application=memcached 'sudo systemctl restart memcached'

.. LINKS
.. _placement: https://charmhub.io/placement
.. _paused-single-unit: upgrade-openstack.html#paused-single-unit

.. BUGS
.. _LP #1828534: https://bugs.launchpad.net/charm-designate/+bug/1828534
.. _LP #1928992: https://bugs.launchpad.net/charm-deployment-guide/+bug/1928992
