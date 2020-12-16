=================
OpenStack upgrade
=================

This document outlines how to upgrade a Juju-deployed OpenStack cloud.

.. warning::

   Upgrading an OpenStack cloud is not risk-free. The procedures outlined in
   this guide should first be tested in a pre-production environment.

Please read the following before continuing:

* the :doc:`Upgrades overview <upgrade-overview>` page
* the OpenStack charms `Release Notes`_ for the corresponding current and
  target versions of OpenStack
* the :doc:`Upgrade issues <upgrade-issues>` page

.. note::

   The charms only support single-step OpenStack upgrades (N+1). That is, to
   upgrade two releases forward you need to upgrade twice. You cannot skip
   releases when upgrading OpenStack with charms.

It may be worthwhile to read the upstream OpenStack `Upgrades`_ guide.

Release Notes
-------------

The OpenStack charms `Release Notes`_ for the corresponding current and target
versions of OpenStack **must** be consulted for any special instructions. In
particular, pay attention to services and/or configuration options that may be
retired, deprecated, or changed.

Manual intervention
-------------------

It is intended that the now upgraded charms are able to accommodate all
software changes associated with the corresponding OpenStack services to be
upgraded. A new charm will also strive to produce a service as similarly
configured to the pre-upgraded service as possible. Nevertheless, there are
still times when intervention on the part of the operator may be needed, such
as when:

* a service is removed, added, or replaced
* a software bug affecting the OpenStack upgrade is present in the new charm

All known issues requiring manual intervention are documented on the
:doc:`Known upgrade issues <upgrade-issues>` page. You **must** look these
over.

Verify the current deployment
-----------------------------

Confirm that the output for the :command:`juju status` command of the current
deployment is error-free. In addition, if monitoring is in use (e.g. Nagios),
ensure that all alerts have been resolved. This is to make certain that any
issues that may appear after the upgrade are not for pre-existing problems.

Perform a database backup
-------------------------

Before making any changes to cloud services perform a backup of the cloud
database by running the ``backup`` action on any single percona-cluster unit:

.. code-block:: none

   juju run-action --wait percona-cluster/0 backup

Now transfer the backup directory to the Juju client with the intention of
subsequently storing it somewhere safe. This command will grab **all** existing
backups:

.. code-block:: none

   juju scp -- -r percona-cluster/0:/opt/backups/mysql /path/to/local/directory

Permissions may first need to be altered on the remote machine.

Archive old database data
-------------------------

During the upgrade, database migrations will be run. This operation can be
optimised by first archiving any stale data (e.g. deleted instances). Do this
by running the ``archive-data`` action on any single nova-cloud-controller
unit:

.. code-block:: none

   juju run-action --wait nova-cloud-controller/0 archive-data

This action may need to be run multiple times until the action output reports
'Nothing was archived'.

Purge old compute service entries
---------------------------------

Old compute service entries for units which are no longer part of the model
should be purged before the upgrade. These entries will show as 'down' (and be
hosted on machines no longer in the model) in the current list of compute
services:

.. code-block:: none

   openstack compute service list

To remove a compute service:

.. code-block:: none

   openstack compute service delete <service-id>

Disable unattended-upgrades
---------------------------

When performing a service upgrade on a unit that hosts multiple principle
charms (e.g. ``nova-compute`` and ``ceph-osd``), ensure that
``unattended-upgrades`` is disabled on the underlying machine for the duration
of the upgrade process. This is to prevent the other services from being
upgraded outside of Juju's control. On a unit run:

.. code-block:: none

   sudo dpkg-reconfigure -plow unattended-upgrades

Upgrade order
-------------

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
| 1     | ceph-mon              | Storage       |
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
| 4     | nova-compute          | Compute       |
+-------+-----------------------+---------------+

.. important::

   OpenStack services whose software is not a part of the Ubuntu Cloud Archive
   are not represented in the above list. This type of software can only have
   their major versions changed during a series (Ubuntu) upgrade on the
   associated unit. Common charms where this applies are ``ntp``,
   ``memcached``, ``percona-cluster``, and ``rabbitmq-server``.

.. _perform_the_upgrade:

Perform the upgrade
-------------------

The essence of a charmed OpenStack service upgrade is a change of the
corresponding machine software sources so that a more recent combination of
Ubuntu release and OpenStack release is used. This combination is based on the
`Ubuntu Cloud Archive`_ and translates to a "cloud archive OpenStack release".
It takes on the following syntax:

``<ubuntu series>-<openstack-release>``

For example, the 'bionic-train' UCA release is expressed during configuration
as:

``cloud:bionic-train``

There are three methods available for performing an OpenStack service upgrade.
The appropriate method is chosen based on the actions supported by the charm.
Actions for a charm can be listed in this way:

.. code-block:: none

   juju actions <charm-name>

All-in-one
~~~~~~~~~~

The "all-in-one" method upgrades an application immediately. Although it is the
quickest route, it can be harsh when applied in the context of multi-unit
applications. This is because all the units are upgraded simultaneously, and is
likely to cause a transient service outage. This method must be used if the
application has a sole unit.

.. attention::

   The "all-in-one" method should only be used when the charm does not
   support the ``openstack-upgrade`` action.

The syntax is:

.. code-block:: none

   juju config <openstack-charm> openstack-origin=cloud:<cloud-archive-release>

Charms whose services are not technically part of the OpenStack project will
use the ``source`` charm option instead. The Ceph charms are a classic example:

.. code-block:: none

   juju config ceph-mon source=cloud:bionic-train

.. note::

   The ceph-osd and ceph-mon charms are able to maintain service availability
   during the upgrade.

So to upgrade Cinder across all units (currently running Bionic) from Stein to
Train:

.. code-block:: none

   juju config cinder openstack-origin=cloud:bionic-train

Single-unit
~~~~~~~~~~~

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

.. code-block:: none

   juju run --application <application-name> is-leader

For example, to upgrade a three-unit glance application from Stein to Train
where ``glance/1`` is the leader:

.. code-block:: none

   juju config glance action-managed-upgrade=True
   juju config glance openstack-origin=cloud:bionic-train

   juju run-action --wait glance/1 openstack-upgrade
   juju run-action --wait glance/0 openstack-upgrade
   juju run-action --wait glance/2 openstack-upgrade

.. note::

   The ``openstack-upgrade`` action is only available for charms whose services
   are part of the OpenStack project. For instance, you will need to use the
   "all-in-one" method for the Ceph charms.

.. _paused_single_unit:

Paused-single-unit
~~~~~~~~~~~~~~~~~~

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

.. code-block:: none

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

.. code-block:: none

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
-------------------------

Check for errors in :command:`juju status` output and any monitoring service.

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
