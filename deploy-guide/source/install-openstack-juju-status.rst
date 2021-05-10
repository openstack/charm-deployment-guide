:orphan:

.. _install_openstack_juju_status:

===============
OpenStack cloud
===============

The below :command:`juju status --relations` output represents the cloud
installed from the instructions given on the :doc:`Install OpenStack
<install-openstack>` page.

.. code-block:: console

   Model      Controller  Cloud/Region      Version  SLA          Timestamp
   openstack  maas-one    maas-one/default  2.9.0    unsupported  01:35:20Z

   App                       Version  Status  Scale  Charm                   Store       Channel  Rev  OS      Message
   ceph-mon                  16.2.0   active      3  ceph-mon                charmstore  stable   464  ubuntu  Unit is ready and clustered
   ceph-osd                  16.2.0   active      4  ceph-osd                charmstore  stable   489  ubuntu  Unit is ready (2 OSD)
   ceph-radosgw              16.2.0   active      1  ceph-radosgw            charmstore  stable   398  ubuntu  Unit is ready
   cinder                    18.0.0   active      1  cinder                  charmstore  stable   436  ubuntu  Unit is ready
   cinder-ceph               18.0.0   active      1  cinder-ceph             charmstore  stable   352  ubuntu  Unit is ready
   cinder-mysql-router       8.0.23   active      1  mysql-router            charmstore  stable    48  ubuntu  Unit is ready
   dashboard-mysql-router    8.0.23   active      1  mysql-router            charmstore  stable    48  ubuntu  Unit is ready
   glance                    22.0.0   active      1  glance                  charmstore  stable   450  ubuntu  Unit is ready
   glance-mysql-router       8.0.23   active      1  mysql-router            charmstore  stable    48  ubuntu  Unit is ready
   keystone                  19.0.0   active      1  keystone                charmstore  stable   542  ubuntu  Application Ready
   keystone-mysql-router     8.0.23   active      1  mysql-router            charmstore  stable    48  ubuntu  Unit is ready
   mysql-innodb-cluster      8.0.23   active      3  mysql-innodb-cluster    charmstore  stable    74  ubuntu  Unit is ready: Mode: R/W, Cluster is ONLINE and can tolerate up to ONE failure.
   ncc-mysql-router          8.0.23   active      1  mysql-router            charmstore  stable    48  ubuntu  Unit is ready
   neutron-api               18.0.0   active      1  neutron-api             charmstore  stable   471  ubuntu  Unit is ready
   neutron-api-mysql-router  8.0.23   active      1  mysql-router            charmstore  stable    48  ubuntu  Unit is ready
   neutron-api-plugin-ovn    18.0.0   active      1  neutron-api-plugin-ovn  charmstore  stable    40  ubuntu  Unit is ready
   nova-cloud-controller     23.0.0   active      1  nova-cloud-controller   charmstore  stable   521  ubuntu  Unit is ready
   nova-compute              23.0.0   active      3  nova-compute            charmstore  stable   539  ubuntu  Unit is ready
   ntp                       3.5      active      4  ntp                     charmstore  stable    45  ubuntu  chrony: Ready
   openstack-dashboard       19.2.0   active      1  openstack-dashboard     charmstore  stable   505  ubuntu  Unit is ready
   ovn-central               20.12.0  active      3  ovn-central             charmstore  stable    51  ubuntu  Unit is ready (leader: ovnnb_db, ovnsb_db northd: active)
   ovn-chassis               20.12.0  active      3  ovn-chassis             charmstore  stable    63  ubuntu  Unit is ready
   placement                 5.0.0    active      1  placement               charmstore  stable    47  ubuntu  Unit is ready
   placement-mysql-router    8.0.23   active      1  mysql-router            charmstore  stable    48  ubuntu  Unit is ready
   rabbitmq-server           3.8.2    active      1  rabbitmq-server         charmstore  stable   406  ubuntu  Unit is ready
   vault                     1.5.4    active      1  vault                   charmstore  stable   141  ubuntu  Unit is ready (active: true, mlock: disabled)
   vault-mysql-router        8.0.23   active      1  mysql-router            charmstore  stable    48  ubuntu  Unit is ready

   Unit                           Workload  Agent  Machine  Public address  Ports              Message
   ceph-mon/0                     active    idle   0/lxd/3  10.0.0.170                         Unit is ready and clustered
   ceph-mon/1                     active    idle   1/lxd/3  10.0.0.169                         Unit is ready and clustered
   ceph-mon/2*                    active    idle   2/lxd/4  10.0.0.168                         Unit is ready and clustered
   ceph-osd/0*                    active    idle   0        10.0.0.150                         Unit is ready (2 OSD)
     ntp/3                        active    idle            10.0.0.150      123/udp            chrony: Ready
   ceph-osd/1                     active    idle   1        10.0.0.151                         Unit is ready (2 OSD)
     ntp/2                        active    idle            10.0.0.151      123/udp            chrony: Ready
   ceph-osd/2                     active    idle   2        10.0.0.152                         Unit is ready (2 OSD)
     ntp/1                        active    idle            10.0.0.152      123/udp            chrony: Ready
   ceph-osd/3                     active    idle   3        10.0.0.153                         Unit is ready (2 OSD)
     ntp/0*                       active    idle            10.0.0.153      123/udp            chrony: Ready
   ceph-radosgw/0*                active    idle   0/lxd/4  10.0.0.172      80/tcp             Unit is ready
   cinder/0*                      active    idle   1/lxd/4  10.0.0.171      8776/tcp           Unit is ready
     cinder-ceph/0*               active    idle            10.0.0.171                         Unit is ready
     cinder-mysql-router/0*       active    idle            10.0.0.171                         Unit is ready
   glance/0*                      active    idle   3/lxd/3  10.0.0.167      9292/tcp           Unit is ready
     glance-mysql-router/0*       active    idle            10.0.0.167                         Unit is ready
   keystone/0*                    active    idle   0/lxd/2  10.0.0.162      5000/tcp           Unit is ready
     keystone-mysql-router/0*     active    idle            10.0.0.162                         Unit is ready
   mysql-innodb-cluster/0*        active    idle   0/lxd/0  10.0.0.154                         Unit is ready: Mode: R/W, Cluster is ONLINE and can tolerate up to ONE failure.
   mysql-innodb-cluster/1         active    idle   1/lxd/0  10.0.0.155                         Unit is ready: Mode: R/O, Cluster is ONLINE and can tolerate up to ONE failure.
   mysql-innodb-cluster/2         active    idle   2/lxd/0  10.0.0.156                         Unit is ready: Mode: R/O, Cluster is ONLINE and can tolerate up to ONE failure.
   neutron-api/0*                 active    idle   1/lxd/2  10.0.0.161      9696/tcp           Unit is ready
     neutron-api-mysql-router/0*  active    idle            10.0.0.161                         Unit is ready
     neutron-api-plugin-ovn/0*    active    idle            10.0.0.161                         Unit is ready
   nova-cloud-controller/0*       active    idle   3/lxd/1  10.0.0.164      8774/tcp,8775/tcp  Unit is ready
     ncc-mysql-router/0*          active    idle            10.0.0.164                         Unit is ready
   nova-compute/0*                active    idle   1        10.0.0.151                         Unit is ready
     ovn-chassis/2                active    idle            10.0.0.151                         Unit is ready
   nova-compute/1                 active    idle   2        10.0.0.152                         Unit is ready
     ovn-chassis/0*               active    idle            10.0.0.152                         Unit is ready
   nova-compute/2                 active    idle   3        10.0.0.153                         Unit is ready
     ovn-chassis/1                active    idle            10.0.0.153                         Unit is ready
   openstack-dashboard/0*         active    idle   2/lxd/3  10.0.0.166      80/tcp,443/tcp     Unit is ready
     dashboard-mysql-router/0*    active    idle            10.0.0.166                         Unit is ready
   ovn-central/0*                 active    idle   0/lxd/1  10.0.0.158      6641/tcp,6642/tcp  Unit is ready (leader: ovnnb_db, ovnsb_db northd: active)
   ovn-central/1                  active    idle   1/lxd/1  10.0.0.159      6641/tcp,6642/tcp  Unit is ready
   ovn-central/2                  active    idle   2/lxd/1  10.0.0.160      6641/tcp,6642/tcp  Unit is ready
   placement/0*                   active    idle   3/lxd/2  10.0.0.165      8778/tcp           Unit is ready
     placement-mysql-router/0*    active    idle            10.0.0.165                         Unit is ready
   rabbitmq-server/0*             active    idle   2/lxd/2  10.0.0.163      5672/tcp           Unit is ready
   vault/0*                       active    idle   3/lxd/0  10.0.0.157      8200/tcp           Unit is ready (active: true, mlock: disabled)
     vault-mysql-router/0*        active    idle            10.0.0.157                         Unit is ready

   Machine  State    DNS         Inst id              Series  AZ       Message
   0        started  10.0.0.150  node4                focal   default  Deployed
   0/lxd/0  started  10.0.0.154  juju-3d942c-0-lxd-0  focal   default  Container started
   0/lxd/1  started  10.0.0.158  juju-3d942c-0-lxd-1  focal   default  Container started
   0/lxd/2  started  10.0.0.162  juju-3d942c-0-lxd-2  focal   default  Container started
   0/lxd/3  started  10.0.0.170  juju-3d942c-0-lxd-3  focal   default  Container started
   0/lxd/4  started  10.0.0.172  juju-3d942c-0-lxd-4  focal   default  Container started
   1        started  10.0.0.151  node1                focal   default  Deployed
   1/lxd/0  started  10.0.0.155  juju-3d942c-1-lxd-0  focal   default  Container started
   1/lxd/1  started  10.0.0.159  juju-3d942c-1-lxd-1  focal   default  Container started
   1/lxd/2  started  10.0.0.161  juju-3d942c-1-lxd-2  focal   default  Container started
   1/lxd/3  started  10.0.0.169  juju-3d942c-1-lxd-3  focal   default  Container started
   1/lxd/4  started  10.0.0.171  juju-3d942c-1-lxd-4  focal   default  Container started
   2        started  10.0.0.152  node2                focal   default  Deployed
   2/lxd/0  started  10.0.0.156  juju-3d942c-2-lxd-0  focal   default  Container started
   2/lxd/1  started  10.0.0.160  juju-3d942c-2-lxd-1  focal   default  Container started
   2/lxd/2  started  10.0.0.163  juju-3d942c-2-lxd-2  focal   default  Container started
   2/lxd/3  started  10.0.0.166  juju-3d942c-2-lxd-3  focal   default  Container started
   2/lxd/4  started  10.0.0.168  juju-3d942c-2-lxd-4  focal   default  Container started
   3        started  10.0.0.153  node3                focal   default  Deployed
   3/lxd/0  started  10.0.0.157  juju-3d942c-3-lxd-0  focal   default  Container started
   3/lxd/1  started  10.0.0.164  juju-3d942c-3-lxd-1  focal   default  Container started
   3/lxd/2  started  10.0.0.165  juju-3d942c-3-lxd-2  focal   default  Container started
   3/lxd/3  started  10.0.0.167  juju-3d942c-3-lxd-3  focal   default  Container started

   Relation provider                      Requirer                                     Interface                       Type         Message
   ceph-mon:client                        cinder-ceph:ceph                             ceph-client                     regular
   ceph-mon:client                        glance:ceph                                  ceph-client                     regular
   ceph-mon:client                        nova-compute:ceph                            ceph-client                     regular
   ceph-mon:mon                           ceph-mon:mon                                 ceph                            peer
   ceph-mon:osd                           ceph-osd:mon                                 ceph-osd                        regular
   ceph-mon:radosgw                       ceph-radosgw:mon                             ceph-radosgw                    regular
   ceph-osd:juju-info                     ntp:juju-info                                juju-info                       subordinate
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
   ntp:ntp-peers                          ntp:ntp-peers                                ntp                             peer
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
