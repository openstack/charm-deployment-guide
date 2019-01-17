Appendix F: Series Upgrade
==============================

Introduction
++++++++++++

Juju and OpenStack charms provide the primitives to prepare for and
respond to an upgrade from one Ubuntu LTS series to another.


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

Charms are upgraded to the latest release.

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
