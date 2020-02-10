================================
Appendix R: Swift Global Cluster
================================

Overview
--------

As of the 20.02 charm release, with OpenStack Newton or later, support for
a global cluster in swift is available. See the `Upstream Overview`_ for
an overview of the feature.

.. warning::

   The Swift Global Cluster feature is in a preview state (still under
   development).

Swift global cluster is primarily useful when there are groups of Swift
storage and proxy nodes in different physical regions creating a
geographically-distributed cluster. From a Juju perspective these regions
typically exist in distinct models. To join the regions together the Juju
`Cross model relations`_
feature is needed. The example below assumes distinct models but Swift Global
Cluster can also work within a single model.

Basic configuration
+++++++++++++++++++

To enable the Swift global cluster feature a few extra charm configuration
options need to be set. The feature needs to be enables by setting
``enable-multi-region`` to ``True`` in the 'swift-proxy' charm. The storage nodes
need to be allocated to storage regions, this is done by the ``storage-region``
option in the 'swift-storage' charm. Usually all storage nodes in the same
geographic location will be placed in the same storage zone.

.. note::

   The name of a swift storage region must be of the form 'rN'
   where N in an integer. This means that the name of the storage region may
   be different from the name of an OpenStack region.

Tuning configuration
++++++++++++++++++++

The 'swift-proxy' charm exposes configuration options for tuning the
deployment. The ``read-affinity`` option can be used to control what order
the regions and zones are examined when searching for an object. A common
approach would be to put the local region first on the search path for a proxy.
For instance, in the deployment example below the Swift proxy in New York is
configured to read from the New York storage nodes first. Similarly the San
Francisco proxy prefers storage nodes in San Francisco.

The ``write-affinity`` option allows nodes to be stored locally before being
eventually distributed globally. This write_affinity setting is useful only when
you do not read objects immediately after writing them.

The ``write-affinity-node-count`` option is used to further configure
``write-affinity``. This option dictates how many local storage servers will
be tried before falling back to remote ones.

For more details see the `Upstream Overview`_.

Deployment
++++++++++

This example assumes there are two datacenters, one in San Francisco (SF) and
one in New York (NY). These contain the respective Juju models called 'swift-sf'
and 'swift-ny'. 'swift-ny' contains the OpenStack region 'RegionOne' and the swift
storage region '1'. 'swift-sf' contains the OpenStack region 'RegionTwo' and the
swift storage region '2'.

Bundle overlays need to be used to encapsulate Juju cross-model relations. So
the deployment in each region consists of a bundle and an overlay.

Below is the contents of bundle ``swift-ny.yaml``

.. code-block:: yaml

   series: bionic
   applications:
     swift-proxy-region1:
       charm: cs:~gnuoy/swift-proxy-12
       num_units: 1
       options:
         region: RegionOne
         zone-assignment: manual
         replicas: 2
         enable-multi-region: true
         swift-hash: "global-cluster"
         read-affinity: "r1=100, r2=200"
         write-affinity: "r1, r2"
         write-affinity-node-count: '1'
         openstack-origin: cloud:bionic-train
     swift-storage-region1-zone1:
       charm: cs:~gnuoy/swift-storage-6
       num_units: 1
       options:
         storage-region: 1
         zone: 1
         block-device: /etc/swift/storage.img|2G
         openstack-origin: cloud:bionic-train
     swift-storage-region1-zone2:
       charm: cs:~gnuoy/swift-storage-6
       num_units: 1
       options:
         storage-region: 1
         zone: 2
         block-device: /etc/swift/storage.img|2G
         openstack-origin: cloud:bionic-train
     swift-storage-region1-zone3:
       charm: cs:~gnuoy/swift-storage-6
       num_units: 1
       options:
         storage-region: 1
         zone: 3
         block-device: /etc/swift/storage.img|2G
         openstack-origin: cloud:bionic-train
     percona-cluster:
       charm: cs:~openstack-charmers-next/percona-cluster
       num_units: 1
       options:
         dataset-size: 25%
         max-connections: 1000
         source: cloud:bionic-train
     keystone:
       expose: True
       charm: cs:~openstack-charmers-next/keystone
       num_units: 1
       options:
         openstack-origin: cloud:bionic-train
     glance:
       expose: True
       charm: cs:~openstack-charmers-next/glance
       num_units: 1
       options:
         openstack-origin: cloud:bionic-train
   relations:
     - - swift-proxy-region1:swift-storage
       - swift-storage-region1-zone1:swift-storage
     - - swift-proxy-region1:swift-storage
       - swift-storage-region1-zone2:swift-storage
     - - swift-proxy-region1:swift-storage
       - swift-storage-region1-zone3:swift-storage
     - - keystone:shared-db
       - percona-cluster:shared-db
     - - glance:shared-db
       - percona-cluster:shared-db
     - - glance:identity-service
       - keystone:identity-service
     - - swift-proxy-region1:identity-service
       - keystone:identity-service
     - - glance:object-store
       - swift-proxy-region1:object-store


Below is the contents of the overlay bundle ``swift-ny-offers.yaml``

.. code-block:: yaml

    applications:
      keystone:
        offers:
          keystone-offer:
            endpoints:
            - identity-service
      swift-proxy-region1:
        offers:
          swift-proxy-region1-offer:
            endpoints:
            - swift-storage
            - rings-distributor
      swift-storage-region1-zone1:
        offers:
          swift-storage-region1-zone1-offer:
            endpoints:
            - swift-storage
      swift-storage-region1-zone2:
        offers:
          swift-storage-region1-zone2-offer:
            endpoints:
            - swift-storage
      swift-storage-region1-zone3:
        offers:
          swift-storage-region1-zone3-offer:
            endpoints:
            - swift-storage


Below is the contents of bundle ``swift-sf.yaml``

.. code-block:: yaml

    series: bionic
    applications:
      swift-proxy-region2:
        charm: cs:~gnuoy/swift-proxy-12
        num_units: 1
        options:
          region: RegionTwo
          zone-assignment: manual
          replicas: 2
          enable-multi-region: true
          swift-hash: "global-cluster"
          read-affinity: "r1=100, r2=200"
          write-affinity: "r1, r2"
          write-affinity-node-count: '1'
          openstack-origin: cloud:bionic-train
      swift-storage-region2-zone1:
        charm: cs:~gnuoy/swift-storage-6
        num_units: 1
        options:
          storage-region: 2
          zone: 1
          block-device: /etc/swift/storage.img|2G
          openstack-origin: cloud:bionic-train
      swift-storage-region2-zone2:
        charm: cs:~gnuoy/swift-storage-6
        num_units: 1
        options:
          storage-region: 2
          zone: 2
          block-device: /etc/swift/storage.img|2G
          openstack-origin: cloud:bionic-train
      swift-storage-region2-zone3:
        charm: cs:~gnuoy/swift-storage-6
        num_units: 1
        options:
          storage-region: 2
          zone: 3
          block-device: /etc/swift/storage.img|2G
          openstack-origin: cloud:bionic-train
    relations:
      - - swift-proxy-region2:swift-storage
        - swift-storage-region2-zone1:swift-storage
      - - swift-proxy-region2:swift-storage
        - swift-storage-region2-zone2:swift-storage
      - - swift-proxy-region2:swift-storage
        - swift-storage-region2-zone3:swift-storage

Below is the contents of the overlay bundle ``swift-sf-consumer.yaml``.

.. code-block:: yaml

    relations:
    - - swift-proxy-region2:identity-service
      - keystone:identity-service
    - - swift-proxy-region2:swift-storage
      - swift-storage-region1-zone1:swift-storage
    - - swift-proxy-region2:swift-storage
      - swift-storage-region1-zone2:swift-storage
    - - swift-proxy-region2:swift-storage
      - swift-storage-region1-zone3:swift-storage
    - - swift-storage-region2-zone1:swift-storage
      - swift-proxy-region1:swift-storage
    - - swift-storage-region2-zone2:swift-storage
      - swift-proxy-region1:swift-storage
    - - swift-storage-region2-zone3:swift-storage
      - swift-proxy-region1:swift-storage
    - - swift-proxy-region2:rings-consumer
      - swift-proxy-region1:rings-distributor
    saas:
      keystone:
        url: admin/swift-ny.keystone-offer
      swift-proxy-region1:
        url: admin/swift-ny.swift-proxy-region1-offer
      swift-storage-region1-zone1:
        url: admin/swift-ny.swift-storage-region1-zone1-offer
      swift-storage-region1-zone2:
        url: admin/swift-ny.swift-storage-region1-zone2-offer
      swift-storage-region1-zone3:
        url: admin/swift-ny.swift-storage-region1-zone3-offer

In this case swift-ny.yaml must be deployed first as it contains the Juju
offers that swift-sf.yaml will consume:

.. code-block:: none

    juju deploy -m swift-ny ./swift-ny.yaml --overlay ./swift-ny-offers.yaml
    juju deploy -m swift-sf ./swift-sf.yaml --overlay ./swift-sf-consumer.yaml

.. LINKS
.. _Upstream Overview: https://docs.openstack.org/swift/latest/overview_global_cluster.html
.. _Cross model relations: https://jaas.ai/docs/cross-model-relations
