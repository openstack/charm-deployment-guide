Appendix B: OpenStack Upgrades
==============================

Overview
--------

This document outlines approaches to upgrading OpenStack using the charms.

Definitions and Terms
---------------------

Charm Upgrade
~~~~~~~~~~~~~

This is an upgrade of the charm software which is used to deploy and manage
OpenStack. This will include charms that manage applications which are not
part of the OpenStack project such as Rabbitmq and MySQL.

OpenStack Upgrade
~~~~~~~~~~~~~~~~~

This is an upgrade of the Openstack software (packages) that are installed
and managed by the charms.

Ubuntu Server Package Upgrade
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This is an upgrade of the Ubuntu packages on the server that are not part of
the OpenStack project such as kernel modules, QEMU binaries, KVM kernel module
etc.

Ubuntu Release Upgrade
~~~~~~~~~~~~~~~~~~~~~~

This is an upgrade from one Ubuntu release to the next.

Testing
-------

All procedures outlined below should be tested in a non-production environment
first.

Skipping Releases
-----------------

The charms support triggering a multi-release upgrade and will handle stepping
through the upgrade. However, given that some OpenStack projects (including
Nova) only support there being a single release difference between components
this is not recommended.

1. Charm Upgrades
-----------------

All charms should be upgraded to the latest stable release before performing
an OpenStack upgrade. It is recommended to upgrade the Keystone charm first.
The order of upgrading subsequent charms is usually not important but
check the release notes for each release to ensure there are no
special requirements.

To upgrade a charm that was deployed from the charm store:

.. code:: bash

    juju upgrade-charm <charm name>


The progress of the upgrade can be monitored by checking the workload status
of the charm which can been with **juju status**. Once the upgrade is complete
the charm status should contain the message 'Unit is ready'. The version of
the deployed software can also been seen from **juju status**:

.. code:: bash

    juju status
    ...
    App       Version  Status  Scale  Charm      Store  Rev  OS      Notes
    keystone  11.0.3   active      1  keystone   local  0  ubuntu
    ...

This shows that the deployed version of keystone is 11.0.3 (Ocata)

If the Juju controller is resource constrained it may be beneficial to do the
charm upgrades in series rather than in parallel. After each charm upgrade
check for any unforeseen errors reported in **juju status** before proceeding.

2. Pre-Upgrade Tasks
--------------------

2.1 Release Notes
~~~~~~~~~~~~~~~~~

Check the release notes for the charm releases for any special instructions.

2.2 Check current deployment
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Check for any charm errors in **juju status**. If a monitor is in use like
Nagios then make sure any alerts have been cleared before proceeding. This is
to ensure that alerts after the upgrade are not pre-existing problems.

Also ensure that the current charms must not do not contain any customisations
since that is not supported and they will be overwritten by the upgrade.

2.3 Database row archiving
~~~~~~~~~~~~~~~~~~~~~~~~~~

During the upgrade, database migrations will be run. These can be significantly
sped up by archiving any stale data (such as deleted instances). To perform the
archive of nova data run the nova-cloud-controller action:

.. code:: bash

    juju run-action nova-cloud-controller/0 archive-data

This action may need to be run multiple times until the action output reports
'Nothing was archived'


3. Upgrade Order
----------------

The charms are grouped together below. The ordering of upgrade within a group
does not matter but all the charms in each group should be upgraded before
moving on to the next group. Any release note guidance overrides the order
listed here.

+-------+-----------------------+---------------+
| Group | Charm Name            | Charm Type    |
+=======+=======================+===============+
| 1     | keystone              | Core Identity |
+-------+-----------------------+---------------+
| 2     | ceph                  | Storage       |
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
| 3     | nova-cloud-controller | Control Plane |
+-------+-----------------------+---------------+
| 3     | odl-controller        | Control Plane |
+-------+-----------------------+---------------+
| 3     | openstack-dashboard   | Control Plane |
+-------+-----------------------+---------------+
| 4     | nova-compute          | Compute       |
+-------+-----------------------+---------------+

4. Performing The Upgrade
-------------------------

If the service to be upgraded is in a highly-available cluster then the best
way to minimise service interruption is to follow the "HA with pause/resume"
instructions below. If there are multiple units of the service but they are
not clustered then follow the "Action managed" instructions.  Finally, if there
is a single unit then follow "Application one-shot".

Some parts of the upgrade, like database migrations, only need to run once per
application and these tasks are handled by the lead unit. It is advisable that
these tasks are run first (this is not applicable for one-shot deployments). To
achieve this run the upgrade on the lead unit first. To check which unit is the
lead unit either check which unit has a '*' next to it in **juju status** or
run:

.. code:: bash

    juju run --application application-name is-leader



HA with pause/resume
~~~~~~~~~~~~~~~~~~~~

The majority of charms support pause and resume actions. These actions can be
used to place units of a charm into a state where maintenance operations can
be carried out. Using these actions along with action managed upgrades allows
a charm to be removed from service, upgraded and returned to service.


For example to upgrade a three node nova-cloud-controller service from Ocata
to Pike where nova-cloud-controller/2 is the leader:

.. code:: bash

    juju config nova-cloud-controller action-managed-upgrade=True
    juju config nova-cloud-controller openstack-origin='cloud:xenial-pike'
    juju run-action nova-cloud-controller/2 --wait pause
    juju run-action nova-cloud-controller/2 --wait openstack-upgrade
    juju run-action nova-cloud-controller/2 --wait resume
    juju run-action nova-cloud-controller/1 --wait pause
    juju run-action nova-cloud-controller/1 --wait openstack-upgrade
    juju run-action nova-cloud-controller/1 --wait resume
    juju run-action nova-cloud-controller/0 --wait pause
    juju run-action nova-cloud-controller/0 --wait openstack-upgrade
    juju run-action nova-cloud-controller/0 --wait resume

Action managed
~~~~~~~~~~~~~~

If there are multiple units of an application then each unit can be upgraded
one at a time using Juju actions. This allows for rolling upgrades. To use
this feature the charm configuration option action-managed-upgrade must be set
to True.

For example to upgrade a three node keystone service from Ocata to Pike where
keystone/1 is the leader:

.. code:: bash

    juju config keystone action-managed-upgrade=True
    juju config keystone openstack-origin='cloud:xenial-pike'
    juju run-action keystone/1 --wait openstack-upgrade
    juju run-action keystone/0 --wait openstack-upgrade
    juju run-action keystone/2 --wait openstack-upgrade



Application one-shot
~~~~~~~~~~~~~~~~~~~~

This is the simplest and quickest way to perform the upgrade. Using this method
will cause all the units in the application to be upgraded at the same time.
This is likely to cause a service outage while the upgrade completes. If there
is only one unit in the application then this is the only option.

.. code:: bash

    juju config keystone openstack-origin='cloud:xenial-pike'


5. Post-Upgrade Tasks
---------------------

Check **juju status** and any monitoring solution for errors.
