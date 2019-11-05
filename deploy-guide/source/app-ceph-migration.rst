Appendix A: Ceph Charm Migration
================================

In order to continue to receive updates to newer Ceph versions, and for general
improvements and features in the charms to deploy Ceph, users of the ceph charm
should migrate existing services to using ceph-mon and ceph-osd.

.. note::

    This example migration assumes that the ceph charm is deployed to machines
    0, 1 and 2 with the ceph-osd charm deployed to other machines within the
    model.

Upgrade charms
~~~~~~~~~~~~~~

The entire suite of charms used to manage the cloud should be upgraded to the
latest stable charm revision before any major change is made to the cloud such
as the current migration to these new charms. See `Charm upgrades`_ for
guidance.

Deploy ceph-mon
~~~~~~~~~~~~~~~

First deploy the ceph-mon charm; if the existing ceph charm is deployed to machines
0, 1 and 2, you can place the ceph-mon units in LXD containers on these machines:

.. code:: bash

    juju deploy --to lxd:0 ceph-mon
    juju config ceph-mon no-bootstrap=True
    juju add-unit --to lxd:1 ceph-mon
    juju add-unit --to lxd:2 ceph-mon

These units will install ceph, but will not bootstrap into a running monitor cluster.

Bootstrap ceph-mon from ceph
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Next, we'll use the existing ceph application to bootstrap the new ceph-mon units:

.. code:: bash

    juju add-relation ceph ceph-mon

Once this process has completed, you should have a Ceph MON cluster of 6 units;
this can be verified on any of the ceph or ceph-mon units:

.. code:: bash

    sudo ceph -s

Deploy ceph-osd to ceph units
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In order to retain any running Ceph OSD processes on the ceph units, the ceph-osd
charm must be deployed to the existing machines running the ceph units:

.. code:: bash

    juju config ceph-osd osd-reformat=False
    juju add-unit --to 0 ceph-osd
    juju add-unit --to 1 ceph-osd
    juju add-unit --to 2 ceph-osd

NOTE: as of the 18.05 charm release, the osd-reformat config option has now been
completely removed.

The charm installation and configuration will not impact any existing running
Ceph OSD's.

Relate ceph-mon to all ceph clients
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The new ceph-mon units now need to be related to the ceph-osd application:

.. code:: bash

    juju add-relation ceph-mon ceph-osd

and depending on your deployment you'll also need to add relations for other
applications, for example:

.. code:: bash

    juju add-relation ceph-mon cinder-ceph
    juju add-relation ceph-mon glance
    juju add-relation ceph-mon nova-compute
    juju add-relation ceph-mon ceph-radosgw
    juju add-relation ceph-mon gnocchi

once hook execution completes across all units, each client should be configured
with 6 MON addresses.

Remove the ceph application
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Its now safe to remove the ceph application from your deployment:

.. code:: bash

    juju remove-application ceph

As each unit of the ceph application is destroyed, its stop hook will remove the
MON process from the Ceph cluster monmap and disable Ceph MON and MGR processes
running on the machine; any Ceph OSD processes remain untouched and are now
owned by the ceph-osd units deployed alongside ceph.

.. raw:: html

   <!-- LINKS -->

.. _Charm upgrades: app-upgrade-openstack#charm-upgrades
