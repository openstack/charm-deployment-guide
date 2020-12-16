:orphan:

========================
Series upgrade OpenStack
========================

This document will provide specific steps for how to perform a series upgrade
across the entirety of a Charmed OpenStack cloud.

.. warning::

   This document is based upon the foundational knowledge and guidelines set
   forth in the more general `Series upgrade`_ appendix. That reference must be
   studied in-depth prior to attempting the steps outlined here. In particular,
   ensure that the :ref:`Pre-upgrade requirements <pre-upgrade_requirements>`
   are satisfied; the :doc:`Upgrade issues <upgrade-issues>` have been reviewed
   and considered; and that :ref:`Workload specific preparations
   <workload_specific_preparations>` have been addressed during planning.

Downtime
--------

Although the goal is to minimise downtime the series upgrade process across a
cloud will nonetheless result in some level of downtime for the control plane.

When the machines associated with stateful applications such as percona-cluster
and rabbitmq-server undergo a series upgrade all cloud APIs will experience
downtime, in addition to the stateful application itself.

When machines associated with a single API application undergo a series upgrade
that individual API will also experience downtime. This is because it is
necessary to pause services in order to avoid race condition errors.

For those applications working in tandem with hacluster, as will be shown, some
hacluster units will need to be paused before the upgrade. One should assume
that the commencement of an outage coincides with this step (it will cause
cluster quorum heartbeats to fail and the service VIP will consequently go
offline).

Reference cloud topology
------------------------

This section describes a hyperconverged cloud topology that this document will
use for the procedural steps to follow. Hyperconvergence refers to the practice
of co-locating principal applications on the same machine.

The topology is defined in this way:

* Only compute and storage charms (and their subordinates) may be co-located.

* Third-party charms either do not exist or have been thoroughly tested
  for a series upgrade.

* The following are containerised:

  * All API applications

  * The percona-cluster application

  * The rabbitmq-server application

  * The ceph-mon application

Storage charms are charms that manage physical disks. For example, ceph-osd and
swift-storage. Example OpenStack subordinate charms are networking SDN charms
for the nova-compute charm, or monitoring charms for compute or storage charms.

.. caution::

   If your cloud differs from this topology you must adapt the procedural steps
   accordingly. In particular, look at the aspects of co-located applications
   and containerised applications. Recall that:

   * the :command:`upgrade-series` command:

     * affects all applications residing on the target machine

     * does not affect containers hosted on the target machine

   * an application's leader should be upgraded before its non-leaders

Generalised OpenStack series upgrade
------------------------------------

This section will summarise the series upgrade steps in the context of specific
OpenStack applications. It is an enhancement of the :ref:`Generic series
upgrade <generic_series_upgrade>` section in the companion document.

Applications for which this summary does **not** apply include:

* nova-compute
* ceph-mon
* ceph-osd

This is because the above applications do not require the pausing of units and
application leadership is irrelevant for them.

However, this summary does apply to all API applications (e.g. neutron-api,
keystone, nova-cloud-controller), as well as percona-cluster, and
rabbitmq-server.

.. important::

   The first machine to be upgraded is always associated with the leader of the
   principal application. Let this machine be called the "principal leader
   machine" and its unit be called the "principal leader unit".

The steps are as follows:

#. Set the default series for the principal application and ensure the same has
   been done to the model.

#. If hacluster is used, pause the hacluster units not associated with the
   principal leader machine.

#. Pause the principal non-leader units.

#. Perform a series upgrade on the principal leader machine.

   #. Perform any pre-upgrade workload maintenance tasks.

   #. Invoke the :command:`prepare` sub-command.

   #. Upgrade the operating system (APT commands).

   #. Perform any post-upgrade workload maintenance tasks.

   #. Reboot.

#. Set the value of the (application-dependent) ``openstack-origin`` or the
   ``source`` configuration option to 'distro' (new operating system).

#. Invoke the :command:`complete` sub-command on the principal leader machine.

#. Repeat steps 4 and 6 for the application non-leader machines.

#. Perform any possible cluster completed upgrade tasks once all machines have
   had their series upgraded.

   .. note::

      Here is a non-extensive list of the most common post-upgrade tasks for
      OpenStack and supporting charms:

      * percona-cluster: run action ``complete-cluster-series-upgrade`` on the
        leader unit.
      * rabbitmq-server: run action ``complete-cluster-series-upgrade`` on the
        leader unit.
      * ceilometer: run action ``ceilometer-upgrade`` on the leader unit.
      * vault: Each vault unit will need to be unsealed after its machine is
        rebooted.

Procedures
----------

The procedures are categorised based on application types. The example scenario
used throughout is a 'xenial' to 'bionic' series upgrade, within an OpenStack
release of Queens (i.e. the starting point is a cloud archive pocket of
'xenial-queens').

Stateful applications
~~~~~~~~~~~~~~~~~~~~~

This section covers the series upgrade procedure for containerised stateful
applications. These include:

* ceph-mon
* percona-cluster
* rabbitmq-server

A stateful application is one that maintains the state of various aspects of
the cloud. Clustered stateful applications, such as all the ones given above,
also require a quorum to function properly. Because of these reasons a stateful
application should not have all of its units restarted simultaneously; it must
have the series of its corresponding machines upgraded sequentially.

.. note::

   The concurrent upgrade approach is theoretically possible, although to use
   it all cloud workloads will need to be stopped in order to ensure
   consistency. This is not recommended.

The example procedure will be based on the percona-cluster application.

.. warning::

   The eoan series is the last series supported by the percona-cluster charm.
   It is replaced by the `mysql-innodb-cluster`_ and `mysql-router`_ charms in the
   focal series. The migration steps are documented in `percona-cluster charm
   - series upgrade to focal`_.

   Do not upgrade the machines hosting percona-cluster units to the focal
   series. To be clear, if percona-cluster is containerised then it is the LXD
   container that must not be upgraded.

.. important::

   Unlike percona-cluster, the ceph-mon and rabbitmq-server applications do not
   use hacluster to achieve HA, nor do they need backups. Disregard therefore
   the hacluster and backup steps for these two applications.

   The ceph-mon charm will maintain the MON cluster during a series upgrade, so
   ceph-mon units do not need to be paused.

This scenario is represented by the following partial :command:`juju status`
command output:

.. code-block:: console

   Model    Controller       Cloud/Region    Version  SLA          Timestamp
   upgrade  maas-controller  mymaas/default  2.7.6    unsupported  18:26:57Z

   App                        Version  Status  Scale  Charm            Store       Rev  OS      Notes
   percona-cluster            5.6.37   active      3  percona-cluster  jujucharms  286  ubuntu
   percona-cluster-hacluster           active      3  hacluster        jujucharms   66  ubuntu

   Unit                            Workload  Agent  Machine  Public address  Ports     Message
   percona-cluster/0               active    idle   0/lxd/0  10.0.0.47       3306/tcp  Unit is ready
     percona-cluster-hacluster/0*  active    idle            10.0.0.47                 Unit is ready and clustered
   percona-cluster/1*              active    idle   1/lxd/0  10.0.0.48       3306/tcp  Unit is ready
     percona-cluster-hacluster/2   active    idle            10.0.0.48                 Unit is ready and clustered
   percona-cluster/2               active    idle   2/lxd/0  10.0.0.49       3306/tcp  Unit is ready
     percona-cluster-hacluster/1   active    idle            10.0.0.49                 Unit is ready and clustered

In summary, the principal leader unit is percona-cluster/1 and is deployed on
machine 1/lxd/0 (the principal leader machine).

.. warning::

   During this upgrade, there will be a MySQL service outage. The HA resources
   provided by hacluster will **not** be monitored during the series upgrade
   due to the pausing of units.

#. Perform any workload maintenance pre-upgrade steps. For percona-cluster,
   take a backup and transfer it to a secure location:

   .. code-block:: none

      juju run-action --wait percona-cluster/1 backup
      juju scp -- -r percona-cluster/1:/opt/backups/mysql /path/to/local/directory

   Permissions will need to be altered on the remote machine, and note that the
   last command transfers **all** existing backups.

.. note::

   These upstream resources may also be useful:

   * `Upgrading Percona XtraDB Cluster`_
   * `Percona XtraDB Cluster In-Place Upgrading Guide From 5.5 to 5.6`_
   * `Galera replication - how to recover a PXC cluster`_

#. Set the default series for both the model and the principal application:

   .. code-block:: none

      juju model-config default-series=bionic
      juju set-series percona-cluster bionic

#. Pause the hacluster units not associated with the principal leader machine:

   .. code-block:: none

      juju run-action --wait percona-cluster-hacluster/0 pause
      juju run-action --wait percona-cluster-hacluster/1 pause

#. Pause the principal non-leader units:

   .. code-block:: none

      juju run-action --wait percona-cluster/0 pause
      juju run-action --wait percona-cluster/2 pause

   For percona-cluster, leaving the principal leader unit up will ensure it
   has the latest MySQL sequence number; it will be considered the most up to
   date cluster member.

#. Perform a series upgrade on the principal leader machine:

   .. code-block:: none

      juju upgrade-series 1/lxd/0 prepare bionic
      juju run --machine=1/lxd/0 -- sudo apt update
      juju ssh 1/lxd/0 sudo apt full-upgrade
      juju ssh 1/lxd/0 sudo do-release-upgrade

   For percona-cluster, there are no post-upgrade steps; the prompt to reboot
   can be answered in the affirmative.

#. Set the value of the ``source`` configuration option to 'distro':

   .. code-block:: none

      juju config percona-cluster source=distro

#. Invoke the :command:`complete` sub-command on the principal leader machine:

   .. code-block:: none

      juju upgrade-series 1/lxd/0 complete

   At this point the :command:`juju status` output looks like this:

   .. code-block:: console

      Model    Controller       Cloud/Region    Version  SLA          Timestamp
      upgrade  maas-controller  mymaas/default  2.7.6    unsupported  19:51:52Z

      App                        Version  Status       Scale  Charm            Store       Rev  OS      Notes
      percona-cluster            5.7.20   maintenance      3  percona-cluster  jujucharms  286  ubuntu
      percona-cluster-hacluster           blocked          3  hacluster        jujucharms   66  ubuntu

      Unit                            Workload     Agent  Machine  Public address  Ports     Message
      percona-cluster/0               maintenance  idle   0/lxd/0  10.0.0.47       3306/tcp  Paused. Use 'resume' action to resume normal service.
        percona-cluster-hacluster/0*  maintenance  idle            10.0.0.47                 Paused. Use 'resume' action to resume normal service.
      percona-cluster/1*              active       idle   1/lxd/0  10.0.0.48       3306/tcp  Unit is ready
        percona-cluster-hacluster/2   blocked      idle            10.0.0.48                 Resource: res_mysql_11810cc_vip not running
      percona-cluster/2               maintenance  idle   2/lxd/0  10.0.0.49       3306/tcp  Paused. Use 'resume' action to resume normal service.
        percona-cluster-hacluster/1   maintenance  idle            10.0.0.49                 Paused. Use 'resume' action to resume normal service.

      Machine  State    DNS        Inst id              Series  AZ     Message
      0        started  10.0.0.44  node1                xenial  zone1  Deployed
      0/lxd/0  started  10.0.0.47  juju-f83fcd-0-lxd-0  xenial  zone1  Container started
      1        started  10.0.0.45  node2                xenial  zone2  Deployed
      1/lxd/0  started  10.0.0.48  juju-f83fcd-1-lxd-0  bionic  zone2  Running
      2        started  10.0.0.46  node3                xenial  zone3  Deployed
      2/lxd/0  started  10.0.0.49  juju-f83fcd-2-lxd-0  xenial  zone3  Container started

#. For percona-cluster, a sanity check should be done on the leader unit's
   databases and data.

#. Repeat steps 5 and 7 for the principal non-leader machines.

#. Perform any possible cluster completed upgrade tasks once all machines have
   had their series upgraded:

   .. code-block:: none

      juju run-action --wait percona-cluster/leader complete-cluster-series-upgrade

   For percona-cluster (and rabbitmq-server), the above action is performed on
   the leader unit. It informs each cluster node that the upgrade process is
   complete cluster-wide. This also updates MySQL configuration with all peers
   in the cluster.

API applications
~~~~~~~~~~~~~~~~

This section covers series upgrade procedures for containerised API
applications. These include, but are not limited to:

* cinder
* glance
* keystone
* neutron-api
* nova-cloud-controller

Machines hosting API applications can have their series upgraded concurrently
because those applications are stateless. This results in a dramatically
reduced downtime for the application. A sequential approach will not reduce
downtime as the HA services will still need to be brought down during the
upgrade associated with the application leader.

The following two sub-sections will show how to perform a series upgrade
concurrently for a single API application and for multiple API applications.

Upgrading a single API application concurrently
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This example procedure will be based on the keystone application.

This scenario is represented by the following partial :command:`juju status`
command output:

.. code-block:: console

   Model    Controller       Cloud/Region    Version  SLA          Timestamp
   upgrade  maas-controller  mymaas/default  2.7.6    unsupported  22:48:41Z

   App                 Version  Status  Scale  Charm            Store       Rev  OS      Notes
   keystone            13.0.2   active      3  keystone         jujucharms  312  ubuntu
   keystone-hacluster           active      3  hacluster        jujucharms   66  ubuntu

   Unit                     Workload  Agent  Machine  Public address  Ports     Message
   keystone/0*              active    idle   0/lxd/0  10.0.0.70       5000/tcp  Unit is ready
     keystone-hacluster/0*  active    idle            10.0.0.70                 Unit is ready and clustered
   keystone/1               active    idle   1/lxd/0  10.0.0.71       5000/tcp  Unit is ready
     keystone-hacluster/2   active    idle            10.0.0.71                 Unit is ready and clustered
   keystone/2               active    idle   2/lxd/0  10.0.0.72       5000/tcp  Unit is ready
     keystone-hacluster/1   active    idle            10.0.0.72                 Unit is ready and clustered

In summary, the principal leader unit is keystone/0 and is deployed on machine
0/lxd/0 (the principal leader machine).

#. Set the default series for both the model and the principal application:

   .. code-block:: none

      juju model-config default-series=bionic
      juju set-series keystone bionic

#. Pause the hacluster units not associated with the principal leader machine:

   .. code-block:: none

      juju run-action --wait keystone-hacluster/1 pause
      juju run-action --wait keystone-hacluster/2 pause

#. Pause the principal non-leader units:

   .. code-block:: none

      juju run-action --wait keystone/1 pause
      juju run-action --wait keystone/2 pause

#. Perform any workload maintenance pre-upgrade steps on all machines. There
   are no keystone-specific steps to perform.

#. Invoke the :command:`prepare` sub-command on all machines, **starting with
   the principal leader machine**:

   .. code-block:: none

      juju upgrade-series 0/lxd/0 prepare bionic
      juju upgrade-series 1/lxd/0 prepare bionic
      juju upgrade-series 2/lxd/0 prepare bionic

   At this point the :command:`juju status` output looks like this:

   .. code-block:: console

      Model    Controller       Cloud/Region    Version  SLA          Timestamp
      upgrade  maas-controller  mymaas/default  2.7.6    unsupported  23:11:01Z

      App                 Version  Status   Scale  Charm            Store       Rev  OS      Notes
      keystone            13.0.2   blocked      3  keystone         jujucharms  312  ubuntu
      keystone-hacluster           blocked      3  hacluster        jujucharms   66  ubuntu

      Unit                     Workload  Agent  Machine  Public address  Ports     Message
      keystone/0*              blocked   idle   0/lxd/0  10.0.0.70       5000/tcp  Ready for do-release-upgrade and reboot. Set complete when finished.
        keystone-hacluster/0*  blocked   idle            10.0.0.70                 Ready for do-release-upgrade. Set complete when finished
      keystone/1               blocked   idle   1/lxd/0  10.0.0.71       5000/tcp  Ready for do-release-upgrade and reboot. Set complete when finished.
        keystone-hacluster/2   blocked   idle            10.0.0.71                 Ready for do-release-upgrade. Set complete when finished
      keystone/2               blocked   idle   2/lxd/0  10.0.0.72       5000/tcp  Ready for do-release-upgrade and reboot. Set complete when finished.
        keystone-hacluster/1   blocked   idle            10.0.0.72                 Ready for do-release-upgrade. Set complete when finished

#. Upgrade the operating system on all machines. The non-interactive method is
   used here:

   .. code-block:: none

      juju run --machine=0/lxd/0,1/lxd/0,2/lxd/0 --timeout=10m \
         -- sudo apt-get update
      juju run --machine=0/lxd/0,1/lxd/0,2/lxd/0 --timeout=60m \
         -- sudo DEBIAN_FRONTEND=noninteractive apt-get --assume-yes \
         -o "Dpkg::Options::=--force-confdef" \
         -o "Dpkg::Options::=--force-confold" dist-upgrade
      juju run --machine=0/lxd/0,1/lxd/0,2/lxd/0 --timeout=120m \
         -- sudo DEBIAN_FRONTEND=noninteractive \
         do-release-upgrade -f DistUpgradeViewNonInteractive

   .. important::

      Choose values for the ``--timeout`` option that are appropriate for the
      task at hand.

#. Perform any workload maintenance post-upgrade steps on all machines. There
   are no keystone-specific steps to perform.

#. Reboot all machines:

   .. code-block:: none

      juju run --machine=0/lxd/0,1/lxd/0,2/lxd/0 -- sudo reboot

#. Set the value of the ``openstack-origin`` configuration option to 'distro':

   .. code-block:: none

      juju config keystone openstack-origin=distro

#. Invoke the :command:`complete` sub-command on all machines:

   .. code-block:: none

      juju upgrade-series 0/lxd/0 complete
      juju upgrade-series 1/lxd/0 complete
      juju upgrade-series 2/lxd/0 complete

Upgrading multiple API applications concurrently
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This example procedure will be based on the nova-cloud-controller and glance
applications.

This scenario is represented by the following partial :command:`juju status`
command output:

.. code-block:: console

   Model    Controller       Cloud/Region    Version  SLA          Timestamp
   upgrade  maas-controller  mymaas/default  2.7.6    unsupported  19:23:41Z

   App                    Version  Status  Scale  Charm                  Store       Rev  OS      Notes
   glance                 16.0.1   active      3  glance                 jujucharms  295  ubuntu
   glance-hacluster                active      3  hacluster              jujucharms   66  ubuntu
   nova-cc-hacluster               active      3  hacluster              jujucharms   66  ubuntu
   nova-cloud-controller  17.0.12  active      3  nova-cloud-controller  jujucharms  343  ubuntu

   Unit                      Workload  Agent  Machine  Public address  Ports              Message
   glance/0*                 active    idle   0/lxd/0  10.246.114.39   9292/tcp           Unit is ready
     glance-hacluster/0*     active    idle            10.246.114.39                      Unit is ready and clustered
   glance/1                  active    idle   1/lxd/0  10.246.114.40   9292/tcp           Unit is ready
     glance-hacluster/1      active    idle            10.246.114.40                      Unit is ready and clustered
   glance/2                  active    idle   2/lxd/0  10.246.114.41   9292/tcp           Unit is ready
     glance-hacluster/2      active    idle            10.246.114.41                      Unit is ready and clustered
   nova-cloud-controller/0   active    idle   3/lxd/0  10.246.114.48   8774/tcp,8778/tcp  Unit is ready
     nova-cc-hacluster/2     active    idle            10.246.114.48                      Unit is ready and clustered
   nova-cloud-controller/1*  active    idle   4/lxd/0  10.246.114.43   8774/tcp,8778/tcp  Unit is ready
     nova-cc-hacluster/0*    active    idle            10.246.114.43                      Unit is ready and clustered
   nova-cloud-controller/2   active    idle   5/lxd/0  10.246.114.47   8774/tcp,8778/tcp  Unit is ready
     nova-cc-hacluster/1     active    idle            10.246.114.47                      Unit is ready and clustered

In summary,

* The glance principal leader unit is glance/0 and is deployed on machine
  0/lxd/0 (the glance principal leader machine).
* The nova-cloud-controller principal leader unit is nova-cloud-controller/1
  and is deployed on machine 4/lxd/0 (the nova-cloud-controller principal
  leader machine).

The procedure has been expedited slightly by adding the ``--yes`` confirmation
option to the :command:`prepare` sub-command.

#. Set the default series for both the model and the principal applications:

   .. code-block:: none

      juju model-config default-series=bionic
      juju set-series glance bionic
      juju set-series nova-cloud-controller bionic

#. Pause the hacluster units not associated with their principal leader
   machines:

   .. code-block:: none

      juju run-action --wait glance-hacluster/1 pause
      juju run-action --wait glance-hacluster/2 pause
      juju run-action --wait nova-cc-hacluster/1 pause
      juju run-action --wait nova-cc-hacluster/2 pause

#. Pause the principal non-leader units:

   .. code-block:: none

      juju run-action --wait glance/1 pause
      juju run-action --wait glance/2 pause
      juju run-action --wait nova-cloud-controller/0 pause
      juju run-action --wait nova-cloud-controller/2 pause

#. Perform any workload maintenance pre-upgrade steps on all machines. There
   are no glance-specific or nova-cloud-controller-specific steps to perform.

#. Invoke the :command:`prepare` sub-command on all machines, **starting with
   the principal leader machines**:

   .. code-block:: none

      juju upgrade-series --yes 0/lxd/0 prepare bionic
      juju upgrade-series --yes 4/lxd/0 prepare bionic
      juju upgrade-series --yes 1/lxd/0 prepare bionic
      juju upgrade-series --yes 2/lxd/0 prepare bionic
      juju upgrade-series --yes 3/lxd/0 prepare bionic
      juju upgrade-series --yes 5/lxd/0 prepare bionic

#. Upgrade the operating system on all machines. The non-interactive method is
   used here:

   .. code-block:: none

      juju run --machine=0/lxd/0,1/lxd/0,2/lxd/0,3/lxd/0,4/lxd/0,5/lxd/0 \
         --timeout=20m -- sudo apt-get update
      juju run --machine=0/lxd/0,1/lxd/0,2/lxd/0,3/lxd/0,4/lxd/0,5/lxd/0 \
         --timeout=120m -- sudo DEBIAN_FRONTEND=noninteractive apt-get --assume-yes \
         -o "Dpkg::Options::=--force-confdef" \
         -o "Dpkg::Options::=--force-confold" dist-upgrade
      juju run --machine=0/lxd/0,1/lxd/0,2/lxd/0,3/lxd/0,4/lxd/0,5/lxd/0 \
         --timeout=200m -- sudo DEBIAN_FRONTEND=noninteractive \
         do-release-upgrade -f DistUpgradeViewNonInteractive

#. Perform any workload maintenance post-upgrade steps on all machines. There
   are no glance-specific or nova-cloud-controller-specific steps to perform.

#. Reboot all machines:

   .. code-block:: none

      juju run --machine=0/lxd/0,1/lxd/0,2/lxd/0,3/lxd/0,4/lxd/0,5/lxd/0 -- sudo reboot

#. Set the value of the ``openstack-origin`` configuration option to 'distro':

   .. code-block:: none

      juju config glance openstack-origin=distro
      juju config nova-cloud-controller openstack-origin=distro

#. Invoke the :command:`complete` sub-command on all machines:

   .. code-block:: none

      juju upgrade-series 0/lxd/0 complete
      juju upgrade-series 1/lxd/0 complete
      juju upgrade-series 2/lxd/0 complete
      juju upgrade-series 3/lxd/0 complete
      juju upgrade-series 4/lxd/0 complete
      juju upgrade-series 5/lxd/0 complete

Physical machines
~~~~~~~~~~~~~~~~~

This section covers series upgrade procedures for applications hosted on
physical machines in particular. These typically include:

* ceph-osd
* neutron-gateway
* nova-compute

When performing a series upgrade on a physical machine more attention should be
given to any workload maintenance pre-upgrade steps:

* For compute nodes migrate all running VMs to another hypervisor.
* For network nodes force HA routers off of the current node.
* Any storage related tasks that may be required.
* Any site specific tasks that may be required.

The following two sub-sections will show how to perform a series upgrade
for a single physical machine and for multiple physical machines concurrently.

Upgrading a single physical machine
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This example procedure will be based on the nova-compute and ceph-osd
applications residing on the same physical machine. Since application
leadership does not play a significant role with these two applications, and
because the hacluster application is not present, there will be no units to
pause (as there were in previous scenarios).

This scenario is represented by the following partial :command:`juju status`
command output:

.. code-block:: console

   Model    Controller       Cloud/Region    Version  SLA          Timestamp
   upgrade  maas-controller  mymaas/default  2.7.6    unsupported  15:23:21Z

   App           Version  Status  Scale  Charm         Store       Rev  OS      Notes
   ceph-osd      12.2.12  active      1  ceph-osd      jujucharms  301  ubuntu
   keystone      13.0.2   active      1  keystone      jujucharms  312  ubuntu
   nova-compute  17.0.12  active      1  nova-compute  jujucharms  314  ubuntu

   Unit             Workload  Agent  Machine  Public address  Ports     Message
   ceph-osd/0*      active    idle   0        10.0.0.235                Unit is ready (1 OSD)
   keystone/0*      active    idle   0/lxd/0  10.0.0.240      5000/tcp  Unit is ready
   nova-compute/0*  active    idle   0        10.0.0.235                Unit is ready

   Machine  State    DNS         Inst id              Series  AZ     Message
   0        started  10.0.0.235  node1                xenial  zone1  Deployed
   0/lxd/0  started  10.0.0.240  juju-88b27a-0-lxd-0  xenial  zone1  Container started

In summary, the ceph-osd and nova-compute applications are hosted on machine 0.
Recall that container 0/lxd/0 will need to have its series upgraded separately.

#. It is recommended to set the Ceph cluster OSDs to 'noout'. This is typically
   done at the application level (i.e. not at the unit or machine level):

   .. code-block:: none

      juju run-action --wait ceph-mon/leader set-noout

#. All running VMs should be migrated to another hypervisor.

#. Upgrade the series on machine 0:

   #. Invoke the :command:`prepare` sub-command:

      .. code-block:: none

         juju upgrade-series 0 prepare bionic

   #. Upgrade the operating system:

      .. code-block:: none

         juju run --machine=0 -- sudo apt update
         juju ssh 0 sudo apt full-upgrade
         juju ssh 0 sudo do-release-upgrade

   #. Reboot (if not already done):

      .. code-block:: none

         juju run --machine=0 -- sudo reboot

   #. Set the value of the ``openstack-origin`` or ``source`` configuration
      options to 'distro':

      .. code-block:: none

         juju config nova-compute openstack-origin=distro
         juju config ceph-osd source=distro

   #. Invoke the :command:`complete` sub-command on the machine:

      .. code-block:: none

         juju upgrade-series 0 complete

#. If OSDs were previously set to 'noout' then check up/in status of those
   OSDs in ceph status, then unset 'noout' for the cluster:

   .. code-block:: none

      juju run --unit ceph-mon/leader -- ceph status
      juju run-action --wait ceph-mon/leader unset-noout

Upgrading multiple physical hosts concurrently
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

When physical machines have their series upgraded concurrently Availability
Zones need to be taken into account. Machines should be placed into upgrade
groups such that any API services running on them have a maximum of one unit
per group. This is to ensure API availability at the reboot stage.

This simplified bundle is used to demonstrate the general idea:

.. code-block:: yaml

   series: xenial
   machines:
     0: {}
     1: {}
     2: {}
     3: {}
     4: {}
     5: {}
   applications:
     nova-compute:
       charm: cs:nova-compute
       num_units: 3
       options:
         openstack-origin: cloud:xenial-queens
       to:
         - 0
         - 2
         - 4
     keystone:
       charm: cs:keystone
       constraints: mem=1G
       num_units: 3
       options:
         vip: 10.85.132.200
         openstack-origin: cloud:xenial-queens
       to:
         - lxd:1
         - lxd:3
         - lxd:5
     keystone-hacluster:
       charm: cs:hacluster
       options:
         cluster_count: 3

Three upgrade groups could consist of the following machines:

#. Machines 0 and 1
#. Machines 2 and 3
#. Machines 4 and 5

In this way, a less time-consuming series upgrade can be performed while still
ensuring the availability of services.

.. caution::

   For the ceph-osd application, ensure that rack-aware replication rules exist
   in the CRUSH map if machines are being rebooted together. This is to prevent
   significant interruption to running workloads from occurring if the
   same placement group is hosted on those machines. For example, if ceph-mon
   is deployed with ``customize-failure-domain`` set to 'true' and the ceph-osd
   units are hosted on machines in three or more separate Juju AZs you can
   safely reboot ceph-osd machines concurrently in the same zone. See
   :ref:`Ceph AZ <ceph_az>` in :doc:`OpenStack high availability <app-ha>` for
   details.

Automation
----------

Series upgrades across an OpenStack cloud can be time consuming, even when
using concurrent methods wherever possible. They can also be tedious and thus
susceptible to human error.

The following code examples encapsulate the processes described in this
document. They are provided solely to illustrate the methods used to develop
and test the series upgrade primitives:

* `Parallel tests`_: An example that is used as a functional verification of
  a series upgrade in the OpenStack Charms project.
* `Upgrade helpers`_: A set of helpers used in the above upgrade example.

.. caution::

   The example code should only be used for its intended use case of
   development and testing. Do not attempt to automate a series upgrade on a
   production cloud.

.. LINKS
.. _Series upgrade: upgrade-series.html
.. _Parallel tests: https://github.com/openstack-charmers/zaza-openstack-tests/blob/c492ecdcac3b2724833c347e978de97ea2e626d7/zaza/openstack/charm_tests/series_upgrade/parallel_tests.py#L64
.. _Upgrade helpers: https://github.com/openstack-charmers/zaza-openstack-tests/blob/9cec2efabe30fb0709bc098c48ec10bcb85cc9d4/zaza/openstack/utilities/parallel_series_upgrade.py
.. _Upgrading Percona XtraDB Cluster: https://www.percona.com/doc/percona-xtradb-cluster/LATEST/howtos/upgrade_guide.html
.. _Percona XtraDB Cluster In-Place Upgrading Guide From 5.5 to 5.6: https://www.percona.com/doc/percona-xtradb-cluster/5.6/upgrading_guide_55_56.html
.. _Galera replication - how to recover a PXC cluster: https://www.percona.com/blog/2014/09/01/galera-replication-how-to-recover-a-pxc-cluster
.. _mysql-innodb-cluster: https://jaas.ai/mysql-innodb-cluster
.. _mysql-router: https://jaas.ai/mysql-router
.. _percona-cluster charm - series upgrade to focal: percona-series-upgrade-to-focal.html
