Appendix K: Ceph RBD Mirroring
==============================

Overview
++++++++

RADOS Block Device (RBD) mirroring is a process of asynchronous replication of
Ceph block device images between two or more Ceph clusters. Mirroring ensures
point-in-time consistent replicas of all changes to an image, including reads
and writes, block device resizing, snapshots, clones, and flattening. RBD
mirroring is mainly used for disaster recovery (i.e. having a secondary site as
a failover).

This appendix includes detailed steps for deploying Ceph clusters with RBD
mirroring with the use of the ``ceph-rbd-mirror`` charm. The `ceph-rbd-mirror charm project page`_
includes in-depth information on this charm as well as how to use it for
cluster failover and disaster recovery. It also includes more brief deployment
steps as the ones shown here.

.. warning::

    RBD mirroring is only one specific element in datacentre redundancy. Refer
    to `Ceph RADOS Gateway Multisite Replication`_ and other work to arrive at
    a complete solution.

See `Upstream Ceph documentation on RBD mirroring`_ for complete information.

Deployment
++++++++++

This section will show how to use charms to deploy two Ceph clusters (site 'a'
and site 'b') with RBD mirroring between them. Two scenarios will be shown:

#. Both clusters within the same model
#. Each cluster within a separate model

Notes that apply to both scenarios:

- each cluster/site employs 7 units (3 ``ceph-osd``, 3 ``ceph-mon``, and 1
  ``ceph-rbd-mirror``)
- application names are used to distinguish between applications in site 'a'
  from those in site 'b'
- the ``ceph-osd`` units use block device ``/dev/vdd`` for OSD storage

Using one model
---------------

Create the common model ``sites-ab``:

.. code::

    juju add-model sites-ab

Deploy units for site 'a':

.. code::

    juju deploy -n 3 ceph-osd ceph-osd-a --config osd-devices=/dev/vdd
    juju deploy -n 3 ceph-mon ceph-mon-a
    juju deploy ceph-rbd-mirror ceph-rbd-mirror-a

Deploy units for site 'b':

.. code::

    juju deploy -n 3 ceph-osd ceph-osd-b --config osd-devices=/dev/vdd
    juju deploy -n 3 ceph-mon ceph-mon-b
    juju deploy ceph-rbd-mirror ceph-rbd-mirror-b

Add relations between application endpoints. Notice how ``ceph-mon`` in one
site gets related to ``ceph-rbd-mirror`` in the other site (the "inter-site"
relations):

For site 'a',

.. code::

    juju add-relation ceph-mon-a ceph-osd-a
    juju add-relation ceph-mon-a ceph-rbd-mirror-a:ceph-local
    juju add-relation ceph-mon-a ceph-rbd-mirror-b:ceph-remote

For site 'b',

.. code::

    juju add-relation ceph-mon-b ceph-osd-b
    juju add-relation ceph-mon-b ceph-rbd-mirror-b:ceph-local
    juju add-relation ceph-mon-b ceph-rbd-mirror-a:ceph-remote

Verify the output of ``juju status`` for the model (only partial output is shown):

.. code::

    juju status -m sites-ab

    Unit                  Workload  Agent  Machine  Public address  Ports  Message
    ceph-mon-a/0*         active    idle   3        10.5.0.20              Unit is ready and clustered
    ceph-mon-a/1          active    idle   4        10.5.0.9               Unit is ready and clustered
    ceph-mon-a/2          active    idle   5        10.5.0.10              Unit is ready and clustered
    ceph-mon-b/0*         active    idle   10       10.5.0.4               Unit is ready and clustered
    ceph-mon-b/1          active    idle   11       10.5.0.11              Unit is ready and clustered
    ceph-mon-b/2          active    idle   12       10.5.0.24              Unit is ready and clustered
    ceph-osd-a/0*         active    idle   0        10.5.0.3               Unit is ready (1 OSD)
    ceph-osd-a/1          active    idle   1        10.5.0.12              Unit is ready (1 OSD)
    ceph-osd-a/2          active    idle   2        10.5.0.7               Unit is ready (1 OSD)
    ceph-osd-b/0*         active    idle   7        10.5.0.21              Unit is ready (1 OSD)
    ceph-osd-b/1          active    idle   8        10.5.0.6               Unit is ready (1 OSD)
    ceph-osd-b/2          active    idle   9        10.5.0.23              Unit is ready (1 OSD)
    ceph-rbd-mirror-a/0*  waiting   idle   6        10.5.0.30              Waiting for pools to be created
    ceph-rbd-mirror-b/0*  waiting   idle   13       10.5.0.39              Waiting for pools to be created

You're done.

Note that Ceph pools have not yet been initialised. This can be done by other
charms or directly within Ceph.

Using two models
----------------

For this scenario we use model names ``site-a`` and ``site-b``.

For site 'a',

.. code::

    juju add-model site-a
    juju deploy -n 3 ceph-osd ceph-osd-a --config osd-devices=/dev/vdd
    juju deploy -n 3 ceph-mon ceph-mon-a
    juju deploy ceph-rbd-mirror ceph-rbd-mirror-a

For site 'b',

.. code::

    juju add-model site-b
    juju deploy -n 3 ceph-osd ceph-osd-b --config osd-devices=/dev/vdd
    juju deploy -n 3 ceph-mon ceph-mon-b
    juju deploy ceph-rbd-mirror ceph-rbd-mirror-b

Add relations between local application endpoints as before:

.. code::

    juju add-relation -m site-a ceph-mon-a ceph-osd-a
    juju add-relation -m site-a ceph-mon-a ceph-rbd-mirror-a:ceph-local

    juju add-relation -m site-b ceph-mon-b ceph-osd-b
    juju add-relation -m site-b ceph-mon-b ceph-rbd-mirror-b:ceph-local

To create the inter-site relations one must export one of the application
endpoints from each model by means of an "offer". Here, we make offers for
``ceph-rbd-mirror`` in each model:

.. code::

    juju switch site-a
    juju offer ceph-rbd-mirror-a:ceph-remote
    Application "ceph-rbd-mirror-a" endpoints [ceph-remote] available at "admin/site-a.ceph-rbd-mirror-a"

    juju switch site-b
    juju offer ceph-rbd-mirror-b:ceph-remote
    Application "ceph-rbd-mirror-b" endpoints [ceph-remote] available at "admin/site-b.ceph-rbd-mirror-b"

These *cross model relations* can now be made by referring to the offer URLs
(included in the output above) as if they were applications in the local model:

.. code::

    juju add-relation -m site-a ceph-mon-a admin/site-b.ceph-rbd-mirror-b
    juju add-relation -m site-b ceph-mon-b admin/site-a.ceph-rbd-mirror-a

Verify the output of ``juju status`` for both models (only partial output is shown):

.. code::

    juju status -m site-a

    Unit                  Workload  Agent  Machine  Public address  Ports  Message
    ceph-mon-a/0*         active    idle   3        10.5.0.23              Unit is ready and clustered
    ceph-mon-a/1          active    idle   4        10.5.0.5               Unit is ready and clustered
    ceph-mon-a/2          active    idle   5        10.5.0.9               Unit is ready and clustered
    ceph-osd-a/0*         active    idle   0        10.5.0.19              Unit is ready (1 OSD)
    ceph-osd-a/1          active    idle   1        10.5.0.7               Unit is ready (1 OSD)
    ceph-osd-a/2          active    idle   2        10.5.0.10              Unit is ready (1 OSD)
    ceph-rbd-mirror-a/0*  waiting   idle   6        10.5.0.11              Waiting for pools to be created

    juju status -m site-b

    Unit                  Workload  Agent  Machine  Public address  Ports  Message
    ceph-mon-b/0*         active    idle   3        10.5.0.29              Unit is ready and clustered
    ceph-mon-b/1          active    idle   4        10.5.0.4               Unit is ready and clustered
    ceph-mon-b/2          active    idle   5        10.5.0.8               Unit is ready and clustered
    ceph-osd-b/0*         active    idle   0        10.5.0.13              Unit is ready (1 OSD)
    ceph-osd-b/1          active    idle   1        10.5.0.24              Unit is ready (1 OSD)
    ceph-osd-b/2          active    idle   2        10.5.0.33              Unit is ready (1 OSD)
    Ceph-rbd-mirror-b/0*  waiting   idle   6        10.5.0.27              Waiting for pools to be created

You're done.

.. note::

    Minimal two-cluster test bundles can be found in the ``ceph-rbd-mirror``
    charm's ``src/tests/bundles`` subdirectory. Examples include both clusters
    deployed in one model as well as in separate models.

.. LINKS

.. _ceph-rbd-mirror charm project page: https://opendev.org/openstack/charm-ceph-rbd-mirror/src/branch/master/src/README.md
.. _Ceph RADOS Gateway Multisite replication: https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/app-rgw-multisite.html
.. _Upstream Ceph documentation on RBD mirroring: https://docs.ceph.com/docs/mimic/rbd/rbd-mirroring/
