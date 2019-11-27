==============================
Appendix B: OpenStack Upgrades
==============================

Overview
--------

This document outlines how to upgrade a Juju-deployed OpenStack cloud.

.. warning::

   Upgrading an OpenStack cloud is not risk-free. The procedures outlined in
   this guide should first be tested in a pre-production environment.

Please read the following before continuing:

- the OpenStack charms `Release Notes`_ for the corresponding current and
  target versions of OpenStack
- the `Known OpenStack upgrade issues`_ section in this document

Definitions
-----------

Charm upgrade
  An upgrade of the charm software which is used to deploy and manage
  OpenStack. This includes charms that manage applications which are not
  technically part of the OpenStack project such as Rabbitmq and MySQL.

OpenStack upgrade
  An upgrade of the OpenStack software (packages) that are installed and
  managed by the charms. Each OpenStack service is upgraded (by the operator)
  via its corresponding (and upgraded) charm. This constitutes an upgrade from
  one major release to the next (e.g. Stein to Train).

Ubuntu Server package upgrade
  An upgrade of the software packages on a Juju machine that are not part of
  the OpenStack project (e.g. kernel modules, QEMU binaries, KVM kernel
  module).

Series upgrade
  An upgrade of the operating system (Ubuntu) on a Juju machine (e.g. Xenial to
  Bionic). See appendix `Series upgrade`_ for more information.

Charm upgrades
--------------

All charms should be upgraded to their latest stable revision prior to
performing the OpenStack upgrade. The Juju command to use is
:command:`upgrade-charm`. For extra guidance see `Upgrading applications`_
in the Juju documentation.

Although it may be possible to upgrade some charms in parallel it is
recommended that the upgrades be performed in series (i.e. one at a time).
Verify a charm upgrade before moving on to the next.

In terms of the upgrade order, begin with 'keystone'. After that, the rest of
the charms can be upgraded in any order.

Do check the `Release Notes`_ for any special instructions regarding charm
upgrades.

.. caution::

   Any software changes that may have (exceptionally) been made to a charm
   currently running on a unit will be overwritten by the target charm during
   the upgrade.

Before upgrading, a (partial) output to :command:`juju status` may look like:

.. code::

   App       Version  Status   Scale  Charm     Store       Rev  OS      Notes
   keystone  15.0.0   active       1  keystone  jujucharms  306  ubuntu

   Unit             Workload  Agent  Machine  Public address  Ports      Message
   keystone/0*      active    idle   3/lxd/1  10.248.64.69    5000/tcp   Unit is ready

Here, as deduced from the Keystone **service** version of '15.0.0', the cloud
is running Stein. The 'keystone' **charm** however shows a revision number of
'306'. Upon charm upgrade, the service version will remain unchanged but the
charm revision is expected to increase in number.

So to upgrade this 'keystone' charm (to the most recent promulgated version in
the Charm Store):

.. code:: bash

   juju upgrade-charm keystone

The upgrade progress can be monitored via :command:`juju status`. Any
encountered problem will surface as a message in its output. This sample
(partial) output reflects a successful upgrade:

.. code::

   App       Version  Status   Scale  Charm     Store       Rev  OS      Notes
   keystone  15.0.0   active       1  keystone  jujucharms  309  ubuntu

   Unit             Workload  Agent  Machine  Public address  Ports      Message
   keystone/0*      active    idle   3/lxd/1  10.248.64.69    5000/tcp   Unit is ready

This shows that the charm now has a revision number of '309' but Keystone
itself remains at '15.0.0'.

OpenStack upgrades
------------------

Go through each of the following sections to ensure a trouble-free OpenStack
upgrade.

.. note::

   The charms only support single-step OpenStack upgrades (N+1). That is, to
   upgrade two releases forward you need to upgrade twice. You cannot skip
   releases when upgrading OpenStack with charms.

It may be worthwhile to read the upstream OpenStack `Upgrades`_ guide.

Release Notes
~~~~~~~~~~~~~

The OpenStack charms `Release Notes`_ for the corresponding current and target
versions of OpenStack **must** be consulted for any special instructions. In
particular, pay attention to services and/or configuration options that may be
retired, deprecated, or changed.

Manual intervention
~~~~~~~~~~~~~~~~~~~

It is intended that the now upgraded charms are able to accommodate all
software changes associated with the corresponding OpenStack services to be
upgraded. A new charm will also strive to produce a service as similarly
configured to the pre-upgraded service as possible. Nevertheless, there are
still times when intervention on the part of the operator may be needed, such
as when:

- a service is removed, added, or replaced
- a software bug affecting the OpenStack upgrade is present in the new charm

All known issues requiring manual intervention are documented in section `Known
OpenStack upgrade issues`_. You **must** look these over.

Verify the current deployment
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Confirm that the output for the :command:`juju status` command of the current
deployment is error-free. In addition, if monitoring is in use (e.g. Nagios),
ensure that all alerts have been resolved. This is to make certain that any
issues that may appear after the upgrade are not for pre-existing problems.

Perform a database backup
~~~~~~~~~~~~~~~~~~~~~~~~~

Before making any changes to cloud services perform a backup of the cloud
database by running the ``backup`` action on any single percona-cluster unit:

.. code:: bash

   juju run-action --wait percona-cluster/0 backup

Now transfer the backup directory to the Juju client with the intention of
subsequently storing it somewhere safe. This command will grab **all** existing
backups:

.. code:: bash

   juju scp -- -r percona-cluster/0:/opt/backups/mysql /path/to/local/directory

Permissions may first need to be altered on the remote machine.

Archive old database data
~~~~~~~~~~~~~~~~~~~~~~~~~

During the upgrade, database migrations will be run. This operation can be
optimised by first archiving any stale data (e.g. deleted instances). Do this
by running the ``archive-data`` action on any single nova-cloud-controller
unit:

.. code:: bash

   juju run-action --wait nova-cloud-controller/0 archive-data

This action may need to be run multiple times until the action output reports
'Nothing was archived'.

Purge old compute service entries
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Old compute service entries for units which are no longer part of the model
should be purged before the upgrade. These entries will show as 'down' (and be
hosted on machines no longer in the model) in the current list of compute
services:

.. code:: bash

   openstack compute service list

To remove a compute service:

.. code:: bash

   openstack compute service delete <service-id>

Disable unattended-upgrades
~~~~~~~~~~~~~~~~~~~~~~~~~~~

When performing a service upgrade on a unit that hosts multiple principle
charms (e.g. ``nova-compute`` and ``ceph-osd``), ensure that
``unattended-upgrades`` is disabled on the underlying machine for the duration
of the upgrade process. This is to prevent the other services from being
upgraded outside of Juju's control. On a unit run:

.. code:: bash

   sudo dpkg-reconfigure -plow unattended-upgrades

Upgrade order
~~~~~~~~~~~~~

The charms are put into groups to indicate the order in which their
corresponding OpenStack services should be upgraded. The order within a group
is unimportant. What matters is that all the charms within the same group are
acted on before those in the next group (i.e. upgrade all charms in group 2
before moving on to group 3). Any `Release Notes`_ guidance overrides the
information listed here. You may also consult the upstream documentation on the
subject: `Update services`_.

Each service represented by a charm in the below table will need to be upgraded
individually.

+-------+-----------------------+---------------+
| Group | Charm Name            | Charm Type    |
+=======+=======================+===============+
| 1     | keystone              | Core Identity |
+-------+-----------------------+---------------+
| 2     | ceph-mon              | Storage       |
+-------+-----------------------+---------------+
| 2     | ceph-osd              | Storage       |
+-------+-----------------------+---------------+
| 2     | ceph-fs               | Storage       |
+-------+-----------------------+---------------+
| 2     | ceph-radosgw          | Storage       |
+-------+-----------------------+---------------+
| 2     | swift-proxy           | Storage       |
+-------+-----------------------+---------------+
| 2     | swift-storage         | Storage       |
+-------+-----------------------+---------------+
| 3     | aodh                  | Control Plane |
+-------+-----------------------+---------------+
| 3     | barbican              | Control Plane |
+-------+-----------------------+---------------+
| 3     | ceilometer            | Control Plane |
+-------+-----------------------+---------------+
| 3     | cinder                | Control Plane |
+-------+-----------------------+---------------+
| 3     | designate             | Control Plane |
+-------+-----------------------+---------------+
| 3     | designate-bind        | Control Plane |
+-------+-----------------------+---------------+
| 3     | glance                | Control Plane |
+-------+-----------------------+---------------+
| 3     | gnocchi               | Control Plane |
+-------+-----------------------+---------------+
| 3     | heat                  | Control Plane |
+-------+-----------------------+---------------+
| 3     | manila                | Control Plane |
+-------+-----------------------+---------------+
| 3     | manila-generic        | Control Plane |
+-------+-----------------------+---------------+
| 3     | neutron-api           | Control Plane |
+-------+-----------------------+---------------+
| 3     | neutron-gateway       | Control Plane |
+-------+-----------------------+---------------+
| 3     | placement             | Control Plane |
+-------+-----------------------+---------------+
| 3     | nova-cloud-controller | Control Plane |
+-------+-----------------------+---------------+
| 3     | openstack-dashboard   | Control Plane |
+-------+-----------------------+---------------+
| 3     | nova-compute          | Compute       |
+-------+-----------------------+---------------+

.. important::

   OpenStack services whose software is not a part of the Ubuntu Cloud Archive
   are not represented in the above list. This type of software can only have
   their major versions changed during a series (Ubuntu) upgrade on the
   associated unit. Common charms where this applies are ``ntp``,
   ``memcached``, ``percona-cluster``, and ``rabbitmq-server``.

Perform the upgrade
~~~~~~~~~~~~~~~~~~~

The essence of a charmed OpenStack service upgrade is a change of the
corresponding machine software sources so that a more recent combination of
Ubuntu release and OpenStack release is used. This combination is based on the
`Ubuntu Cloud Archive`_ and translates to a configuration known as the "cloud
archive pocket". It takes on the following syntax:

``cloud:<ubuntu series>-<openstack-release>``

For example:

``cloud:bionic-train``

There are three methods available for performing an OpenStack service upgrade.
The appropriate method is chosen based on the actions supported by the charm.
Actions for a charm can be listed in this way:

.. code:: bash

   juju actions <charm-name>

All-in-one
^^^^^^^^^^

The "all-in-one" method upgrades an application immediately. Although it is the
quickest route, it can be harsh when applied in the context of multi-unit
applications. This is because all the units are upgraded simultaneously, and is
likely to cause a transient service outage. This method must be used if the
application has a sole unit.

.. attention::

   The "all-in-one" method should only be used when the charm does not
   support the ``openstack-upgrade`` action.

The syntax is:

.. code:: bash

   juju config <openstack-charm> openstack-origin=cloud:<cloud-archive-pocket>

Charms whose services are not technically part of the OpenStack project will
use a different charm option. The Ceph charms are a classic example:

.. code:: bash

   juju config <ceph-charm> source=cloud:<cloud-archive-pocket>

So to upgrade Cinder across all units (currently running Bionic) from Stein to
Train:

.. code:: bash

   juju config cinder openstack-origin=cloud:bionic-train

Single-unit
^^^^^^^^^^^

The "single-unit" method builds upon the "all-in-one" method by allowing for
the upgrade of individual units in a controlled manner. It requires the
enablement of charm option ``action-managed-upgrade`` and the charm action
``openstack-upgrade``.

.. attention::

   The "single-unit" method should only be used when the charm does not
   support the ``pause`` and ``resume`` actions.

As a general rule, whenever there is the possibility of upgrading units
individually, **always upgrade the application leader first.** The leader is
the unit with a ***** next to it in the :command:`juju status` output. It can
also be discovered via the CLI:

.. code:: bash

   juju run --application <application-name> is-leader

For example, to upgrade a three-unit glance application from Stein to Train
where ``glance/1`` is the leader:

.. code:: bash

   juju config glance action-managed-upgrade=True
   juju config glance openstack-origin=cloud:bionic-train

   juju run-action --wait glance/1 openstack-upgrade
   juju run-action --wait glance/0 openstack-upgrade
   juju run-action --wait glance/2 openstack-upgrade

.. note::

   The ``openstack-upgrade`` action is only available for charms whose services
   are part of the OpenStack project. For instance, you will need to use the
   "all-in-one" method for the Ceph charms.

Paused-single-unit
^^^^^^^^^^^^^^^^^^

The "paused-single-unit" method extends the "single-unit" method by allowing
for the upgrade of individual units *while paused*. Additional charm
requirements are the ``pause`` and ``resume`` actions. This method provides
more versatility by allowing a unit to be removed from service, upgraded, and
returned to service. Each of these are distinct events whose timing is chosen
by the operator.

.. attention::

   The "paused-single-unit" method is the recommended OpenStack service upgrade
   method.

For example, to upgrade a three-unit nova-compute application from Stein to
Train where ``nova-compute/0`` is the leader:

.. code:: bash

   juju config nova-compute action-managed-upgrade=True
   juju config nova-compute openstack-origin=cloud:bionic-train

   juju run-action nova-compute/0 --wait pause
   juju run-action nova-compute/0 --wait openstack-upgrade
   juju run-action nova-compute/0 --wait resume

   juju run-action nova-compute/1 --wait pause
   juju run-action nova-compute/1 --wait openstack-upgrade
   juju run-action nova-compute/1 --wait resume

   juju run-action nova-compute/2 --wait pause
   juju run-action nova-compute/2 --wait openstack-upgrade
   juju run-action nova-compute/2 --wait resume

In addition, this method also permits a possible hacluster subordinate unit,
which typically manages a VIP, to be paused so that client traffic will not
flow to the associated parent unit while its upgrade is underway.

.. attention::

   When there is an hacluster subordinate unit then it is recommended to always
   take advantage of the "pause-single-unit" method's ability to pause it
   before upgrading the parent unit.

For example, to upgrade a three-unit keystone application from Stein to Train
where ``keystone/2`` is the leader:

.. code:: bash

   juju config keystone action-managed-upgrade=True
   juju config keystone openstack-origin=cloud:bionic-train

   juju run-action keystone-hacluster/1 --wait pause
   juju run-action keystone/2 --wait pause
   juju run-action keystone/2 --wait openstack-upgrade
   juju run-action keystone/2 --wait resume
   juju run-action keystone-hacluster/1 --wait resume

   juju run-action keystone-hacluster/2 --wait pause
   juju run-action keystone/1 --wait pause
   juju run-action keystone/1 --wait openstack-upgrade
   juju run-action keystone/1 --wait resume
   juju run-action keystone-hacluster/2 --wait resume

   juju run-action keystone-hacluster/0 --wait pause
   juju run-action keystone/0 --wait pause
   juju run-action keystone/0 --wait openstack-upgrade
   juju run-action keystone/0 --wait resume
   juju run-action keystone-hacluster/0 --wait resume

.. warning::

   The hacluster subordinate unit number may not necessarily match its parent
   unit number. As in the above example, only for keystone/0 do the unit
   numbers correspond (i.e. keystone-hacluster/0 is the subordinate unit).

Verify the new deployment
~~~~~~~~~~~~~~~~~~~~~~~~~

Check for errors in :command:`juju status` output and any monitoring service.

Known OpenStack upgrade issues
------------------------------

Nova RPC version mismatches
~~~~~~~~~~~~~~~~~~~~~~~~~~~

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
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Confirm existing volumes are in an RBD pool called 'cinder':

.. code:: bash

   juju run --unit cinder/0 "rbd --name client.cinder -p cinder ls"

Sample output:

.. code::

   volume-b45066d3-931d-406e-a43e-ad4eca12cf34
   volume-dd733b26-2c56-4355-a8fc-347a964d5d55

Step 1: Deploy new topology
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Deploy the ``cinder-ceph`` charm and set the 'rbd-pool-name' to match the pool
that any existing volumes are in (see above):

.. code:: bash

   juju deploy --config rbd-pool-name=cinder cs:~openstack-charmers-next/cinder-ceph
   juju add-relation cinder cinder-ceph
   juju add-relation cinder-ceph ceph-mon
   juju remove-relation cinder ceph-mon
   juju add-relation cinder-ceph nova-compute

Step 2: Update volume configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The existing volumes now need to be updated to associate them with the newly
defined cinder-ceph backend:

.. code:: bash

   juju run-action cinder/0 rename-volume-host currenthost='cinder' \
       newhost='cinder@cinder-ceph#cinder.volume.drivers.rbd.RBDDriver'

Placement charm and nova-cloud-controller: upgrading from Stein to Train
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

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

.. code:: bash

   juju deploy --series bionic --config openstack-origin=cloud:bionic-train cs:placement
   juju run-action nova-cloud-controller/0 pause
   juju add-relation placement mysql
   juju add-relation placement keystone
   juju add-relation placement nova-cloud-controller
   openstack endpoint list # ensure placement endpoints are listening on new placment IP address
   juju run-action nova-cloud-controller/0 resume

Only after these steps have been completed can nova-cloud-controller be
upgraded. Here we upgrade all units simultaneously but see the
`Paused-single-unit`_ service upgrade method for a more controlled approach:

.. code:: bash

   juju config nova-cloud-controller openstack-origin=cloud:bionic-train

Neutron LBaaS: upgrading from Stein to Train
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

As of Train, support for Neutron LBaaS has been retired. The load-balancing
services are now provided by `Octavia LBaaS`_. There is no automatic migration
path, please review the `Octavia LBaaS`_ appendix for more information.

.. LINKS

.. _Series upgrade: app-series-upgrade
.. _Release Notes: https://docs.openstack.org/charm-guide/latest/release-notes.html
.. _Upgrading applications: https://jaas.ai/docs/upgrading-applications
.. _Ubuntu Cloud Archive: https://wiki.ubuntu.com/OpenStack/CloudArchive
.. _Upgrades: https://docs.openstack.org/operations-guide/ops-upgrades.html
.. _Update services: https://docs.openstack.org/operations-guide/ops-upgrades.html#update-services
.. _Octavia LBaaS: app-octavia

.. BUGS
.. _LP #1825999: https://bugs.launchpad.net/charm-nova-compute/+bug/1825999
.. _LP #1809190: https://bugs.launchpad.net/charm-neutron-gateway/+bug/1809190
