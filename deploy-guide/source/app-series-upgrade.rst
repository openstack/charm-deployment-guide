Appendix F: Series Upgrade
==============================

Introduction
++++++++++++

Juju and OpenStack charms provide the primitives to prepare for and
respond to an upgrade from one Ubuntu LTS series to another.

.. note::

   The recommended best practice is that the Juju machines that comprise the
   cloud should eventually all be running the same series (e.g. 'xenial' or
   'bionic', but not a mix of the two).

Warnings
++++++++

Upgrading a single machine from one LTS to another is a complex task.
Doing so on a running OpenStack cloud is an order of magnitude more
complex.

Please read through this document thoroughly before attempting a series
upgrade. Please pay particular attention to the Assumptions section and
the order of operations.

The series upgrade should be executed by an administrator or team of
administrators who are intimately familiar with the cloud undergoing
upgrade, OpenStack in general, working with Juju and OpenStack charms.

The tasks of preparing stateful OpenStack services for series upgrade is
not automated and is the responsibility of the administrator. For
example: evacuating a compute node, switching HA routers to a network
node, any storage rebalancing that may be required.

The actual task of executing the do-release-upgrade on an individual
machine is not automated. It will be performed by the administrator. Any
bespoke preparation for or cleanup after the do-release-upgrade is the
responsibility of the administrator.

The series upgrade process requires API downtime. Although the goal is
minimal downtime, it is necessary to pause services to avoid race
condition errors. Therefore, the API undergoing upgrade will require
downtime.

Stateful services which OpenStack depends on such as percona-cluster and
rabbitmq will affect all APIs during series upgrade and therefore
require downtime.

Third party charms may not have implemented series upgrade yet. Please
pay particular attention to SDN and storage charms which may affect
cloud operation.

If the architecture and layout of charms does not match the assumptions
section of this document, great care needs to be taken to avoid problems
with application leadership across machines. In other words, if most
services are not in LXD containers, it is possible to have the leader of
percona-cluster on one host and the leader of rabbit on another causing
complication's in the procure for series upgrade.

Test, test, test! The series upgrade process should be tested on a
non-production cloud that closely resembles the eventual production
environment. Not only does this validate the software involved but it
prepares the administrator for the complex task ahead.


Juju
++++

Please read all Juju documentation on the series upgrade feature.

https://docs.jujucharms.com/devel/en/getting-started

.. note::
    The Juju upgrade-series command operates on the machine level. This
    document will be focused on applications as many require pausing their
    peers and some subordinates. But it is important to remember the whole
    machine is upgraded.

    Applications deployed in a LXD container are considered a machine apart
    from the physical host machine the container is hosted on.

    Upgrading the host machine will not upgrade the LXD contained machines.
    However, when the required post-upgrade reboot of the host machine
    occurs all the services contained in LXD containers will be unavailable
    during the reboot.

    For example a physical host with nova-compute, neutron-openvswitch and
    ceph-osd colocated as well as hosting a keystone unit in a LXD. When
    the juju upgrade-series prepare command is executed on the machine,
    nova-compute, neutron-openvswitch and ceph-osd will execute their
    pre-series-upgrade hooks but keystone will not. Nor will the LXD
    operating system be affected by the do-release-upgrade on the host. At
    reboot however, the keystone unit will be unavailable during the
    duration of the reboot. Please plan accordingly.


Assumptions
+++++++++++

This document makes a number of assumptions about the architecture and
preparation of the cloud undergoing series upgrade. Please review these
and compare to the running cloud before performing the series upgrade.

Preparations
~~~~~~~~~~~~

The entire suite of charms used to manage the cloud should be upgraded to the
latest stable charm revision before any major change is made to the cloud such
as the current machine series upgrades. See `Charm upgrades`_ for guidance.

OpenStack is upgraded to the highest version the current LTS supports.
Mitaka for Trusty and Queens for Xenial.

The current Ubuntu operating system is up to date prior to do-release-upgrade.

Stateful services have been backed up. Percona-cluster and mongodb
should be backed up prior to upgrading.

General cloud health. Confirm the cloud is fully operational before
beginning a series upgrade.

OpenStack charms health. No charms are in hook error. Confirm the health
of the juju environment before beginning series upgrade.

Per machine preparations. Individual compute nodes are evacuated prior
to series upgrade. HA routers are moved to network nodes not undergoing
series upgrade.

`Automatic Updates aka. Unattended Upgrades <https://help.ubuntu.com/lts/serverguide/automatic-updates.html.en>`_
is enabled by default on Ubuntu Server and must be disabled on all machines
prior to initiating the upgrade procedure.  This is imperative to stay in
control of when and where updates are applied throughout the upgrade procedure.


Hyper-Converged Architecture
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Compute, storage and their subordinates may be colocated.

API Services are deployed in LXD containers.

Percona-cluster is deployed in a LXD container.

Rabbitmq is deployed in a LXD container.

Third party charms either do not exist or have been thoroughly tested
for series upgrade.

No other non-subordinate charms are colocated on the same machine.


Overview
++++++++

.. note::
    This overview is not a substitute for understanding the
    entirety of this document. It is the general case but the individual
    details matter. Read "where appropriate" at the end of each step.

Evacuate or otherwise prepare the machine

Pause hacluster for non-leader units not undergoing upgrade

Pause non-leader peer units not undergoing upgrade

Juju upgrade-series prepare the leader's machine

Execute do-release-upgrade and any post-upgrade operating system tasks

Reboot

Set openstack-origin or source for new operating system ("distro")

Juju upgrade-series complete the machine

Repeat the steps from prepare to complete for the non-leader machines

Perform any cluster completed upgrade tasks after all units of
application have been upgraded.

Juju set-series to the new series for all future units of an application.

Exceptions
~~~~~~~~~~

This overview describes the general case that includes the API charms,
percona culster and rabbitmq.

The notable exceptions are nova-compute, ceph-mon and ceph-osd which
do not require pausing of any units and unit leadership is irrelevant.


Example as code
~~~~~~~~~~~~~~~

Attempting an automated series upgrade on a running production cloud is
not recommended. The following example-as-code encapsulates the
processes described in this document, and are provided solely to
illustrate the methods used to develop and test the series upgrade
primitives. The example code should not be consumed in an automation
outside of its intended use case (charm dev/test gate automation).

https://github.com/openstack-charmers/zaza/blob/master/zaza/charm_tests/series_upgrade/tests.py

https://github.com/openstack-charmers/zaza/blob/master/zaza/utilities/generic.py#L173


Procedures
++++++++++

The following procures are broken up into categories of charms that
follow the same procedure.

.. note::
    Example commands used in this documentation assume a Trusty to Xenial
    series upgrade, the same approach is used for Xenial to Bionic
    series upgrades. Unit and machine numbers are examples only they will
    differ from site to site. For example the machine number 0 is reused
    purely for example purposes.


Physical Host Nodes
~~~~~~~~~~~~~~~~~~~

Procedure for the physical host nodes which may include nova-compute,
neutron-openvswitch and ceph-osd as well as neutron-gateway. Though
ceph-mon is most often deployed in LXD containers it follows this
procedure.

 .. note::
    Nova-compute and ceph-osd are  used in the commands below for
    example purposes. In this example, physical host where
    nova-compute/0 and ceph-osd/0 are deployed is machine 0.

Evacuate or otherwise prepare the machine
 For compute nodes move all running VMs off the physical host.
 For network nodes force HA routers off of the current node.
 Any storage related tasks that may be required.
 Any site specific tasks that may be required.


Juju upgrade-series prepare the machine
 .. code:: bash

    juju upgrade-series 0 prepare xenial

 .. note::
    The upgrade-series prepare command causes all the charms on the given
    machine to run their pre-series-upgrade hook. For most cases with the
    OpenStack charms this pauses the unit. At the completion of the
    pre-series-upgrade hook the workload status should be "blocked" with
    the message "Ready for do-release-upgrade and reboot."

Execute do-release-upgrade and any post-upgrade operating system tasks
 The do-release-upgrade process is performed by the administrator. Any
 post do-release-upgrade tasks are also the responsibility of the
 administrator.

Reboot
 Post do-release-upgrade reboot executed by the administrator.

Set openstack-origin or source for new operating system ("distro")
 This step is required and should occur before the first node is
 completed.

 .. code:: bash

    juju config nova-compute openstack-origin=distro
    juju config ceph-osd source=distro


Juju upgrade-series complete the machine
 .. code:: bash

    juju upgrade-series 0 complete

 .. note::

    The upgrade-series complete command causes all the charms on the given
    machine to run their post-series-upgrade hook. For most cases with the
    OpenStack charms this re-writes configuration files and resumes the unit.
    At the completion of the post-series-upgrade hook the workload status
    should be "active" with the message "Unit is ready."

Juju set-series to the new series for all future units of an application.
 To guarantee that any future unit-add commands create new
 instantiations of the application on the correct series it is necessary
 to set the series on the application.

 .. code:: bash

    juju set-series nova-compute xenial
    juju set-series neutron-openvswitch xenial
    juju set-series ceph-osd xenial


Repeat the procedure for all physical host nodes.
 It is not necessary to repeat the set openstack-origin step.



Stateful Services
~~~~~~~~~~~~~~~~~

Procedure for the stateful services deployed on LXD containers.
These include percona-cluster and rabbitmq.

.. warning::

    For Bionic to Focal series upgrades see percona-cluster migration to
    mysql-innodb-cluster and mysql-router under Series Specific Procedures.


.. note::
    While percona-cluster is often deployed with hacluster for HA,
    rabbitmq is not. Ignore the hacluster steps for rabbitmq.
    Likewise no backup is required of rabbitmq. Percona-cluster is used
    below for example purposes. In this example, the LXD container the
    leader node of percona-cluster/0 is deployed on is machine 0.


Prepare the machine
 Perform backups of percona-cluster and scp the backup to a secure
 location.

 .. code:: bash

    juju run-action percona-cluster/0 backup
    juju scp -- -r percona-cluster/0:/opt/backups/mysql /path/to/local/backup/dir


Pause hacluster for non-leader units not undergoing upgrade
 .. code:: bash

    juju run-action percona-cluster-hacluster/1 pause
    juju run-action percona-cluster-hacluster/2 pause


Pause non-leader peer units not undergoing upgrade
 .. code:: bash

    juju run-action percona-cluster/1 pause
    juju run-action percona-cluster/2 pause


Juju upgrade-series prepare the leader's machine
 .. code:: bash

    juju upgrade-series 0 prepare xenial

 .. note::
    The upgrade-series prepare command causes all the charms on the given
    machine to run their pre-series-upgrade hook. For most cases with the
    OpenStack charms this pauses the unit. At the completion of the
    pre-series-upgrade hook the workload status should be "blocked" with
    the message "Ready for do-release-upgrade and reboot."

Execute do-release-upgrade and any post-upgrade operating system tasks
 The do-release-upgrade process is performed by the administrator. Any
 post do-release-upgrade tasks are also the responsibility of the
 administrator.

Reboot
 Post do-release-upgrade reboot executed by the administrator.

Set openstack-origin or source for new operating system ("distro")
 This step is required and should occur before the first node is
 completed but after the other units are paused.

 .. code:: bash

    juju config percona-cluster source=distro


Juju upgrade-series complete the machine
 .. code:: bash

    juju upgrade-series 0 complete

 .. note::

    The upgrade-series complete command causes all the charms on the given
    machine to run their post-series-upgrade hook. For most cases with the
    OpenStack charms this re-writes configuration files and resumes the unit.
    At the completion of the post-series-upgrade hook the workload status
    should be "active" with the message "Unit is ready."

Repeat the procedure for non-leader nodes
 It is not necessary to repeat the set openstack-origin step.

Perform any cluster completed upgrade tasks after all units of application have been upgraded.
 Run the complete-cluster-series-upgrade action on the leader node. This
 action informs each node of the cluster the upgrade process is complete
 cluster wide. This also updates mysql configuration with all peers in
 the cluster.

 .. code:: bash

    juju run-action percona-cluster/0 complete-cluster-series-upgrade

Juju set-series to the new series for all future units of an application.
 To guarantee that any future unit-add commands create new
 instantiations of the application on the correct series it is necessary
 to set the series on the application.

 .. code:: bash

    juju set-series percona-cluster xenial


API Services
~~~~~~~~~~~~

Procedure for the API services in LXD containers. These include but are
not limited to keystone, glance, cinder, neutron-api and
nova-cloud-controller. Any subordinates deployed with these applications
will be upgraded at the same time.

.. note::
    Keystone is used in the commands below for example purposes. In this
    example, the LXD container the leader node of keystone/0 is deployed
    on is machine 0.


Pause hacluster for non-leader units not undergoing upgrade
 .. code:: bash

    juju run-action keystone-hacluster/1 pause
    juju run-action keystone-hacluster/2 pause


Pause non-leader peer units not undergoing upgrade
 .. code:: bash

    juju run-action keystone/1 pause
    juju run-action keystone/2 pause


Juju upgrade-series prepare the leader's machine
 .. code:: bash

    juju upgrade-series 0 prepare xenial

 .. note::
    The upgrade-series prepare command causes all the charms on the given
    machine to run their pre-series-upgrade hook. For most cases with the
    OpenStack charms this pauses the unit. At the completion of the
    pre-series-upgrade hook the workload status should be "blocked" with
    the message "Ready for do-release-upgrade and reboot."

Execute do-release-upgrade and any post-upgrade operating system tasks
 The do-release-upgrade process is performed by the administrator. Any
 post do-release-upgrade tasks are also the responsibility of the
 administrator.

Reboot
 Post do-release-upgrade reboot executed by the administrator.

Set openstack-origin or source for new operating system ("distro")
 This step is required and should occur before the first node is
 completed but after the other units are paused.

 .. code:: bash

    juju config keystone source=distro


Juju upgrade-series complete the machine
 .. code:: bash

    juju upgrade-series 0 complete

 .. note::

    The upgrade-series complete command causes all the charms on the given
    machine to run their post-series-upgrade hook. For most cases with the
    OpenStack charms this re-writes configuration files and resumes the unit.
    At the completion of the post-series-upgrade hook the workload status
    should be "active" with the message "Unit is ready."

Repeat the procedure for non-leader nodes
 It is not necessary to repeat the set openstack-origin step.

Juju set-series to the new series for all future units of an application.
 To guarantee that any future unit-add commands create new
 instantiations of the application on the correct series it
 is necessary to set the series on the application.

 .. code:: bash

    juju set-series keystone xenial

.. raw:: html

   <!-- LINKS -->

.. _Charm upgrades: app-upgrade-openstack#charm-upgrades


Series Specific Procedures
++++++++++++++++++++++++++

Bionic to Focal
~~~~~~~~~~~~~~~

percona-cluster migration to mysql-innodb-cluster and mysql-router
__________________________________________________________________


In Ubuntu 20.04 LTS (Focal) the percona-xtradb-cluster-server package will no
longer be available. It has been replaced by mysql-server-8.0 and mysql-router
in Ubuntu main. Therefore, there is no way to series upgrade percona-cluster to
Focal. Instead the databases hosted by percona-cluster will need to be migrated
to mysql-innodb-cluster and mysql-router will need to be deployed as a
subordinate on the applications that use MySQL as a data store.

.. warning::

   Since the DB affects most OpenStack services it is important to have a
   sufficient downtime window. The following procedure is written in an attempt
   to migrate one service at a time (i.e. keystone, glance, cinder, etc).
   However, it may be more practical to migrate all databases at the same time
   during an extended downtime window, as there may be unexpected
   interdependencies between services.

.. note::

   It is possible for percona-cluster to remain on Ubuntu 18.04 LTS while
   the rest of the cloud migrates to Ubuntu 20.04 LTS. In fact, this state
   will be one step of the migration process.


Procedure

* Leave all the percona-cluster machines on Bionic and upgrade the series of
  the remaining machines in the cloud per this document.

* Deploy a mysql-innodb-cluster on Focal.

 .. code-block:: none

    juju deploy -n 3 mysql-innodb-cluster --series focal

* Deploy (but do not yet relate) an instance of mysql-router for every
  application that requires a data store (i.e. every application that was
  related to percona-cluster).

 .. code-block:: none

    juju deploy mysql-router cinder-mysql-router
    juju deploy mysql-router glance-mysql-router
    juju deploy mysql-router keystone-mysql-router
    ...

* Add relations between the mysql-router instances and the
  mysql-innodb-cluster.

 .. code-block:: none

    juju add-relation cinder-mysql-router:db-router mysql-innodb-cluster:db-router
    juju add-relation glance-mysql-router:db-router mysql-innodb-cluster:db-router
    juju add-relation keystone-mysql-router:db-router mysql-innodb-cluster:db-router
    ...

On a per-application basis:

* Remove the relation between the application charm and the percona-cluster
  charm. You can view existing relations with the :command:`juju status
  percona-cluster --relations` command.

 .. code-block:: none

    juju remove-relation keystone:shared-db percona-cluster:shared-db

* Dump the existing database(s) from percona-cluster.

 .. note::

    In the following, the percona-cluster/0 and mysql-innodb-cluster/0 units
    are used as examples. For percona, any unit of the application may be used,
    though all the steps should use the same unit. For mysql-innodb-cluster,
    the RW unit should be used. The RW unit of the mysql-innodb-cluster can be
    determined from the :command:`juju status mysql-innodb-cluster` command.

 * Allow Percona to dump databases. See `Percona strict mode`_ to understand
   the implications of this setting.

 .. code-block:: none

    juju run-action --wait percona-cluster/0 set-pxc-strict-mode mode=MASTER

 * Dump the specific application's database(s).

  .. note::

     Depending on downtime restrictions it is possible to dump all databases at
     one time: run the ``mysqldump`` action without setting the ``databases``
     parameter.  Similarly, it is possible to import all the databases into
     mysql-innodb-clulster from that single dump file.

  .. note::

     The database name may or may not match the application name. For example,
     while keystone has a DB named keystone, openstack-dashboard has a database
     named horizon. Some applications have multiple databases. Notably,
     nova-cloud-controller which has at least: nova,nova_api,nova_cell0 and a
     nova_cellN for each additional cell. See upstream documentation for the
     respective application to determine the database name.

 .. code-block:: none

    # Single DB
    juju run-action --wait percona-cluster/0 mysqldump databases=keystone

    # Multiple DBs
    juju run-action --wait percona-cluster/0 mysqldump databases=nova,nova_api,nova_cell0

 * Return Percona enforcing strict mode. See `Percona strict mode`_ to
   understand the implications of this setting.

 .. code-block:: none

    juju run-action --wait percona-cluster/0 set-pxc-strict-mode mode=ENFORCING

* Transfer the mysqldump file from the percona-cluster unit to the
  mysql-innodb-cluster RW unit. The RW unit of the mysql-innodb-cluster can be
  determined from juju status: `juju status mysql-innodb-cluster`. Bellow we
  use mysql-innodb-cluster/0 as an example.

 .. code-block:: none

    juju scp percona-cluster/0:/var/backups/mysql/mysqldump-keystone-<DATE>.gz .
    juju scp mysqldump-keystone-<DATE>.gz mysql-innodb-cluster/0:/home/ubuntu

* Import the database(s) into mysql-innodb-cluster.

 .. code-block:: none

    juju run-action --wait mysql-innodb-cluster/0 restore-mysqldump dump-file=/home/ubuntu/mysqldump-keystone-<DATE>.gz

* Relate an instance of mysql-router for every application that requires a data
  store (i.e. every application that needed percona-cluster):

 .. code-block:: none

    juju add-relation keystone:shared-db keystone-mysql-router:shared-db

* Repeat for remaining applications.

An overview of this process can be seen in the OpenStack charmer's team CI `Zaza migration code`_.

Post-migration

As noted above it is possible to run the cloud with percona-cluster remaining
on Bionic indefinitely. Once all databases have been migrated to
mysql-innodb-cluster, all the databases have been backed up, and the cloud has
been verified to be in good working order the percona-cluster application (and
its probable hacluster subordinates) may be removed.

 .. code-block:: none

    juju remove-application percona-cluster-hacluster
    juju remove-application percona-cluster


.. _Zaza migration code: https://github.com/openstack-charmers/zaza-openstack-tests/blob/master/zaza/openstack/charm_tests/mysql/tests.py#L556
.. _Percona strict mode: https://www.percona.com/doc/percona-xtradb-cluster/LATEST/features/pxc-strict-mode.html
