:orphan:

===========================================
placement charm: OpenStack upgrade to Train
===========================================

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

   juju deploy --series bionic --config openstack-origin=cloud:bionic-train placement
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

.. LINKS
.. _placement: https://jaas.ai/placement
.. _paused-single-unit: upgrade-openstack.html#paused-single-unit
