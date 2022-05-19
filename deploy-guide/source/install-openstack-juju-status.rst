:orphan:

.. _install_openstack_juju_status:

===============
OpenStack cloud
===============

The below :command:`juju status --relations` output represents the cloud
installed from the instructions given on the :doc:`Install OpenStack
<install-openstack>` page.

.. code-block:: console

   Model      Controller       Cloud/Region      Version  SLA          Timestamp
   openstack  maas-controller  maas-one/default  2.9.29   unsupported  18:51:46Z

   App                       Version  Status  Scale  Charm                   Channel        Rev  Exposed  Message
   ceph-mon                  17.1.0   active      3  ceph-mon                quincy/stable  106  no       Unit is ready and clustered
   ceph-osd                  17.1.0   active      4  ceph-osd                quincy/stable  534  no       Unit is ready (2 OSD)
   ceph-radosgw              17.1.0   active      1  ceph-radosgw            quincy/stable  526  no       Unit is ready
   cinder                    20.0.0   active      1  cinder                  yoga/stable    554  no       Unit is ready
   cinder-ceph               20.0.0   active      1  cinder-ceph             yoga/stable    502  no       Unit is ready
   cinder-mysql-router       8.0.29   active      1  mysql-router            8.0/stable      26  no       Unit is ready
   dashboard-mysql-router    8.0.29   active      1  mysql-router            8.0/stable      26  no       Unit is ready
   glance                    24.0.0   active      1  glance                  yoga/stable    544  no       Unit is ready
   glance-mysql-router       8.0.29   active      1  mysql-router            8.0/stable      26  no       Unit is ready
   keystone                  21.0.0   active      1  keystone                yoga/stable    568  no       Application Ready
   keystone-mysql-router     8.0.29   active      1  mysql-router            8.0/stable      26  no       Unit is ready
   mysql-innodb-cluster      8.0.29   active      3  mysql-innodb-cluster    8.0/stable      24  no       Unit is ready: Mode: R/O, Cluster is ONLINE and can tolerate up to ONE failure.
   ncc-mysql-router          8.0.29   active      1  mysql-router            8.0/stable      26  no       Unit is ready
   neutron-api               20.0.0   active      1  neutron-api             yoga/stable    526  no       Unit is ready
   neutron-api-mysql-router  8.0.29   active      1  mysql-router            8.0/stable      26  no       Unit is ready
   neutron-api-plugin-ovn    20.0.0   active      1  neutron-api-plugin-ovn  yoga/stable     29  no       Unit is ready
   nova-cloud-controller     25.0.0   active      1  nova-cloud-controller   yoga/stable    601  no       Unit is ready
   nova-compute              25.0.0   active      3  nova-compute            yoga/stable    588  no       Unit is ready
   openstack-dashboard       22.1.0   active      1  openstack-dashboard     yoga/stable    536  no       Unit is ready
   ovn-central               22.03.0  active      3  ovn-central             22.03/stable    31  no       Unit is ready
   ovn-chassis               22.03.0  active      3  ovn-chassis             22.03/stable    46  no       Unit is ready
   placement                 7.0.0    active      1  placement               yoga/stable     49  no       Unit is ready
   placement-mysql-router    8.0.29   active      1  mysql-router            8.0/stable      26  no       Unit is ready
   rabbitmq-server           3.9.13   active      1  rabbitmq-server         3.9/stable     149  no       Unit is ready
   vault                     1.7.9    active      1  vault                   1.7/stable      68  no       Unit is ready (active: true, mlock: disabled)
   vault-mysql-router        8.0.29   active      1  mysql-router            8.0/stable      26  no       Unit is ready

   Unit                           Workload  Agent  Machine  Public address  Ports               Message
   ceph-mon/0                     active    idle   0/lxd/4  10.0.0.180                          Unit is ready and clustered
   ceph-mon/1*                    active    idle   1/lxd/4  10.0.0.182                          Unit is ready and clustered
   ceph-mon/2                     active    idle   2/lxd/5  10.0.0.181                          Unit is ready and clustered
   ceph-osd/0                     active    idle   0        10.0.0.160                          Unit is ready (2 OSD)
   ceph-osd/1*                    active    idle   1        10.0.0.159                          Unit is ready (2 OSD)
   ceph-osd/2                     active    idle   2        10.0.0.162                          Unit is ready (2 OSD)
   ceph-osd/3                     active    idle   3        10.0.0.161                          Unit is ready (2 OSD)
   ceph-radosgw/0*                active    idle   0/lxd/5  10.0.0.184      80/tcp              Unit is ready
   cinder/0*                      active    idle   1/lxd/5  10.0.0.183      8776/tcp            Unit is ready
     cinder-ceph/0*               active    idle            10.0.0.183                          Unit is ready
     cinder-mysql-router/0*       active    idle            10.0.0.183                          Unit is ready
   glance/0*                      active    idle   3/lxd/3  10.0.0.179      9292/tcp            Unit is ready
     glance-mysql-router/0*       active    idle            10.0.0.179                          Unit is ready
   keystone/0*                    active    idle   0/lxd/3  10.0.0.174      5000/tcp            Unit is ready
     keystone-mysql-router/0*     active    idle            10.0.0.174                          Unit is ready
   mysql-innodb-cluster/0*        active    idle   0/lxd/0  10.0.0.163                          Unit is ready: Mode: R/O, Cluster is ONLINE and can tolerate up to ONE failure.
   mysql-innodb-cluster/1         active    idle   1/lxd/0  10.0.0.164                          Unit is ready: Mode: R/W, Cluster is ONLINE and can tolerate up to ONE failure.
   mysql-innodb-cluster/2         active    idle   2/lxd/0  10.0.0.165                          Unit is ready: Mode: R/O, Cluster is ONLINE and can tolerate up to ONE failure.
   neutron-api/0*                 active    idle   1/lxd/3  10.0.0.173      9696/tcp            Unit is ready
     neutron-api-mysql-router/0*  active    idle            10.0.0.173                          Unit is ready
     neutron-api-plugin-ovn/0*    active    idle            10.0.0.173                          Unit is ready
   nova-cloud-controller/0*       active    idle   3/lxd/1  10.0.0.176      8774/tcp,8775/tcp   Unit is ready
     ncc-mysql-router/0*          active    idle            10.0.0.176                          Unit is ready
   nova-compute/0*                active    idle   1        10.0.0.159                          Unit is ready
     ovn-chassis/0*               active    idle            10.0.0.159                          Unit is ready
   nova-compute/1                 active    idle   2        10.0.0.162                          Unit is ready
     ovn-chassis/2                active    idle            10.0.0.162                          Unit is ready
   nova-compute/2                 active    idle   3        10.0.0.161                          Unit is ready
     ovn-chassis/1                active    idle            10.0.0.161                          Unit is ready
   openstack-dashboard/0*         active    idle   2/lxd/4  10.0.0.178      80/tcp,443/tcp      Unit is ready
     dashboard-mysql-router/0*    active    idle            10.0.0.178                          Unit is ready
   ovn-central/3                  active    idle   0/lxd/2  10.0.0.170      6641/tcp,6642/tcp   Unit is ready
   ovn-central/4                  active    idle   1/lxd/2  10.0.0.171      6641/tcp,6642/tcp   Unit is ready (northd: active)
   ovn-central/5*                 active    idle   2/lxd/2  10.0.0.172      6641/tcp,6642/tcp   Unit is ready (leader: ovnnb_db, ovnsb_db)
   placement/0*                   active    idle   3/lxd/2  10.0.0.177      8778/tcp            Unit is ready
     placement-mysql-router/0*    active    idle            10.0.0.177                          Unit is ready
   rabbitmq-server/0*             active    idle   2/lxd/3  10.0.0.175      5672/tcp,15672/tcp  Unit is ready
   vault/0*                       active    idle   3/lxd/0  10.0.0.166      8200/tcp            Unit is ready (active: true, mlock: disabled)
     vault-mysql-router/0*        active    idle            10.0.0.166                          Unit is ready

   Machine  State    DNS         Inst id              Series  AZ       Message
   0        started  10.0.0.160  node1                jammy   default  Deployed
   0/lxd/0  started  10.0.0.163  juju-df2f3d-0-lxd-0  jammy   default  Container started
   0/lxd/2  started  10.0.0.170  juju-df2f3d-0-lxd-2  jammy   default  Container started
   0/lxd/3  started  10.0.0.174  juju-df2f3d-0-lxd-3  jammy   default  Container started
   0/lxd/4  started  10.0.0.180  juju-df2f3d-0-lxd-4  jammy   default  Container started
   0/lxd/5  started  10.0.0.184  juju-df2f3d-0-lxd-5  jammy   default  Container started
   1        started  10.0.0.159  node2                jammy   default  Deployed
   1/lxd/0  started  10.0.0.164  juju-df2f3d-1-lxd-0  jammy   default  Container started
   1/lxd/2  started  10.0.0.171  juju-df2f3d-1-lxd-2  jammy   default  Container started
   1/lxd/3  started  10.0.0.173  juju-df2f3d-1-lxd-3  jammy   default  Container started
   1/lxd/4  started  10.0.0.182  juju-df2f3d-1-lxd-4  jammy   default  Container started
   1/lxd/5  started  10.0.0.183  juju-df2f3d-1-lxd-5  jammy   default  Container started
   2        started  10.0.0.162  node4                jammy   default  Deployed
   2/lxd/0  started  10.0.0.165  juju-df2f3d-2-lxd-0  jammy   default  Container started
   2/lxd/2  started  10.0.0.172  juju-df2f3d-2-lxd-2  jammy   default  Container started
   2/lxd/3  started  10.0.0.175  juju-df2f3d-2-lxd-3  jammy   default  Container started
   2/lxd/4  started  10.0.0.178  juju-df2f3d-2-lxd-4  jammy   default  Container started
   2/lxd/5  started  10.0.0.181  juju-df2f3d-2-lxd-5  jammy   default  Container started
   3        started  10.0.0.161  node3                jammy   default  Deployed
   3/lxd/0  started  10.0.0.166  juju-df2f3d-3-lxd-0  jammy   default  Container started
   3/lxd/1  started  10.0.0.176  juju-df2f3d-3-lxd-1  jammy   default  Container started
   3/lxd/2  started  10.0.0.177  juju-df2f3d-3-lxd-2  jammy   default  Container started
   3/lxd/3  started  10.0.0.179  juju-df2f3d-3-lxd-3  jammy   default  Container started

   Relation provider                      Requirer                                     Interface                       Type         Message
   ceph-mon:client                        cinder-ceph:ceph                             ceph-client                     regular
   ceph-mon:client                        glance:ceph                                  ceph-client                     regular
   ceph-mon:client                        nova-compute:ceph                            ceph-client                     regular
   ceph-mon:mon                           ceph-mon:mon                                 ceph                            peer
   ceph-mon:osd                           ceph-osd:mon                                 ceph-osd                        regular
   ceph-mon:radosgw                       ceph-radosgw:mon                             ceph-radosgw                    regular
   ceph-radosgw:cluster                   ceph-radosgw:cluster                         swift-ha                        peer
   cinder-ceph:ceph-access                nova-compute:ceph-access                     cinder-ceph-key                 regular
   cinder-ceph:storage-backend            cinder:storage-backend                       cinder-backend                  subordinate
   cinder-mysql-router:shared-db          cinder:shared-db                             mysql-shared                    subordinate
   cinder:cinder-volume-service           nova-cloud-controller:cinder-volume-service  cinder                          regular
   cinder:cluster                         cinder:cluster                               cinder-ha                       peer
   dashboard-mysql-router:shared-db       openstack-dashboard:shared-db                mysql-shared                    subordinate
   glance-mysql-router:shared-db          glance:shared-db                             mysql-shared                    subordinate
   glance:cluster                         glance:cluster                               glance-ha                       peer
   glance:image-service                   cinder:image-service                         glance                          regular
   glance:image-service                   nova-cloud-controller:image-service          glance                          regular
   glance:image-service                   nova-compute:image-service                   glance                          regular
   keystone-mysql-router:shared-db        keystone:shared-db                           mysql-shared                    subordinate
   keystone:cluster                       keystone:cluster                             keystone-ha                     peer
   keystone:identity-service              cinder:identity-service                      keystone                        regular
   keystone:identity-service              glance:identity-service                      keystone                        regular
   keystone:identity-service              neutron-api:identity-service                 keystone                        regular
   keystone:identity-service              nova-cloud-controller:identity-service       keystone                        regular
   keystone:identity-service              openstack-dashboard:identity-service         keystone                        regular
   keystone:identity-service              placement:identity-service                   keystone                        regular
   mysql-innodb-cluster:cluster           mysql-innodb-cluster:cluster                 mysql-innodb-cluster            peer
   mysql-innodb-cluster:coordinator       mysql-innodb-cluster:coordinator             coordinator                     peer
   mysql-innodb-cluster:db-router         cinder-mysql-router:db-router                mysql-router                    regular
   mysql-innodb-cluster:db-router         dashboard-mysql-router:db-router             mysql-router                    regular
   mysql-innodb-cluster:db-router         glance-mysql-router:db-router                mysql-router                    regular
   mysql-innodb-cluster:db-router         keystone-mysql-router:db-router              mysql-router                    regular
   mysql-innodb-cluster:db-router         ncc-mysql-router:db-router                   mysql-router                    regular
   mysql-innodb-cluster:db-router         neutron-api-mysql-router:db-router           mysql-router                    regular
   mysql-innodb-cluster:db-router         placement-mysql-router:db-router             mysql-router                    regular
   mysql-innodb-cluster:db-router         vault-mysql-router:db-router                 mysql-router                    regular
   ncc-mysql-router:shared-db             nova-cloud-controller:shared-db              mysql-shared                    subordinate
   neutron-api-mysql-router:shared-db     neutron-api:shared-db                        mysql-shared                    subordinate
   neutron-api-plugin-ovn:neutron-plugin  neutron-api:neutron-plugin-api-subordinate   neutron-plugin-api-subordinate  subordinate
   neutron-api:cluster                    neutron-api:cluster                          neutron-api-ha                  peer
   neutron-api:neutron-api                nova-cloud-controller:neutron-api            neutron-api                     regular
   nova-cloud-controller:cluster          nova-cloud-controller:cluster                nova-ha                         peer
   nova-compute:cloud-compute             nova-cloud-controller:cloud-compute          nova-compute                    regular
   nova-compute:compute-peer              nova-compute:compute-peer                    nova                            peer
   openstack-dashboard:cluster            openstack-dashboard:cluster                  openstack-dashboard-ha          peer
   ovn-central:ovsdb                      ovn-chassis:ovsdb                            ovsdb                           regular
   ovn-central:ovsdb-cms                  neutron-api-plugin-ovn:ovsdb-cms             ovsdb-cms                       regular
   ovn-central:ovsdb-peer                 ovn-central:ovsdb-peer                       ovsdb-cluster                   peer
   ovn-chassis:nova-compute               nova-compute:neutron-plugin                  neutron-plugin                  subordinate
   placement-mysql-router:shared-db       placement:shared-db                          mysql-shared                    subordinate
   placement:cluster                      placement:cluster                            openstack-ha                    peer
   placement:placement                    nova-cloud-controller:placement              placement                       regular
   rabbitmq-server:amqp                   cinder:amqp                                  rabbitmq                        regular
   rabbitmq-server:amqp                   neutron-api:amqp                             rabbitmq                        regular
   rabbitmq-server:amqp                   nova-cloud-controller:amqp                   rabbitmq                        regular
   rabbitmq-server:amqp                   nova-compute:amqp                            rabbitmq                        regular
   rabbitmq-server:cluster                rabbitmq-server:cluster                      rabbitmq-ha                     peer
   vault-mysql-router:shared-db           vault:shared-db                              mysql-shared                    subordinate
   vault:certificates                     cinder:certificates                          tls-certificates                regular
   vault:certificates                     glance:certificates                          tls-certificates                regular
   vault:certificates                     keystone:certificates                        tls-certificates                regular
   vault:certificates                     mysql-innodb-cluster:certificates            tls-certificates                regular
   vault:certificates                     neutron-api-plugin-ovn:certificates          tls-certificates                regular
   vault:certificates                     neutron-api:certificates                     tls-certificates                regular
   vault:certificates                     nova-cloud-controller:certificates           tls-certificates                regular
   vault:certificates                     openstack-dashboard:certificates             tls-certificates                regular
   vault:certificates                     ovn-central:certificates                     tls-certificates                regular
   vault:certificates                     ovn-chassis:certificates                     tls-certificates                regular
   vault:certificates                     placement:certificates                       tls-certificates                regular
   vault:cluster                          vault:cluster                                vault-ha                        peer
