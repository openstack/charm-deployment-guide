:orphan:

.. _install_openstack_juju_status:

===============
OpenStack cloud
===============

The below :command:`juju status --relations` output represents the cloud
installed from the instructions given on the :doc:`Install OpenStack
<install-openstack>` page.

.. code-block:: ini

   Model      Controller       Cloud/Region    Version  SLA          Timestamp
   openstack  maas-controller  mymaas/default  2.7.0    unsupported  21:02:58Z

   App                    Version  Status  Scale  Charm                  Store       Rev  OS      Notes
   ceph-mon               14.2.2   active      3  ceph-mon               jujucharms   44  ubuntu
   ceph-osd               14.2.2   active      4  ceph-osd               jujucharms  294  ubuntu
   cinder                 15.0.0   active      1  cinder                 jujucharms  297  ubuntu
   cinder-ceph            15.0.0   active      1  cinder-ceph            jujucharms  251  ubuntu
   glance                 19.0.0   active      1  glance                 jujucharms  291  ubuntu
   keystone               16.0.0   active      1  keystone               jujucharms  309  ubuntu
   mysql                  5.7.20   active      1  percona-cluster        jujucharms  281  ubuntu
   neutron-api            15.0.0   active      1  neutron-api            jujucharms  282  ubuntu
   neutron-gateway        15.0.0   active      1  neutron-gateway        jujucharms  276  ubuntu
   neutron-openvswitch    15.0.0   active      3  neutron-openvswitch    jujucharms  269  ubuntu
   nova-cloud-controller  20.0.0   active      1  nova-cloud-controller  jujucharms  339  ubuntu
   nova-compute           20.0.0   active      3  nova-compute           jujucharms  309  ubuntu
   ntp                    3.2      active      4  ntp                    jujucharms   36  ubuntu
   openstack-dashboard    13.0.2   active      1  openstack-dashboard    jujucharms  297  ubuntu
   placement              2.0.0    active      1  placement              jujucharms    0  ubuntu
   rabbitmq-server        3.6.10   active      1  rabbitmq-server        jujucharms   97  ubuntu

   Unit                      Workload  Agent  Machine  Public address  Ports              Message
   ceph-mon/0*               active    idle   1/lxd/1  10.0.0.21                          Unit is ready and clustered
   ceph-mon/1                active    idle   2/lxd/3  10.0.0.22                          Unit is ready and clustered
   ceph-mon/2                active    idle   3/lxd/1  10.0.0.23                          Unit is ready and clustered
   ceph-osd/0                active    idle   0        10.0.0.27                          Unit is ready (1 OSD)
     ntp/2                   active    idle            10.0.0.27       123/udp            chrony: Ready
   ceph-osd/1                active    idle   1        10.0.0.26                          Unit is ready (1 OSD)
     ntp/1                   active    idle            10.0.0.26       123/udp            chrony: Ready
   ceph-osd/2*               active    idle   2        10.0.0.28                          Unit is ready (1 OSD)
     ntp/0*                  active    idle            10.0.0.28       123/udp            chrony: Ready
   ceph-osd/3                active    idle   3        10.0.0.255                         Unit is ready (1 OSD)
     ntp/3                   active    idle            10.0.0.255      123/udp            chrony: Ready
   cinder/0*                 active    idle   1/lxd/2  10.0.0.24       8776/tcp           Unit is ready
     cinder-ceph/0*          active    idle            10.0.0.24                          Unit is ready
   glance/0*                 active    idle   2/lxd/2  10.0.0.20       9292/tcp           Unit is ready
   keystone/0*               active    idle   3/lxd/0  10.0.0.29       5000/tcp           Unit is ready
   mysql/0*                  active    idle   0/lxd/0  10.0.0.8        3306/tcp           Unit is ready
   neutron-api/0*            active    idle   1/lxd/0  10.0.0.7        9696/tcp           Unit is ready
   neutron-gateway/0*        active    idle   0        10.0.0.27                          Unit is ready
   nova-cloud-controller/0*  active    idle   2/lxd/0  10.0.0.10       8774/tcp,8775/tcp  Unit is ready
   nova-compute/0            active    idle   1        10.0.0.26                          Unit is ready
     neutron-openvswitch/2   active    idle            10.0.0.26                          Unit is ready
   nova-compute/1*           active    idle   2        10.0.0.28                          Unit is ready
     neutron-openvswitch/0*  active    idle            10.0.0.28                          Unit is ready
   nova-compute/2            active    idle   3        10.0.0.255                         Unit is ready
     neutron-openvswitch/1   active    idle            10.0.0.255                         Unit is ready
   openstack-dashboard/0*    active    idle   3/lxd/2  10.0.0.14       80/tcp,443/tcp     Unit is ready
   placement/0*              active    idle   2/lxd/1  10.0.0.11       8778/tcp           Unit is ready
   rabbitmq-server/0*        active    idle   0/lxd/1  10.0.0.9        5672/tcp           Unit is ready

   Machine  State    DNS         Inst id              Series  AZ       Message
   0        started  10.0.0.27   virt-node-03         bionic  default  Deployed
   0/lxd/0  started  10.0.0.8    juju-218d08-0-lxd-0  bionic  default  Container started
   0/lxd/1  started  10.0.0.9    juju-218d08-0-lxd-1  bionic  default  Container started
   1        started  10.0.0.26   virt-node-02         bionic  default  Deployed
   1/lxd/0  started  10.0.0.7    juju-218d08-1-lxd-0  bionic  default  Container started
   1/lxd/1  started  10.0.0.21   juju-218d08-1-lxd-1  bionic  default  Container started
   1/lxd/2  started  10.0.0.24   juju-218d08-1-lxd-2  bionic  default  Container started
   2        started  10.0.0.28   virt-node-04         bionic  default  Deployed
   2/lxd/0  started  10.0.0.10   juju-218d08-2-lxd-0  bionic  default  Container started
   2/lxd/1  started  10.0.0.11   juju-218d08-2-lxd-1  bionic  default  Container started
   2/lxd/2  started  10.0.0.20   juju-218d08-2-lxd-2  bionic  default  Container started
   2/lxd/3  started  10.0.0.22   juju-218d08-2-lxd-3  bionic  default  Container started
   3        started  10.0.0.255  virt-node-01         bionic  default  Deployed
   3/lxd/0  started  10.0.0.29   juju-218d08-3-lxd-0  bionic  default  Container started
   3/lxd/1  started  10.0.0.23   juju-218d08-3-lxd-1  bionic  default  Container started
   3/lxd/2  started  10.0.0.14   juju-218d08-3-lxd-2  bionic  default  Container started

   Relation provider                        Requirer                                       Interface               Type         Message
   ceph-mon:client                          cinder-ceph:ceph                               ceph-client             regular
   ceph-mon:client                          glance:ceph                                    ceph-client             regular
   ceph-mon:client                          nova-compute:ceph                              ceph-client             regular
   ceph-mon:mon                             ceph-mon:mon                                   ceph                    peer
   ceph-mon:osd                             ceph-osd:mon                                   ceph-osd                regular
   ceph-osd:juju-info                       ntp:juju-info                                  juju-info               subordinate
   cinder-ceph:storage-backend              cinder:storage-backend                         cinder-backend          subordinate
   cinder:cinder-volume-service             nova-cloud-controller:cinder-volume-service    cinder                  regular
   cinder:cluster                           cinder:cluster                                 cinder-ha               peer
   glance:cluster                           glance:cluster                                 glance-ha               peer
   glance:image-service                     cinder:image-service                           glance                  regular
   glance:image-service                     nova-cloud-controller:image-service            glance                  regular
   glance:image-service                     nova-compute:image-service                     glance                  regular
   keystone:cluster                         keystone:cluster                               keystone-ha             peer
   keystone:identity-service                cinder:identity-service                        keystone                regular
   keystone:identity-service                glance:identity-service                        keystone                regular
   keystone:identity-service                neutron-api:identity-service                   keystone                regular
   keystone:identity-service                nova-cloud-controller:identity-service         keystone                regular
   keystone:identity-service                openstack-dashboard:identity-service           keystone                regular
   keystone:identity-service                placement:identity-service                     keystone                regular
   mysql:cluster                            mysql:cluster                                  percona-cluster         peer
   mysql:shared-db                          cinder:shared-db                               mysql-shared            regular
   mysql:shared-db                          glance:shared-db                               mysql-shared            regular
   mysql:shared-db                          keystone:shared-db                             mysql-shared            regular
   mysql:shared-db                          neutron-api:shared-db                          mysql-shared            regular
   mysql:shared-db                          nova-cloud-controller:shared-db                mysql-shared            regular
   mysql:shared-db                          placement:shared-db                            mysql-shared            regular
   neutron-api:cluster                      neutron-api:cluster                            neutron-api-ha          peer
   neutron-api:neutron-api                  nova-cloud-controller:neutron-api              neutron-api             regular
   neutron-api:neutron-plugin-api           neutron-gateway:neutron-plugin-api             neutron-plugin-api      regular
   neutron-api:neutron-plugin-api           neutron-openvswitch:neutron-plugin-api         neutron-plugin-api      regular
   neutron-gateway:cluster                  neutron-gateway:cluster                        quantum-gateway-ha      peer
   neutron-gateway:quantum-network-service  nova-cloud-controller:quantum-network-service  quantum                 regular
   neutron-openvswitch:neutron-plugin       nova-compute:neutron-plugin                    neutron-plugin          subordinate
   nova-cloud-controller:cluster            nova-cloud-controller:cluster                  nova-ha                 peer
   nova-compute:cloud-compute               nova-cloud-controller:cloud-compute            nova-compute            regular
   nova-compute:compute-peer                nova-compute:compute-peer                      nova                    peer
   ntp:ntp-peers                            ntp:ntp-peers                                  ntp                     peer
   openstack-dashboard:cluster              openstack-dashboard:cluster                    openstack-dashboard-ha  peer
   placement:cluster                        placement:cluster                              openstack-ha            peer
   placement:placement                      nova-cloud-controller:placement                placement               regular
   rabbitmq-server:amqp                     cinder:amqp                                    rabbitmq                regular
   rabbitmq-server:amqp                     glance:amqp                                    rabbitmq                regular
   rabbitmq-server:amqp                     neutron-api:amqp                               rabbitmq                regular
   rabbitmq-server:amqp                     neutron-gateway:amqp                           rabbitmq                regular
   rabbitmq-server:amqp                     neutron-openvswitch:amqp                       rabbitmq                regular
   rabbitmq-server:amqp                     nova-cloud-controller:amqp                     rabbitmq                regular
   rabbitmq-server:amqp                     nova-compute:amqp                              rabbitmq                regular
   rabbitmq-server:cluster                  rabbitmq-server:cluster                        rabbitmq-ha             peer
