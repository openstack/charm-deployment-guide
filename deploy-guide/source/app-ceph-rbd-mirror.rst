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

.. note::

    Example bundles with a minimal test configuration can be found
    in the ``tests/bundles`` subdirectory of the ``ceph-rbd-mirror`` charm.

    Both examples of two Ceph clusters deployed in one model and Ceph clusters
    deployed in separate models are available.

To make use of cross model relations you must first set up an offer to export
a application endpoint from a model.  In this example we use the model names
``site-a`` and ``site-b``.

.. code::

    juju switch site-a
    juju offer ceph-mon:rbd-mirror site-a-rbd-mirror

    juju switch site-b
    juju offer ceph-mon:rbd-mirror site-b-rbd-mirror


After creating the offers we can import the remote offer to a model and add
a relation between applications just like we normally would do in a
single-model deployment.

.. code::

    juju switch site-a
    juju consume admin/site-b.site-b-rbd-mirror
    juju add-relation ceph-rbd-mirror:ceph-remote site-b-rbd-mirror

    juju switch site-b
    juju consume admin/site-a.site-a-rbd-mirror
    juju add-relation ceph-rbd-mirror:ceph-remote site-a-rbd-mirror

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
