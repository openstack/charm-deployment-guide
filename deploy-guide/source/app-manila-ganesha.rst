Appendix P: Manila Ganesha: Ceph-backed Shared Filesystem Service
=================================================================

Overview
++++++++

As of the 20.02 charm release, with OpenStack Rocky or later, support for
integrating Manila with CephFS to provide shared filesystems is available.

Three new charms are needeed to deploy this solution: manila, manila-ganesha,
and ceph-fs. The manila charm provides the manila API service to the OpenStack
deployment, the ceph-fs charm provides the Ceph services required to provide
CephFS, and the manila-ganesha charm integrates these two via a Manila managed
NFS gateway (Ganesha) to provide access controlled NFS mounts to OpenStack instances.

Deployment
++++++++++

 One way to add Manila Ganesha is to do so during the bundle
 deployment of a new OpenStack cloud. This is done by means of a
 bundle overlay, such as `manila-ganesha-overlay.yaml`:

.. code::

    machines:
      '0':
        series: bionic
      '1':
        series: bionic
      '2':
        series: bionic
      '3':
        series: bionic
    relations:
    - - manila:ha
      - manila-hacluster:ha
    - - manila-ganesha:ha
      - manila-ganesha-hacluster:ha
    - - ceph-mon:mds
      - ceph-fs:ceph-mds
    - - ceph-mon:client
      - manila-ganesha:ceph-client
    - - manila-ganesha:shared-db
      - percona-cluster:shared-db
    - - manila-ganesha:amqp
      - rabbitmq-server:amqp
    - - manila-ganesha:identity-service
      - keystone:identity-service
    - - manila:remote-manila-plugin
      - manila-ganesha:manila-plugin
    - - manila:amqp
      - rabbitmq-server:amqp
    - - manila:identity-service
      - keystone:identity-service
    - - manila:shared-db
      - percona-cluster:shared-db
    series: bionic
    applications:
      ceph-fs:
        charm: cs:ceph-fs
        num_units: 2
        options:
          source: cloud:bionic-rocky
      manila-hacluster:
        charm: cs:hacluster
      manila-ganesha-hacluster:
        charm: cs:hacluster
      manila-ganesha:
        charm: cs:manila-ganesha
        series: bionic
        num_units: 3
        options:
          openstack-origin: cloud:bionic-rocky
          vip: <INSERT VIP(S)>
        bindings:
          public: public
          admin: admin
          internal: internal
          shared-db: internal
          amqp: internal
        to:
        - 'lxd:1'
        - 'lxd:2'
        - 'lxd:3'
      manila:
        charm: cs:manila
        series: bionic
        num_units: 3
        options:
          openstack-origin: cloud:bionic-rocky
          vip: <INSERT VIP(S)>
          default-share-backend: cephfsnfs1
          share-protocols: NFS
        bindings:
          public: public
          admin: admin
          internal: internal
          shared-db: internal
          amqp: internal
        to:
        - 'lxd:1'
        - 'lxd:2'
        - 'lxd:3'

.. warning::

    The machine mappings will almost certainly need to be changed.

To use the overlay with an existing model remember to use the
`--map-machines` switch to juju. To deploy OpenStack with Manila-Ganesha:

.. code::

    juju deploy base.yaml --overlay manila-ganesha-overlay.yaml --map-machines=existing

Configuration
++++++++++++++++++

To create and access CephFS shares over NFS, you'll need to `create the share`_
and then you'll need to `grant access`_ to the share.

.. LINKS
.. _create the share: https://docs.openstack.org/manila/latest/admin/cephfs_driver.html#create-cephfs-nfs-share
.. _grant access: https://docs.openstack.org/manila/latest/admin/cephfs_driver.html#allow-access-to-cephfs-nfs-share
