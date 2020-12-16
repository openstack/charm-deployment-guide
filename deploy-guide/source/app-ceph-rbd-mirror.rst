==================
Ceph RBD mirroring
==================

Overview
--------

RADOS Block Device (RBD) mirroring is a process of asynchronous replication of
Ceph block device images between two or more Ceph clusters. Mirroring ensures
point-in-time consistent replicas of all changes to an image, including reads
and writes, block device resizing, snapshots, clones, and flattening. RBD
mirroring is mainly used for disaster recovery (i.e. having a secondary site as
a failover). See `Upstream Ceph documentation on RBD mirroring`_ for complete
information.

This guide will show how to deploy two Ceph clusters with RBD mirroring between
them with the use of the ceph-rbd-mirror charm. See the `charm's
documentation`_ for basic information and charm limitations.

RBD mirroring is only one aspect of datacentre redundancy. Refer to `Ceph RADOS
Gateway Multisite Replication`_ and other work to arrive at a complete
solution.

.. note::

   RBD mirroring makes use of the journaling feature of Ceph. This incurs an
   overhead for write activity on an RBD image that will adversely affect
   performance. See Florian Haas' performance analysis of `RBD mirror`_ from
   Cephalocon Barcelona 2019.

Requirements
------------

The two Ceph clusters will correspond to sites 'a' and 'b' and each cluster
will reside within a separate model (models 'site-a' and 'site-b'). The
deployment will require the use of `Cross model relations`_.

Deployment characteristics:

* each cluster will have 7 units:

  * 3 x ceph-osd
  * 3 x ceph-mon
  * 1 x ceph-rbd-mirror

* application names will be used to distinguish between applications in site
  'a' from those in site 'b' (e.g. ceph-mon-a and ceph-mon-b)

* the ceph-osd units will use block device ``/dev/vdd`` for their OSD volumes

.. note::

   The two Ceph clusters can optionally be placed within the same model, and
   thus obviate the need for cross model relations. This topology is not
   generally considered to be a real world scenario.

Deployment
----------

For site 'a' the following configuration is placed into file ``site-a.yaml``:

.. code-block:: yaml

   ceph-mon-a:
     monitor-count: 3
     expected-osd-count: 3
     source: distro

   ceph-osd-a:
     osd-devices: /dev/vdd
     source: distro

   ceph-rbd-mirror-a:
     source: distro

Create the model and deploy the software for each site:

* Site 'a'

  .. code-block:: none

     juju add-model site-a
     juju deploy -n 3 --config site-a.yaml ceph-osd ceph-osd-a
     juju deploy -n 3 --config site-a.yaml ceph-mon ceph-mon-a
     juju deploy --config site-a.yaml ceph-rbd-mirror ceph-rbd-mirror-a

* Site 'b'

  An analogous configuration file is used (i.e. replace 'a' with 'b'):

  .. code-block:: none

     juju add-model site-b
     juju deploy -n 3 --config site-b.yaml ceph-osd ceph-osd-b
     juju deploy -n 3 --config site-b.yaml ceph-mon ceph-mon-b
     juju deploy --config site-b.yaml ceph-rbd-mirror ceph-rbd-mirror-b

Add two local relations for each site:

* Site 'a'

  .. code-block:: none

     juju add-relation -m site-a ceph-mon-a:osd ceph-osd-a:mon
     juju add-relation -m site-a ceph-mon-a:rbd-mirror ceph-rbd-mirror-a:ceph-local

* Site 'b'

  .. code-block:: none

     juju add-relation -m site-b ceph-mon-b:osd ceph-osd-b:mon
     juju add-relation -m site-b ceph-mon-b:rbd-mirror ceph-rbd-mirror-b:ceph-local

Export a ceph-rbd-mirror endpoint (by means of an "offer") for each site. This
will enable us to create the inter-site (cross model) relations:

* Site 'a'

  .. code-block:: none

     juju switch site-a
     juju offer ceph-rbd-mirror-a:ceph-remote

  Output:

  .. code-block:: console

     Application "ceph-rbd-mirror-a" endpoints [ceph-remote] available at "admin/site-a.ceph-rbd-mirror-a"

* Site 'b'

  .. code-block:: none

     juju switch site-b
     juju offer ceph-rbd-mirror-b:ceph-remote

  Output:

  .. code-block:: console

     Application "ceph-rbd-mirror-b" endpoints [ceph-remote] available at "admin/site-b.ceph-rbd-mirror-b"

Add the two inter-site relations by referring to the offer URLs (included in
the output above) as if they were applications in the local model:

.. code-block:: none

   juju add-relation -m site-a ceph-mon-a admin/site-b.ceph-rbd-mirror-b
   juju add-relation -m site-b ceph-mon-b admin/site-a.ceph-rbd-mirror-a

Verify the output of :command:`juju status` for each model:

.. code-block:: none

   juju status -m site-a

Output:

.. code-block:: console

   Model   Controller   Cloud/Region    Version  SLA          Timestamp
   site-a  maas-prod-1  acme-1/default  2.8.1    unsupported  20:00:41Z

   SAAS               Status   Store        URL
   ceph-rbd-mirror-b  waiting  icarus-maas  admin/site-b.ceph-rbd-mirror-b

   App                Version  Status   Scale  Charm            Store       Rev  OS      Notes
   ceph-mon-a         15.2.3   active       3  ceph-mon         jujucharms   49  ubuntu
   ceph-osd-a         15.2.3   active       3  ceph-osd         jujucharms  304  ubuntu
   ceph-rbd-mirror-a  15.2.3   waiting      1  ceph-rbd-mirror  jujucharms   12  ubuntu

   Unit                  Workload  Agent  Machine  Public address  Ports  Message
   ceph-mon-a/0*         active    idle   0/lxd/0  10.0.0.57              Unit is ready and clustered
   ceph-mon-a/1          active    idle   1/lxd/0  10.0.0.58              Unit is ready and clustered
   ceph-mon-a/2          active    idle   2/lxd/0  10.0.0.59              Unit is ready and clustered
   ceph-osd-a/0*         active    idle   0        10.0.0.69              Unit is ready (1 OSD)
   ceph-osd-a/1          active    idle   1        10.0.0.19              Unit is ready (1 OSD)
   ceph-osd-a/2          active    idle   2        10.0.0.20              Unit is ready (1 OSD)
   ceph-rbd-mirror-a/0*  waiting   idle   3        10.0.0.22              Waiting for pools to be created

   Machine  State    DNS        Inst id              Series  AZ       Message
   0        started  10.0.0.69  virt-node-08         focal   default  Deployed
   0/lxd/0  started  10.0.0.57  juju-bb0dc1-0-lxd-0  focal   default  Container started
   1        started  10.0.0.19  virt-node-10         focal   default  Deployed
   1/lxd/0  started  10.0.0.58  juju-bb0dc1-1-lxd-0  focal   default  Container started
   2        started  10.0.0.20  virt-node-11         focal   default  Deployed
   2/lxd/0  started  10.0.0.59  juju-bb0dc1-2-lxd-0  focal   default  Container started
   3        started  10.0.0.22  virt-node-03         focal   default  Deployed

   Offer              Application        Charm            Rev  Connected  Endpoint     Interface        Role
   ceph-rbd-mirror-a  ceph-rbd-mirror-a  ceph-rbd-mirror  12   1/1        ceph-remote  ceph-rbd-mirror  requirer

.. code-block:: none

   juju status -m site-b

Output:

.. code-block:: console

   Model   Controller   Cloud/Region    Version  SLA          Timestamp
   site-b  maas-prod-1  acme-1/default  2.8.1    unsupported  20:02:58Z

   SAAS               Status   Store        URL
   ceph-rbd-mirror-a  waiting  icarus-maas  admin/site-a.ceph-rbd-mirror-a

   App                Version  Status   Scale  Charm            Store       Rev  OS      Notes
   ceph-mon-b         15.2.3   active       3  ceph-mon         jujucharms   49  ubuntu
   ceph-osd-b         15.2.3   active       3  ceph-osd         jujucharms  304  ubuntu
   ceph-rbd-mirror-b  15.2.3   waiting      1  ceph-rbd-mirror  jujucharms   12  ubuntu

   Unit                  Workload  Agent  Machine  Public address  Ports  Message
   ceph-mon-b/0*         active    idle   0/lxd/0  10.0.0.60              Unit is ready and clustered
   ceph-mon-b/1          active    idle   1/lxd/0  10.0.0.61              Unit is ready and clustered
   ceph-mon-b/2          active    idle   2/lxd/0  10.0.0.62              Unit is ready and clustered
   ceph-osd-b/0*         active    idle   0        10.0.0.21              Unit is ready (1 OSD)
   ceph-osd-b/1          active    idle   1        10.0.0.54              Unit is ready (1 OSD)
   ceph-osd-b/2          active    idle   2        10.0.0.55              Unit is ready (1 OSD)
   ceph-rbd-mirror-b/0*  waiting   idle   3        10.0.0.56              Waiting for pools to be created

   Machine  State    DNS        Inst id              Series  AZ       Message
   0        started  10.0.0.21  virt-node-02         focal   default  Deployed
   0/lxd/0  started  10.0.0.60  juju-3ef7c5-0-lxd-0  focal   default  Container started
   1        started  10.0.0.54  virt-node-04         focal   default  Deployed
   1/lxd/0  started  10.0.0.61  juju-3ef7c5-1-lxd-0  focal   default  Container started
   2        started  10.0.0.55  virt-node-05         focal   default  Deployed
   2/lxd/0  started  10.0.0.62  juju-3ef7c5-2-lxd-0  focal   default  Container started
   3        started  10.0.0.56  virt-node-06         focal   default  Deployed

   Offer              Application        Charm            Rev  Connected  Endpoint     Interface        Role
   ceph-rbd-mirror-b  ceph-rbd-mirror-b  ceph-rbd-mirror  12   1/1        ceph-remote  ceph-rbd-mirror  requirer

There are no Ceph pools created by default. The next section ('Pool creation')
provides guidance.

Pool creation
-------------

RBD pools can be created by either a supporting charm (through the Ceph broker
protocol) or manually by the operator:

#. A charm-created pool (e.g. the glance or nova-compute charms) will
   automatically be detected and acted upon (i.e. a remote pool will be set up
   in the peer cluster).

#. A manually-created pool, whether done via the ceph-mon application or
   through Ceph directly, will require an action to be run on the
   ceph-rbd-mirror application leader in order for the remote pool to come
   online.

   For example, to create a pool manually in site 'a' and have ceph-rbd-mirror
   (of site 'a') initialise a pool in site 'b':

   .. code-block:: none

      juju run-action --wait -m site-a ceph-mon-a/leader create-pool name=mypool app-name=rbd
      juju run-action --wait -m site-a ceph-rbd-mirror-a/leader refresh-pools

   This can be verified by listing the pools in site 'b':

   .. code-block:: none

      juju run-action --wait -m site-b ceph-mon-b/leader list-pools

.. note::

   Automatic peer-pool creation (for a charm-created pool) is based on the
   local pool being labelled with a Ceph 'rbd' tag. This Ceph-internal
   labelling occurs when the newly-created local pool is associated with the
   RBD application. This last feature is supported starting with Ceph Luminous
   (OpenStack Queens).

Failover and fallback
---------------------

To manage failover and fallback, the ``demote`` and ``promote`` actions are
applied to the ceph-rbd-mirror application leader.

For instance, to fail over from site 'a' to site 'b' the former is demoted and
the latter is promoted. The rest of the commands are status checks:

.. code-block:: none

   juju run-action --wait -m site-a ceph-rbd-mirror-a/leader status verbose=true
   juju run-action --wait -m site-b ceph-rbd-mirror-b/leader status verbose=true

   juju run-action --wait -m site-a ceph-rbd-mirror-a/leader demote

   juju run-action --wait -m site-a ceph-rbd-mirror-a/leader status verbose=true
   juju run-action --wait -m site-b ceph-rbd-mirror-b/leader status verbose=true

   juju run-action --wait -m site-b ceph-rbd-mirror-b/leader promote

To fall back to site 'a' the actions are reversed:

.. code-block:: none

   juju run-action --wait -m site-b ceph-rbd-mirror-b/leader demote
   juju run-action --wait -m site-a ceph-rbd-mirror-a/leader promote

.. note::

   With Ceph Luminous (and greater), the mirror status information may not be
   accurate. Specifically, the ``entries_behind_master`` counter may never get
   to '0' even though the image has been fully synchronised.

Recovering from abrupt shutdown
-------------------------------

It is possible that an abrupt shutdown and/or an interruption to communication
channels may lead to a "split-brain" condition. This may cause the mirroring
daemon in each cluster to claim to be the primary. In such cases, the operator
must make a call as to which daemon is correct. Generally speaking, this means
deciding which cluster has the most recent data.

Elect a primary by applying the ``demote`` and ``promote`` actions to the
appropriate ceph-rbd-mirror leader. After doing so, the ``resync-pools`` action
must be run on the secondary cluster leader. The ``promote`` action may require
a force option.

Here, we make site 'a' be the primary by demoting site 'b' and promoting site
'a':

.. code-block:: none

   juju run-action --wait -m site-b ceph-rbd-mirror/leader demote
   juju run-action --wait -m site-a ceph-rbd-mirror/leader promote force=true

   juju run-action --wait -m site-a ceph-rbd-mirror/leader status verbose=true
   juju run-action --wait -m site-b ceph-rbd-mirror/leader status verbose=true

   juju run-action --wait -m site-b ceph-rbd-mirror/leader resync-pools i-really-mean-it=true

.. note::

   When using Ceph Luminous, the mirror state information will not be accurate
   after recovering from unclean shutdown. Regardless of the output of the
   status information, you will be able to write to images after a forced
   promote.

.. LINKS
.. _charm's documentation: https://opendev.org/openstack/charm-ceph-rbd-mirror/src/branch/master/src/README.md
.. _Ceph RADOS Gateway Multisite replication: https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/app-rgw-multisite.html
.. _Upstream Ceph documentation on RBD mirroring: https://docs.ceph.com/docs/mimic/rbd/rbd-mirroring/
.. _RBD mirror: https://fghaas.github.io/cephalocon2019-rbdmirror/#/7/6
.. _Cross model relations: https://juju.is/docs/cross-model-relations
