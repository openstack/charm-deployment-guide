Appendix M: Ceph Erasure Coding and Device Classing
===================================================

Overview
++++++++

This appendix is intended as a post deployment guide to re-configuring RADOS
gateway pools to use erasure coding rather than replication.  It also covers
use of a specific device class (NVMe, SSD or HDD) when creating the erasure
coding profile as well as other configuration options that need to be
considered during deployment.

.. note::

    Any existing data is maintained by following this process, however
    reconfiguration should take place immediately post deployment to avoid
    prolonged ‘copy-pool’ operations.

RADOS Gateway bucket weighting
++++++++++++++++++++++++++++++

The weighting of the various pools in a deployment drives the number of
placement groups (PG’s) created to support each pool.  In the ceph-radosgw
charm this is configured for the data bucket using:

.. code::

  	juju config ceph-radosgw rgw-buckets-pool-weight=20

Note the default of 20% - if the deployment is a pure ceph-radosgw
deployment this value should be increased to the expected % use of
storage.  The device class also needs to be taken into account (but
for erasure coding this needs to be specified post deployment via action
execution).

Ceph automatic device classing
++++++++++++++++++++++++++++++

Newer versions of Ceph do automatic classing of OSD devices. Each OSD
will be placed into ‘nvme’, ‘ssd’ or ‘hdd’ device classes.  These can
be used when creating erasure profiles or new CRUSH rules (see following
sections).

The classes can be inspected using:

.. code::

    sudo ceph osd crush tree

    ID CLASS WEIGHT  TYPE NAME
    -1       8.18729 root default
    -5       2.72910     host node-laveran
     2  nvme 0.90970         osd.2
     5   ssd 0.90970         osd.5
     7   ssd 0.90970         osd.7
    -7       2.72910     host node-mees
     1  nvme 0.90970         osd.1
     6   ssd 0.90970         osd.6
     8   ssd 0.90970         osd.8
    -3       2.72910     host node-pytheas
     0  nvme 0.90970         osd.0
     3   ssd 0.90970         osd.3
     4   ssd 0.90970         osd.4


Configuring erasure coding
++++++++++++++++++++++++++

The RADOS gateway makes use of a number of pools, but the only pool
that should be converted to use erasure coding (EC) is the data pool:

.. code::

    default.rgw.buckets.data

All other pools should be replicated as they are by default.

To create a new EC profile and pool:

.. code::

    juju run-action --wait ceph-mon/0 create-erasure-profile \
        name=nvme-ec device-class=nvme

    juju run-action --wait ceph-mon/0 create-pool \
      	name=default.rgw.buckets.data.new \
    	pool-type=erasure \
    	erasure-profile-name=nvme-ec \
    	percent-data=90

The percent-data option should be set based on the type of deployment
but if the RADOS gateway is the only target for the NVMe storage class,
then 90% is appropriate (other RADOS gateway pools are tiny and use
between 0.10% and 3% of storage)

.. note::

    The create-erasure-profile action has a number of other
    options including adjustment of the K/M values which affect the
    computational overhead and underlying storage consumed per MB stored.
    Sane defaults are provided but they require a minimum of five hosts
    with block devices of the right class.

To avoid any creation/mutation of stored data during migration,
shutdown all RADOS gateway instances:

.. code::

    juju run --application ceph-radosgw \
        "sudo systemctl stop ceph-radosgw.target"

The existing buckets.data pool can then be copied and switched:

.. code::

    juju run-action --wait ceph-mon/0 rename-pool \
    	name=default.rgw.buckets.data \
    	new-name=default.rgw.buckets.data.old

    juju run-action --wait ceph-mon/0 rename-pool \
    	name=default.rgw.buckets.data.new \
	    new-name=default.rgw.buckets.data

At this point the RADOS gateway instances can be restarted:

.. code::

    juju run --application ceph-radosgw \
        "sudo systemctl start ceph-radosgw.target"

Once successful operation of the deployment has been confirmed,
the old pool can be deleted:

.. code::

    juju run-action --wait ceph-mon/0 delete-pool \
        name=default.rgw.buckets.data.old

Moving other RADOS gateway pools to NVMe storage
++++++++++++++++++++++++++++++++++++++++++++++++

The buckets.data pool is the largest pool and the one that can make
use of EC; other pools could also be migrated to the same storage
class for consistent performance:

.. code::

    juju run-action --wait ceph-mon/0 create-crush-rule \
        name=replicated_nvme device-class=nvme

The CRUSH rule for the other RADOS gateway pools can then be updated:

.. code::

    pools=".rgw.root
    default.rgw.control
    default.rgw.data.root
    default.rgw.gc
    default.rgw.log
    default.rgw.intent-log
    default.rgw.meta
    default.rgw.usage
    default.rgw.users.keys
    default.rgw.users.uid
    default.rgw.buckets.extra
    default.rgw.buckets.index
    default.rgw.users.email
    default.rgw.users.swift"

    for pool in $pools; do
        juju run-action --wait ceph-mon/0 pool-set \
            name=$pool key=crush_rule value=replicated_nvme
    done
