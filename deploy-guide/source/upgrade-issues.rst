==============
Upgrade issues
==============

This page documents upgrade issues and notes. These may apply to either of the
three upgrade types (charms, OpenStack, series).

The items on this page are distinct from those found on the following pages:

* the `Various issues`_ page
* the `Special charm procedures`_ page

The issues are organised by upgrade type.

Charm upgrades
--------------

rabbitmq-server charm
~~~~~~~~~~~~~~~~~~~~~

A timing issue has been observed during the upgrade of the rabbitmq-server
charm (see bug `LP #1912638`_ for tracking). If it occurs the resulting hook
error can be resolved with:

.. code-block:: none

   juju resolved rabbitmq-server/N

openstack-dashboard charm: upgrading to revision 294
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

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

Multiple charms: option ``worker-multiplier``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Starting with OpenStack Charms 21.04 any charm that supports the
``worker-multiplier`` configuration option will, upon upgrade, modify the
active number of service workers according to the following: if the option is
not set explicitly the number of workers will be capped at four regardless of
whether the unit is containerised or not. Previously, the cap applied only to
containerised units.

OpenStack upgrades
------------------

Nova RPC version mismatches: upgrading Neutron and Nova
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If it is not possible to upgrade Neutron and Nova within the same maintenance
window, be mindful that the RPC communication between nova-cloud-controller,
nova-compute, and nova-api-metadata is very likely to cause several errors
while those services are not running the same version. This is due to the fact
that currently those charms do not support RPC version pinning or
auto-negotiation.

See bug `LP #1825999`_.

neutron-gateway charm: upgrading from Mitaka to Newton
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Between the Mitaka and Newton OpenStack releases, the ``neutron-gateway`` charm
added two options, ``bridge-mappings`` and ``data-port``, which replaced the
(now) deprecated ``ext-port`` option. This was to provide for more control over
how ``neutron-gateway`` can configure external networking. Unfortunately, the
charm was only designed to work with either ``ext-port`` (no longer
recommended) *or* ``bridge-mappings`` and ``data-port``.

See bug `LP #1809190`_.

cinder/ceph topology change: upgrading from Newton to Ocata
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If cinder is directly related to ceph-mon rather than via cinder-ceph then
upgrading from Newton to Ocata will result in the loss of some block storage
functionality, specifically live migration and snapshotting. To remedy this
situation the deployment should migrate to using the cinder-ceph charm. This
can be done after the upgrade to Ocata.

.. warning::

   Do not attempt to migrate a deployment with existing volumes to use the
   cinder-ceph charm prior to Ocata.

The intervention is detailed in the below three steps.

Step 0: Check existing configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Confirm existing volumes are in an RBD pool called 'cinder':

.. code-block:: none

   juju run --unit cinder/0 "rbd --name client.cinder -p cinder ls"

Sample output:

.. code-block:: none

   volume-b45066d3-931d-406e-a43e-ad4eca12cf34
   volume-dd733b26-2c56-4355-a8fc-347a964d5d55

Step 1: Deploy new topology
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Deploy the ``cinder-ceph`` charm and set the 'rbd-pool-name' to match the pool
that any existing volumes are in (see above):

.. code-block:: none

   juju deploy --config rbd-pool-name=cinder cinder-ceph
   juju add-relation cinder cinder-ceph
   juju add-relation cinder-ceph ceph-mon
   juju remove-relation cinder ceph-mon
   juju add-relation cinder-ceph nova-compute

Step 2: Update volume configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The existing volumes now need to be updated to associate them with the newly
defined cinder-ceph backend:

.. code-block:: none

   juju run-action cinder/0 rename-volume-host currenthost='cinder' \
       newhost='cinder@cinder-ceph#cinder.volume.drivers.rbd.RBDDriver'

Keystone and Fernet tokens: upgrading from Queens to Rocky
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

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

Neutron LBaaS: upgrading from Stein to Train
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

As of Train, support for Neutron LBaaS has been retired. The load-balancing
services are now provided by `Octavia LBaaS`_. There is no automatic migration
path, please review the `Octavia LBaaS`_ appendix for more information.

Designate: upgrading from Stein to Train
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When upgrading Designate to Train, there is an encoding issue between the
designate-producer and memcached that causes the designate-producer to crash.
See bug `LP #1828534`_. This can be resolved by restarting the memcached service.

.. code-block:: none

   juju run --application=memcached 'sudo systemctl restart memcached'

Ceph BlueStore mistakenly enabled during OpenStack upgrade
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The Ceph BlueStore storage backend is enabled by default when Ceph Luminous is
detected. Therefore it is possible for a non-BlueStore cloud to acquire
BlueStore by default after an OpenStack upgrade (Luminous first appeared in
Queens). Problems will occur if storage is scaled out without first disabling
BlueStore (set ceph-osd charm option ``bluestore`` to 'False'). See bug `LP
#1885516`_ for details.

Placement: endpoints not updated in Keystone service catalog
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When the placement charm is deployed during the upgrade to OpenStack Train (as
described in :doc:`placement charm: OpenStack upgrade to Train
<placement-charm-upgrade-to-train>`) the Keystone service catalog is not
updated accordingly. This issue is tracked in bug `LP #1928992`_, which also
includes an explicit workaround (comment #4).

.. _ceph-require-osd-release:

Ceph: option ``require-osd-release``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Before upgrading Ceph its ``require-osd-release`` option should be set to the
current Ceph release (e.g. 'nautilus' if upgrading to Octopus). Failing to do
so may cause the upgrade to fail, rendering the cluster inoperable.

On any ceph-mon unit, the current value of the option can be queried with:

.. code-block:: none

   sudo ceph osd dump | grep require_osd_release

If it needs changing, it can be done manually on any ceph-mon unit. Here the
current release is Nautilus:

.. code-block:: none

   sudo ceph osd require-osd-release nautilus

In addition, upon completion of the upgrade, the option should be set to the
new release. Here the new release is Octopus:

.. code-block:: none

   sudo ceph osd require-osd-release octopus

The charms should be able to respond intelligently to these two situations. Bug
`LP #1929254`_ is for tracking this effort.

Series upgrades
---------------

DNS HA: upgrade to focal
~~~~~~~~~~~~~~~~~~~~~~~~

DNS HA has been reported to not work on the focal series. See `LP #1882508`_
for more information.

Upgrading while Vault is sealed
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If a series upgrade is attempted while Vault is sealed then manual intervention
will be required (see bugs `LP #1886083`_ and `LP #1890106`_). The vault leader
unit (which will be in error) will need to be unsealed and the hook error
resolved. The `vault charm`_ README has unsealing instructions, and the hook
error can be resolved with:

.. code-block:: none

   juju resolved vault/N

.. LINKS
.. _Release Notes: https://docs.openstack.org/charm-guide/latest/release-notes.html
.. _Ubuntu Cloud Archive: https://wiki.ubuntu.com/OpenStack/CloudArchive
.. _Upgrades: https://docs.openstack.org/operations-guide/ops-upgrades.html
.. _Update services: https://docs.openstack.org/operations-guide/ops-upgrades.html#update-services
.. _Keystone Fernet Token Implementation: https://specs.openstack.org/openstack/charm-specs/specs/rocky/implemented/keystone-fernet-tokens.html
.. _Octavia LBaaS: app-octavia.html
.. _Various issues: various-issues.html
.. _Special charm procedures: upgrade-special.html
.. _vault charm: https://opendev.org/openstack/charm-vault/src/branch/master/src/README.md#unseal-vault

.. BUGS
.. _LP #1825999: https://bugs.launchpad.net/charm-nova-compute/+bug/1825999
.. _LP #1809190: https://bugs.launchpad.net/charm-neutron-gateway/+bug/1809190
.. _LP #1853173: https://bugs.launchpad.net/charm-openstack-dashboard/+bug/1853173
.. _LP #1828534: https://bugs.launchpad.net/charm-designate/+bug/1828534
.. _LP #1882508: https://bugs.launchpad.net/charm-deployment-guide/+bug/1882508
.. _LP #1885516: https://bugs.launchpad.net/charm-deployment-guide/+bug/1885516
.. _LP #1886083: https://bugs.launchpad.net/vault-charm/+bug/1886083
.. _LP #1890106: https://bugs.launchpad.net/vault-charm/+bug/1890106
.. _LP #1912638: https://bugs.launchpad.net/charm-rabbitmq-server/+bug/1912638
.. _LP #1928992: https://bugs.launchpad.net/charm-deployment-guide/+bug/1928992
.. _LP #1929254: https://bugs.launchpad.net/charm-ceph-osd/+bug/1929254
