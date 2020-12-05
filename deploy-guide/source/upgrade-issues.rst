====================
Known upgrade issues
====================

This section documents known software upgrade issues.

DNS HA with the focal series
----------------------------

DNS HA has been reported to not work on the focal series. See `LP #1882508`_
for more information.

Nova RPC version mismatches
---------------------------

If it is not possible to upgrade Neutron and Nova within the same maintenance
window, be mindful that the RPC communication between nova-cloud-controller,
nova-compute, and nova-api-metadata is very likely to cause several errors
while those services are not running the same version. This is due to the fact
that currently those charms do not support RPC version pinning or
auto-negotiation.

See bug `LP #1825999`_.

neutron-gateway charm: upgrading from Mitaka to Newton
------------------------------------------------------

Between the Mitaka and Newton OpenStack releases, the ``neutron-gateway`` charm
added two options, ``bridge-mappings`` and ``data-port``, which replaced the
(now) deprecated ``ext-port`` option. This was to provide for more control over
how ``neutron-gateway`` can configure external networking. Unfortunately, the
charm was only designed to work with either ``ext-port`` (no longer
recommended) *or* ``bridge-mappings`` and ``data-port``.

See bug `LP #1809190`_.

cinder/ceph topology change: upgrading from Newton to Ocata
-----------------------------------------------------------

If ``cinder`` is directly related to ``ceph-mon`` rather than via
``cinder-ceph`` then upgrading from Newton to Ocata will result in the loss of
some block storage functionality, specifically live migration and snapshotting.
To remedy this situation the deployment should migrate to using the cinder-ceph
charm. This can be done after the upgrade to Ocata.

.. warning::

   Do not attempt to migrate a deployment with existing volumes to use the
   ``cinder-ceph`` charm prior to Ocata.

The intervention is detailed in the below three steps.

Step 0: Check existing configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Confirm existing volumes are in an RBD pool called 'cinder':

.. code-block:: none

   juju run --unit cinder/0 "rbd --name client.cinder -p cinder ls"

Sample output:

.. code-block:: none

   volume-b45066d3-931d-406e-a43e-ad4eca12cf34
   volume-dd733b26-2c56-4355-a8fc-347a964d5d55

Step 1: Deploy new topology
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Deploy the ``cinder-ceph`` charm and set the 'rbd-pool-name' to match the pool
that any existing volumes are in (see above):

.. code-block:: none

   juju deploy --config rbd-pool-name=cinder cinder-ceph
   juju add-relation cinder cinder-ceph
   juju add-relation cinder-ceph ceph-mon
   juju remove-relation cinder ceph-mon
   juju add-relation cinder-ceph nova-compute

Step 2: Update volume configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The existing volumes now need to be updated to associate them with the newly
defined cinder-ceph backend:

.. code-block:: none

   juju run-action cinder/0 rename-volume-host currenthost='cinder' \
       newhost='cinder@cinder-ceph#cinder.volume.drivers.rbd.RBDDriver'

Keystone and Fernet tokens: upgrading from Queens to Rocky
----------------------------------------------------------

Starting with OpenStack Rocky only the Fernet format for authentication tokens
is supported. Therefore, prior to upgrading Keystone to Rocky a transition must
be made from the legacy format (of UUID) to Fernet.

Fernet support is available upstream (and in the keystone charm) starting with
Ocata so the transition can be made on either Ocata, Pike, or Queens.

A keystone charm upgrade will not alter the token format. The charm's
``token-provider`` option must be used to make the transition:

.. code-block:: none

   juju config keystone token-provider=fernet

This change may result in a minor control plane outage but any running
instances will remain unaffected.

The ``token-provider`` option has no effect starting with Rocky, where the
charm defaults to Fernet and where upstream removes support for UUID. See
`Keystone Fernet Token Implementation`_ for more information.

Placement charm and nova-cloud-controller: upgrading from Stein to Train
------------------------------------------------------------------------

As of Train, the placement API is managed by the new ``placement`` charm and is
no longer managed by the ``nova-cloud-controller`` charm. The upgrade to Train
therefore requires some coordination to transition to the new API endpoints.

Prior to upgrading nova-cloud-controller to Train, the placement charm must be
deployed for Train and related to the Stein-based nova-cloud-controller. It is
important that the nova-cloud-controller unit leader is paused while the API
transition occurs (paused prior to adding relations for the placement charm) as
the placement charm will migrate existing placement tables from the nova_api
database to a new placement database. Once the new placement endpoints are
registered, nova-cloud-controller can be resumed.

Here's an example of the steps just described where `nova-cloud-controller/0`
is the leader:

.. code-block:: none

   juju deploy --series bionic --config openstack-origin=cloud:bionic-train cs:placement
   juju run-action nova-cloud-controller/0 pause
   juju add-relation placement mysql
   juju add-relation placement keystone
   juju add-relation placement nova-cloud-controller
   openstack endpoint list # ensure placement endpoints are listening on new placment IP address
   juju run-action nova-cloud-controller/0 resume

Only after these steps have been completed can nova-cloud-controller be
upgraded. Here we upgrade all units simultaneously but see the
ref:`paused-single-unit <paused_single_unit>` service upgrade method for a more
controlled approach:

.. code-block:: none

   juju config nova-cloud-controller openstack-origin=cloud:bionic-train

Neutron LBaaS: upgrading from Stein to Train
--------------------------------------------

As of Train, support for Neutron LBaaS has been retired. The load-balancing
services are now provided by `Octavia LBaaS`_. There is no automatic migration
path, please review the `Octavia LBaaS`_ appendix for more information.

openstack-dashboard charm: upgrading to revision 294
----------------------------------------------------

When Horizon is configured with TLS (openstack-dashboard charm option
``ssl-cert``) revisions 294 and 295 of the charm have been reported to break
the dashboard (see bug `LP #1853173`_). The solution is to upgrade to a working
revision. A temporary workaround is to disable TLS without upgrading.

.. note::

   Most users will not be impacted by this issue as the recommended approach is
   to always upgrade to the latest revision.

To upgrade to revision 293:

.. code-block:: none

   juju upgrade-charm openstack-dashboard --revision 293

To upgrade to revision 296:

.. code-block:: none

   juju upgrade-charm openstack-dashboard --revision 296

To disable TLS:

.. code-block:: none

   juju config enforce-ssl=false openstack-dashboard

Designate upgrades to Train
---------------------------

When upgrading Designate to Train, there is an encoding issue between the
designate-producer and memcached that causes the designate-producer to crash.
See bug `LP #1828534`_. This can be resolved by restarting the memcached service.

.. code-block:: none

   juju run --application=memcached 'sudo systemctl restart memcached'

.. LINKS
.. _Release Notes: https://docs.openstack.org/charm-guide/latest/release-notes.html
.. _Ubuntu Cloud Archive: https://wiki.ubuntu.com/OpenStack/CloudArchive
.. _Upgrades: https://docs.openstack.org/operations-guide/ops-upgrades.html
.. _Update services: https://docs.openstack.org/operations-guide/ops-upgrades.html#update-services
.. _Keystone Fernet Token Implementation: https://specs.openstack.org/openstack/charm-specs/specs/rocky/implemented/keystone-fernet-tokens.html
.. _Octavia LBaaS: app-octavia.html

.. BUGS
.. _LP #1825999: https://bugs.launchpad.net/charm-nova-compute/+bug/1825999
.. _LP #1809190: https://bugs.launchpad.net/charm-neutron-gateway/+bug/1809190
.. _LP #1853173: https://bugs.launchpad.net/charm-openstack-dashboard/+bug/1853173
.. _LP #1828534: https://bugs.launchpad.net/charm-designate/+bug/1828534
.. _LP #1882508: https://bugs.launchpad.net/charm-deployment-guide/+bug/1882508
