=================
OpenStack upgrade
=================

This document outlines how to upgrade a Juju-deployed OpenStack cloud.

.. warning::

   Upgrading an OpenStack cloud is not risk-free. The procedures outlined in
   this guide should first be tested in a pre-production environment.

Please read the :doc:`Upgrades overview <upgrade-overview>` page before
continuing.

.. note::

   The charms only support single-step OpenStack upgrades (N+1). That is, to
   upgrade two releases forward you need to upgrade twice. You cannot skip
   releases when upgrading OpenStack with charms.

It may be worthwhile to read the upstream OpenStack `Upgrades`_ guide.

Release Notes
-------------

The OpenStack charms `Release Notes`_ for the corresponding current and target
versions of OpenStack must be consulted for any special instructions. In
particular, pay attention to services and/or configuration options that may be
retired, deprecated, or changed.

Manual intervention
-------------------

It is intended that the now upgraded charms are able to accommodate all
software changes associated with the corresponding OpenStack services to be
upgraded. A new charm will also strive to produce a service as similarly
configured to the pre-upgraded service as possible.

However, there are still times when intervention on the part of the operator
may be needed, such as when an OpenStack service is removed/added/replaced or
when a software bug (in the charms or in upstream OpenStack) affecting the
upgrade is present. The next two resources cover these topics:

* the :doc:`Special charm procedures <upgrade-special>` page
* the :doc:`Upgrade issues <upgrade-issues>` page

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
charms (e.g. nova-compute and ceph-osd), ensure that ``unattended-upgrades`` is
disabled on the underlying machine for the duration of the upgrade process.
This is to prevent the other services from being upgraded outside of Juju's
control. On a unit run:

.. code-block:: none

   sudo dpkg-reconfigure -plow unattended-upgrades

Subordinate charm applications
------------------------------

Applications that are associated with subordinate charms are upgraded along
with their parent application. Subordinate charms do not support the
``openstack-origin`` configuration option which, as will be shown, is a
pre-requisite for initiating an OpenStack charm payload upgrade.

.. _openstack_upgrade_order:

Upgrade order
-------------

Generally speaking, the order is determined by the idea of a dependency tree.
Those services that have the most potential impact on other services are
upgraded first and those services that have the least potential impact on other
services are upgraded last.

In the below table charms are listed in the order in which their corresponding
OpenStack services should be upgraded. Each service represented by a charm will
need to be upgraded individually, and only the packages associated with a
charm's OpenStack service will be updated.

The order provided below is the order used by internal testing.

.. list-table::
   :header-rows: 1
   :widths: auto

   * - Order
     - Charm

   * - 1
     - `rabbitmq-server`_

   * - 2
     - `ceph-mon`_

   * - 3
     - `keystone`_

   * - 4
     - `aodh`_

   * - 5
     - `barbican`_

   * - 6
     - `ceilometer`_

   * - 7
     - `ceph-fs`_

   * - 8
     - `ceph-radosgw`_

   * - 9
     - `cinder`_

   * - 10
     - `designate`_

   * - 11
     - `designate-bind`_

   * - 12
     - `glance`_

   * - 13
     - `gnocchi`_

   * - 14
     - `heat`_

   * - 15
     - `manila`_

   * - 16
     - `manila-generic`_

   * - 17
     - `neutron-api`_

   * - 18
     - `neutron-gateway`_

   * - 19
     - `placement`_

   * - 20
     - `nova-cloud-controller`_

   * - 21
     - `openstack-dashboard`_

   * - 22
     - `nova-compute`_

   * - 23
     - `ceph-osd`_

   * - 24
     - `swift-proxy`_

   * - 25
     - `swift-storage`_

   * - 26
     - `octavia`_

.. important::

   Services whose software is not included in the `Ubuntu Cloud Archive`_ are
   not represented in the above list. This software is upgraded by the
   administrator (on the units) using traditional means (e.g. manually via
   package tools or as part of a series upgrade). Common charms where this
   applies are ntp, memcached, percona-cluster, rabbitmq-server,
   mysql-innodb-cluster, and mysql-router.

.. note::

   An Octavia upgrade may entail an update of its load balancers (amphorae) as
   a post-upgrade task. Reasons for doing this include:

   * API incompatibility between the amphora agent and the new Octavia service
   * the desire to use features available in the new amphora agent or haproxy

   See the upstream documentation on `Rotating amphora images`_.

Software sources
----------------

The essence of an OpenStack upgrade is a change of a machine's software sources
so that a more recent combination of Ubuntu release (series) and OpenStack
release is used. This combination is based on the `Ubuntu Cloud Archive`_ and
translates to a "cloud archive OpenStack release". It takes on the following
syntax:

``<ubuntu series>-<openstack-release>``

This becomes the value given to a charm's ``openstack-origin`` configuration
option. For example, to select the 'bionic-train' release:

``openstack-origin=cloud:bionic-train``

.. important::

   The series must correspond to the series currently in use.

.. _perform_the_upgrade:

Perform the upgrade
-------------------

There are three methods available for performing an OpenStack service upgrade.
The appropriate method is chosen based on the actions supported by the charm.
Actions for a charm can be listed with command :command:`juju actions
<charm-name>`.

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

   juju run-action --wait nova-compute/0 pause
   juju run-action --wait nova-compute/0 openstack-upgrade
   juju run-action --wait nova-compute/0 resume

   juju run-action --wait nova-compute/1 pause
   juju run-action --wait nova-compute/1 openstack-upgrade
   juju run-action --wait nova-compute/1 resume

   juju run-action --wait nova-compute/2 pause
   juju run-action --wait nova-compute/2 openstack-upgrade
   juju run-action --wait nova-compute/2 resume

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

   juju run-action --wait keystone-hacluster/1 pause
   juju run-action --wait keystone/2 pause
   juju run-action --wait keystone/2 openstack-upgrade
   juju run-action --wait keystone/2 resume
   juju run-action --wait keystone-hacluster/1 resume

   juju run-action --wait keystone-hacluster/2 pause
   juju run-action --wait keystone/1 pause
   juju run-action --wait keystone/1 openstack-upgrade
   juju run-action --wait keystone/1 resume
   juju run-action --wait keystone-hacluster/2 resume

   juju run-action --wait keystone-hacluster/0 pause
   juju run-action --wait keystone/0 pause
   juju run-action --wait keystone/0 openstack-upgrade
   juju run-action --wait keystone/0 resume
   juju run-action --wait keystone-hacluster/0 resume

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
.. _Rotating amphora images: https://docs.openstack.org/octavia/latest/admin/guides/operator-maintenance.html#rotating-the-amphora-images

.. BUGS
.. _LP #1825999: https://bugs.launchpad.net/charm-nova-compute/+bug/1825999
.. _LP #1809190: https://bugs.launchpad.net/charm-neutron-gateway/+bug/1809190
.. _LP #1853173: https://bugs.launchpad.net/charm-openstack-dashboard/+bug/1853173
.. _LP #1828534: https://bugs.launchpad.net/charm-designate/+bug/1828534

.. _aodh: https://opendev.org/openstack/charm-aodh/
.. _barbican: https://opendev.org/openstack/charm-barbican/
.. _barbican-vault: https://opendev.org/openstack/charm-barbican-vault/
.. _ceilometer: https://opendev.org/openstack/charm-ceilometer/
.. _ceilometer-agent: https://opendev.org/openstack/charm-ceilometer-agent/
.. _cinder: https://opendev.org/openstack/charm-cinder/
.. _cinder-backup: https://opendev.org/openstack/charm-cinder-backup/
.. _cinder-backup-swift-proxy: https://opendev.org/openstack/charm-cinder-backup-swift-proxy/
.. _cinder-ceph: https://opendev.org/openstack/charm-cinder-ceph/
.. _designate: https://opendev.org/openstack/charm-designate/
.. _glance: https://opendev.org/openstack/charm-glance/
.. _heat: https://opendev.org/openstack/charm-heat/
.. _keystone: https://opendev.org/openstack/charm-keystone/
.. _keystone-ldap: https://opendev.org/openstack/charm-keystone-ldap/
.. _keystone-saml-mellon: https://opendev.org/openstack/charm-keystone-saml-mellon/
.. _manila: https://opendev.org/openstack/charm-manila/
.. _manila-ganesha: https://opendev.org/openstack/charm-manila-ganesha/
.. _masakari: https://opendev.org/openstack/charm-masakari/
.. _masakari-monitors: https://opendev.org/openstack/charm-masakari-monitors/
.. _mysql-innodb-cluster: https://opendev.org/openstack/charm-mysql-innodb-cluster
.. _mysql-router: https://opendev.org/openstack/charm-mysql-router
.. _neutron-api: https://opendev.org/openstack/charm-neutron-api/
.. _neutron-api-plugin-arista: https://opendev.org/openstack/charm-neutron-api-plugin-arista
.. _neutron-api-plugin-ovn: https://opendev.org/openstack/charm-neutron-api-plugin-ovn
.. _neutron-dynamic-routing: https://opendev.org/openstack/charm-neutron-dynamic-routing/
.. _neutron-gateway: https://opendev.org/openstack/charm-neutron-gateway/
.. _neutron-openvswitch: https://opendev.org/openstack/charm-neutron-openvswitch/
.. _nova-cell-controller: https://opendev.org/openstack/charm-nova-cell-controller/
.. _nova-cloud-controller: https://opendev.org/openstack/charm-nova-cloud-controller/
.. _nova-compute: https://opendev.org/openstack/charm-nova-compute/
.. _octavia: https://opendev.org/openstack/charm-octavia/
.. _octavia-dashboard: https://opendev.org/openstack/charm-octavia-dashboard/
.. _octavia-diskimage-retrofit: https://opendev.org/openstack/charm-octavia-diskimage-retrofit/
.. _openstack-dashboard: https://opendev.org/openstack/charm-openstack-dashboard/
.. _placement: https://opendev.org/openstack/charm-placement
.. _swift-proxy: https://opendev.org/openstack/charm-swift-proxy/
.. _swift-storage: https://opendev.org/openstack/charm-swift-storage/

.. _ceph-fs: https://opendev.org/openstack/charm-ceph-fs/
.. _ceph-iscsi: https://opendev.org/openstack/charm-ceph-iscsi/
.. _ceph-mon: https://opendev.org/openstack/charm-ceph-mon/
.. _ceph-osd: https://opendev.org/openstack/charm-ceph-osd/
.. _ceph-proxy: https://opendev.org/openstack/charm-ceph-proxy/
.. _ceph-radosgw: https://opendev.org/openstack/charm-ceph-radosgw/
.. _ceph-rbd-mirror: https://opendev.org/openstack/charm-ceph-rbd-mirror/
.. _cinder-purestorage: https://opendev.org/openstack/charm-cinder-purestorage/
.. _designate-bind: https://opendev.org/openstack/charm-designate-bind/
.. _glance-simplestreams-sync: https://opendev.org/openstack/charm-glance-simplestreams-sync/
.. _gnocchi: https://opendev.org/openstack/charm-gnocchi/
.. _hacluster: https://opendev.org/openstack/charm-hacluster/
.. _ovn-central: https://opendev.org/x/charm-ovn-central
.. _ovn-chassis: https://opendev.org/x/charm-ovn-chassis
.. _ovn-dedicated-chassis: https://opendev.org/x/charm-ovn-dedicated-chassis
.. _pacemaker-remote: https://opendev.org/openstack/charm-pacemaker-remote/
.. _percona-cluster: https://opendev.org/openstack/charm-percona-cluster/
.. _rabbitmq-server: https://opendev.org/openstack/charm-rabbitmq-server/
.. _trilio-data-mover: https://opendev.org/openstack/charm-trilio-data-mover/
.. _trilio-dm-api: https://opendev.org/openstack/charm-trilio-dm-api/
.. _trilio-horizon-plugin: https://opendev.org/openstack/charm-trilio-horizon-plugin/
.. _trilio-wlm: https://opendev.org/openstack/charm-trilio-wlm/
.. _vault: https://opendev.org/openstack/charm-vault/

.. _manila-generic: https://opendev.org/openstack/charm-manila-generic/
