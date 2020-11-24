==============
Series upgrade
==============

The purpose of this document is to provide foundational knowledge for preparing
an administrator to perform a series upgrade across a Charmed OpenStack cloud.
This translates to upgrading the operating system of every cloud node to an
entirely new version.

Please read the following before continuing:

* the :doc:`Upgrades overview <upgrade-overview>` page
* the OpenStack charms `Release Notes`_ for the corresponding current and
  target versions of OpenStack
* the `Known OpenStack upgrade issues`_ section in the OpenStack upgrade
  document

Once this document has been studied the administrator will be ready to graduate
to the :doc:`Series upgrade OpenStack <app-series-upgrade-openstack>` guide
that describes the process in more detail.

Concerning the cloud being operated upon, the following is assumed:

* It is being upgraded from one LTS series to another (e.g. xenial to
  bionic, bionic to focal, etc.).
* Its nodes are backed by MAAS.
* Its services are highly available.
* It is being upgraded with minimal downtime.

.. warning::

   Upgrading a single production machine from one LTS to another is a serious
   task. Doing so for every cloud node can be that much harder. Attempting to
   do this with minimal cloud downtime is an order of magnitude more complex.

   Such an undertaking should be executed by persons who are intimately
   familiar with Juju and the currently deployed charms (and their related
   applications). It should first be tested on a non-production cloud that
   closely resembles the production environment.

The Juju :command:`upgrade-series` command
------------------------------------------

The Juju :command:`upgrade-series` command is the cornerstone of the entire
procedure. This command manages an operating system upgrade of a targeted
machine and operates on every application unit hosted on that machine. The
command works in conjunction with either the :command:`prepare` or the
:command:`complete` sub-command.

The basic process is to inform the units on a machine that a series upgrade
is about to commence, to perform the upgrade, and then inform the units that
the upgrade has finished. In most cases with the OpenStack charms, units will
first be paused and be left with a workload status of "blocked" and a message
of "Ready for do-release-upgrade and reboot."

For example, to inform units on machine '0' that an upgrade (to series
'bionic') is about to occur:

.. code-block:: none

   juju upgrade-series 0 prepare bionic

The :command:`prepare` sub-command causes **all** the charms (including
subordinates) on the machine to run their ``pre-series-upgrade`` hook.

The administrator must then perform the traditional steps involved in upgrading
the OS on the targeted machine (in this example, machine '0'). For example,
update/upgrade packages with :command:`apt update && apt full-upgrade`; invoke
the :command:`do-release-upgrade` command; and reboot the machine once
complete.

The :command:`complete` sub-command causes **all** the charms (including
subordinates) on the machine to run their ``post-series-upgrade`` hook. In most
cases with the OpenStack charms, configuration files will be re-written, units
will be resumed automatically (if paused), and be left with a workload status
of "active" and a message of "Unit is ready":

.. code-block:: none

   juju upgrade-series 0 complete

At this point the series upgrade on the machine and its charms is now done. In
the :command:`juju status` output the machine's entry under the Series column
will have changed from 'xenial' to 'bionic'.

.. note::

   Charms are not obliged to support the two series upgrade hooks but they do
   make for a more intelligent and a less error-prone series upgrade.

Containers (and their charms) hosted on the target machine remain unaffected by
this command. However, during the required post-upgrade reboot of the host all
containerised services will naturally be unavailable.

See the Juju documentation to learn more about the `series upgrade`_ feature.

.. _pre-upgrade_requirements:

Pre-upgrade requirements
------------------------

This is a list of requirements that apply to any cloud. They must be met before
making any changes.

* All the cloud nodes should be using the same series, be in good working
  order, and be updated with the latest stable software packages (APT
  upgrades).

* The cloud should be running the latest OpenStack release supported by the
  current series (e.g. Mitaka for trusty, Queens for xenial, etc.). See `Ubuntu
  OpenStack release cycle`_ and `OpenStack upgrades`_.

* The cloud should be fully operational and error-free.

* All currently deployed charms should be upgraded to the latest stable charm
  revision. See `Charm upgrades`_.

* The Juju model comprising the cloud should be error-free (e.g. there should
  be no charm hook errors).

* `Automatic package updates`_ should be disabled on the nodes to avoid
  potential conflicts with the manual (or scripted) APT steps.

.. _series_specific_procedures:

Specific series upgrade procedures
----------------------------------

Charms belonging to the OpenStack Charms project are designed to accommodate
the next LTS target series wherever possible. However, a new series may
occasionally introduce unavoidable challenges for a deployed charm. For
instance, it could be that a charm is replaced by an entirely new charm on the
new series. This can happen due to development policy concerning the charms
themselves (e.g. the ceph charm is replaced by the ceph-mon and ceph-osd
charms) or due to reasons independent of the charms (e.g. the workload software
is no longer supported on the new operating system). Any core OpenStack charms
affected in this way will be documented below.

* :ref:`percona-cluster charm: series upgrade to Focal <percona_series_upgrade_to_focal>`

Known series-related issues
---------------------------

Ensure that your deployment will not be adversely affected by known
series-related problems when upgrading. The following issues have been flagged
for consideration.

DNS HA with the focal series
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

DNS HA has been reported to not work on the focal series. See `LP #1882508`_
for more information.

.. _workload_specific_preparations:

Workload specific preparations
------------------------------

These are preparations that are specific to the current cloud deployment.
Completing them in advance is an integral part of the upgrade.

Charm upgradability
~~~~~~~~~~~~~~~~~~~

Verify the documented series upgrade processes for all currently deployed
charms. Some charms, especially third-party charms, may either not have
implemented series upgrade yet or simply may not work with the target series.
Pay particular attention to SDN (software defined networking) and storage
charms as these play a crucial role in cloud operations.

Workload maintenance
~~~~~~~~~~~~~~~~~~~~

Any workload-specific pre and post series upgrade maintenance tasks should be
readied in advance. For example, if a node's workload requires a database then
a pre-upgrade backup plan should be drawn up. Similarly, if a workload requires
settings to be adjusted post-upgrade then those changes should be prepared
ahead of time. Pay particular attention to stateful services due to their
importance in cloud operations. Examples include evacuating a compute node,
switching an HA router to another node, and storage rebalancing.

Pre-upgrade tasks are performed before issuing the :command:`prepare`
subcommand, and post-upgrade tasks are done immediately prior to issuing the
:command:`complete` subcommand.

Workflow: sequential vs. concurrent
-----------------------------------

In terms of the workflow there are two approaches:

* Sequential - upgrading one machine at a time
* Concurrent - upgrading a group of machines simultaneously

Normally, it is best to upgrade sequentially as this ensures data reliability
and availability (we've assumed an HA cloud). This approach also minimises
adverse effects to the deployment if something goes wrong.

However, for even moderately sized clouds, an intervention based purely on a
sequential approach can take a very long time to complete. This is where the
concurrent method becomes attractive.

In general, a concurrent approach is a viable option for API applications but
is not an option for stateful applications. During the course of the cloud-wide
series upgrade a hybrid strategy is a reasonable choice.

To be clear, the above pertains to upgrading the series on machines associated
with a single application. It is also possible however to employ similar
thinking to multiple applications.

Application leadership
----------------------

`Application leadership`_ plays an important role in determining the order in
which machines (and their applications) will have their series upgraded. The
guiding principle is that an application's unit leader is acted upon by a
series upgrade before its non-leaders are (the leader is typically used to
coordinate aspects with other services over relations).

.. note::

   Juju will not transfer the leadership of an application (and any
   subordinate) to another unit while the application is undergoing a series
   upgrade. This allows a charm to make assumptions that will lead to a more
   reliable outcome.

Assuming that a cloud is intended to eventually undergo a series upgrade, this
guideline will generally influence the cloud's topology. Containerisation is an
effective response to this.

.. important::

   Applications should be co-located on the same machine only if leadership
   plays a negligible role. Applications deployed with the compute and storage
   charms fall into this category.

.. _generic_series_upgrade:

Generic series upgrade
----------------------

This section contains a generic overview of a series upgrade for three
machines, each hosting a unit of the `ubuntu`_ application. The initial and
target series are xenial and bionic, respectively.

This scenario is represented by the following :command:`juju status` command
output:

.. code-block:: console

   Model    Controller       Cloud/Region    Version  SLA          Timestamp
   upgrade  maas-controller  mymaas/default  2.7.6    unsupported  18:33:49Z

   App      Version  Status  Scale  Charm   Store       Rev  OS      Notes
   ubuntu1  16.04    active      3  ubuntu  jujucharms   15  ubuntu

   Unit        Workload  Agent  Machine  Public address  Ports  Message
   ubuntu1/0*  active    idle   0        10.0.0.241             ready
   ubuntu1/1   active    idle   1        10.0.0.242             ready
   ubuntu1/2   active    idle   2        10.0.0.243             ready

   Machine  State    DNS         Inst id  Series  AZ     Message
   0        started  10.0.0.241  node2    xenial  zone3  Deployed
   1        started  10.0.0.242  node3    xenial  zone4  Deployed
   2        started  10.0.0.243  node1    xenial  zone5  Deployed

First ensure that any new applications will (by default) use the new series, in
this case bionic. This is done by configuring at the model level:

.. code-block:: none

   juju model-config default-series=bionic

Now do the same at the application level. This will affect any new units of the
existing application, in this case 'ubuntu1':

.. code-block:: none

   juju set-series ubuntu1 bionic

Perform the actual series upgrade. We begin with the machine that houses the
application unit leader, machine 0 (see the asterisk in the Unit column). Note
that :command:`juju run` is preferred over :command:`juju ssh` but the latter
should be used for sessions requiring user interaction:

.. code-block:: none
   :linenos:

   # Perform any workload maintenance pre-upgrade steps here
   juju upgrade-series 0 prepare bionic
   juju run --machine=0 -- sudo apt update
   juju ssh 0 sudo apt full-upgrade
   juju ssh 0 sudo do-release-upgrade
   # Perform any workload maintenance post-upgrade steps here
   # Reboot the machine (if not already done)
   juju upgrade-series 0 complete

In this generic example there are no `workload maintenance`_ steps to perform.
If there were post-upgrade steps then the prompt to reboot the machine at the
end of :command:`do-release-upgrade` should be answered in the negative and the
reboot will be initiated manually on line 7 (i.e. :command:`sudo reboot`).

It is possible to invoke the :command:`complete` sub-command before the
upgraded machine is ready to process it. Juju will block until the unit is
ready after being restarted.

In lines 4 and 5 the upgrade proceeds in the usual interactive fashion. If a
non-interactive mode is preferred, those two lines can be replaced with:

.. code-block:: none

   juju run --machine=0 --timeout=30m -- sudo DEBIAN_FRONTEND=noninteractive apt-get --assume-yes \
      -o "Dpkg::Options::=--force-confdef" \
      -o "Dpkg::Options::=--force-confold" dist-upgrade
   juju run --machine=0 --timeout=30m -- sudo DEBIAN_FRONTEND=noninteractive \
      do-release-upgrade -f DistUpgradeViewNonInteractive

The :command:`apt-get` command is preferred while in non-interactive mode (or
with scripting).

By default, an LTS release will not have an upgrade candidate until the "point
release" of the next LTS is published. You can override this policy by using
the ``-d`` (development) option with the :command:`do-release-upgrade` command.

.. caution::

   Performing a series upgrade non-interactively can be risky so the decision
   to do so should be made only after careful deliberation.

Machines 1 and 2 should now be upgraded in the same way (in no particular
order).

.. note::

   It has been reported that a trusty:xenial series upgrade may require an
   additional step to ensure a purely non-interactive mode. A file under
   ``/etc/apt/apt.conf.d`` with a single line as its contents needs to be added
   to the target machine pre-upgrade and be removed post-upgrade. It can be
   created (here on machine 0) in this way:

   juju run --machine=0 -- "echo 'DPkg::options { "--force-confdef"; "--force-confnew"; }' | sudo tee /etc/apt/apt.conf.d/local"

Next steps
----------

When you are ready to perform a series upgrade across your cloud proceed to
appendix :doc:`Series upgrade OpenStack <app-series-upgrade-openstack>`.

.. LINKS
.. _Release Notes: https://docs.openstack.org/charm-guide/latest/release-notes.html
.. _Charm upgrades: app-upgrade-openstack.html#charm-upgrades
.. _OpenStack upgrades: app-series-upgrade-openstack.html
.. _Known OpenStack upgrade issues: app-series-upgrade-openstack.html#known-openstack-upgrade-issues
.. _series upgrade: https://juju.is/docs/upgrading-series
.. _automatic package updates: https://help.ubuntu.com/lts/serverguide/automatic-updates.html.en
.. _Ubuntu OpenStack release cycle: https://ubuntu.com/about/release-cycle#ubuntu-openstack-release-cycle
.. _Application leadership: https://juju.is/docs/implementing-leadership
.. _ubuntu: https://jaas.ai/ubuntu

.. BUGS
.. _LP #1882508: https://bugs.launchpad.net/charm-deployment-guide/+bug/1882508
