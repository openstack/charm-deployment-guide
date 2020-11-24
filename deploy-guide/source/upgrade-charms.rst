==============
Charms upgrade
==============

The Juju command to use is :command:`upgrade-charm`. For extra guidance see
`Upgrading applications`_ in the Juju documentation.

Please read the following before continuing:

* the :doc:`Upgrades overview <upgrade-overview>` page
* the OpenStack charms `Release Notes`_ for the corresponding current and
  target versions of OpenStack

.. note::

   A charm upgrade affects all corresponding units; per-unit upgrades is not
   currently supported.

Although it may be possible to upgrade some charms in parallel it is
recommended that the upgrades be performed sequentially (i.e. one at a time).
Verify a charm upgrade before moving on to the next.

In terms of the upgrade order, begin with 'keystone'. After that, the rest of
the charms can be upgraded in any order.

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

Upgrade target revisions
~~~~~~~~~~~~~~~~~~~~~~~~

By default the :command:`upgrade-charm` command will upgrade a charm to its
latest stable revision (a possible multi-step upgrade). This means that
intervening revisions can be conveniently skipped. Use the ``--revision``
option to specify a target revision.

The current revision can be discovered via :command:`juju status` output (see
column 'Rev'). For the ceph-mon charm:

.. code-block:: console

   App       Version  Status  Scale  Charm     Store       Rev  OS      Notes
   ceph-mon  13.2.8   active      3  ceph-mon  jujucharms   48  ubuntu

The latest available stable revision of a charm can be obtained by querying the
Charm Store with the :command:`charm` snap:

.. code-block:: none

   sudo snap install charm --classic
   charm pull ceph-mon

Sample output:

.. code-block:: console

   cs:ceph-mon-48

Based on the above, the ceph-mon charm does not require an upgrade.

.. important::

   As stated earlier, any kind of upgrade should first be tested in a
   pre-production environment. OpenStack charm upgrades have been tested for
   single-step upgrades only (N+1).

.. LINKS
.. _Upgrading applications: https://jaas.ai/docs/upgrading-applications
.. _Release Notes: https://docs.openstack.org/charm-guide/latest/release-notes.html
.. _Series upgrade: app-series-upgrade
