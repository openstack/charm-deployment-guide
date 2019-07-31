Appendix K: Ceph RBD Mirroring
==============================

Overview
++++++++

The ``ceph-rbd-mirror`` charm supports deployment of the Ceph RBD Mirror daemon
and helps automate remote creation and configuration of mirroring for Ceph
pools used to host RBD images.

Actions for operator driven failover and fallback for the pools used for RBD
images is also provided.

.. warning::

    Data center redundancy is a large topic and this work addresses a very
    specific piece in the puzzle related to Ceph RBD images.  You need to
    combine this with `Ceph RADOS Gateway Multisite replication`_ and other
    work to get a complete solution.

.. _Ceph RADOS Gateway Multisite replication: https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/app-rgw-multisite.html

This is supported both for multiple distinct Ceph clusters within a single Juju
model and between different models with help from cross-model relations.

When the charm is related to a local and a remote Ceph cluster it will
automatically create pools eligible for mirroring on the remote cluster and
enable mirroring.

Eligible pools are selected on the basis of Ceph pool tagging and all pools
with the application ``rbd`` enabled on them will be selected.

.. note::

    As of the 19.04 charm release charms will automatically have newly created
    pools for use with RBD tagged with the ``rbd`` tag.

.. warning::

    Only mirroring of whole pools is supported by the charm.

A prerequisite for RBD Mirroring is that every RBD image within each pool is
created with the ``journaling`` and ``exclusive-lock`` image features enabled.

To support this the ``ceph-mon`` charm will announce these image features over
the ``client`` relation when it has units connected to its ``rbd-mirror``
endpoint.  This will ensure that images created in the deployment get the
appropriate features to support mirroring.

.. warning::

    RBD Mirroring is only supported by the charm when deployed with Ceph
    Luminous or later.

    The feature itself appeared upstream in Ceph Jewel but the ability to run
    multiple rbd-mirror daemons per Ceph cluster first appeared in Ceph
    Luminous.

    Attempts to deploy with earlier versions of Ceph may work if you do not
    deploy multiple ceph-rbd-mirror units per cluster, but we have done no
    validation of this.

The Ceph RBD Mirror feature supports running multiple instances of the daemon.
Having multiple daemons will cause the mirroring load to automatically be
(re-)distributed between the daemons.

This addresses both High Availability and performance concerns.  You can
make use of this feature by increasing the number of ``ceph-rbd-mirror`` units
in your deployment.

.. warning::

    The charm is written for Two-way Replication, which give you the ability to
    fail over and fall back to/from a secondary site.

    Ceph does have support for mirroring to any number of slave clusters but
    this is neither implemented nor supported by the charm.

The charm is aware of network spaces and you will be able to tell the RBD
Mirror daemon about network configuration by binding the ``public`` and
``cluster`` endpoints.

The RBD Mirror daemon will use the network associated with the ``cluster``
endpoint for mirroring traffic when available.

Deployment
++++++++++

This section will cover the essential commands for deploying two Ceph clusters
(site 'a' and site 'b') with RBD mirroring between them. Two scenarios will be
shown:

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
    application "ceph-rbd-mirror-b" endpoints [ceph-remote] available at "admin/site-b.ceph-rbd-mirror-b"

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

Usage
+++++

Pools
-----

Pools created by other charms through the Ceph broker protocol will
automatically be detected and acted upon.  Pools tagged with the ``rbd``
application will be selected for mirroring.

If you manually create a pool, either through actions on the ``ceph-mon``
charm or by talking to Ceph directly, you must inform the ``ceph-rbd-mirror``
charm about them.

This is accomplished by executing the ``refresh-pools`` action.

.. code::

    juju run-action -m site-a ceph-mon/leader --wait create-pool name=mypool \
        app-name=rbd
    juju run-action -m site-a ceph-rbd-mirror/leader --wait refresh-pools


Failover and Fallback
---------------------

Controlled failover and fallback

.. code::

    juju run-action -m site-a ceph-rbd-mirror/leader --wait status verbose=True
    juju run-action -m site-b ceph-rbd-mirror/leader --wait status verbose=True

.. code::

    juju run-action -m site-a ceph-rbd-mirror/leader --wait demote

.. code::

    juju run-action -m site-a ceph-rbd-mirror/leader --wait status verbose=True
    juju run-action -m site-b ceph-rbd-mirror/leader --wait status verbose=True

.. code::

    juju run-action -m site-b ceph-rbd-mirror/leader --wait promote

.. note::

    When using Ceph Luminous, the mirror status information may not be
    accurate.  Specifically the ``entries_behind_master`` counter may never get
    to ``0`` even though the image is fully synchronized.

Recovering from abrupt shutdown
-------------------------------

There exist failure scenarios where abrupt shutdown and/or interruptions to
communication may lead to a split-brain situation where the RBD Mirroring
process in both Ceph clusters claim to be the primary.

In such a situation the operator must decide which cluster has the most
recent data and should be elected primary by using the ``demote`` and
``promote`` (optionally with force parameter) actions.

After making this decision the secondary cluster must be resynced to track
the promoted master, this is done by running the ``resync-pools`` action on
the non-master cluster.

.. code::

    juju run-action -m site-b ceph-rbd-mirror/leader --wait demote
    juju run-action -m site-a ceph-rbd-mirror/leader --wait promote force=True

    juju run-action -m site-a ceph-rbd-mirror/leader --wait status verbose=True
    juju run-action -m site-b ceph-rbd-mirror/leader --wait status verbose=True

    juju run-action -m site-b ceph-rbd-mirror/leader --wait resync-pools i-really-mean-it=True

.. note::

    When using Ceph Luminous, the mirror state information will not be accurate
    after recovering from unclean shutdown.  Regardless of the output of the
    status information you will be able to write to images after a forced
    promote.
