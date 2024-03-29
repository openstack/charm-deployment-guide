=================
Install OpenStack
=================

In the :doc:`previous section <install-juju>`, we installed Juju and created a
Juju controller and model. We are now going to use Juju to install OpenStack
itself.

Despite the length of this page, only two distinct Juju commands will be
employed: :command:`juju deploy` and :command:`juju integrate`. You may want
to review these pertinent sections of the Juju documentation before continuing:

* `Deploying applications`_
* `Deploying to specific machines`_
* `Managing integrations`_

This page will show how to install a minimal non-HA OpenStack cloud. See the
:doc:`cg:admin/ha` page in the Charm Guide for guidance on that subject.

Specification of software versions
----------------------------------

The cloud deployment involves two levels of software:

* charms (e.g. keystone charm)
* charm payload (e.g. Keystone service)

A charm's software version (its revision) is distinct from its payload version.
The charm's channel both expresses the payload version and incorporates
(assumes) an Ubuntu version for the underlying cloud node.

For example, channel '2023.2/stable' installs an OpenStack 2023.2 payload (e.g.
Keystone 23.0.0) that is only compatible with Ubuntu 22.04 LTS (Jammy).

See the Charm Guide for more information:

* :doc:`cg:project/charm-delivery` for charm channels
* :doc:`cg:concepts/software-sources` for charm payload

Optionally, the payload version can be manually specified (see the above
Software sources page).

OpenStack release
-----------------

OpenStack 2023.2 (Bobcat) will be deployed atop Ubuntu 22.04 LTS (Jammy)
cloud nodes. In order to achieve this, charm channels appropriate for the
chosen OpenStack release will be used.

See :ref:`cg:perform_the_upgrade` in the Charm Guide for more details on cloud
archive releases and how they are used when upgrading OpenStack.

.. important::

   The chosen OpenStack release may impact the installation and configuration
   instructions. **This guide assumes that OpenStack 2023.2 is being
   deployed.**

Installation progress
---------------------

There are many moving parts involved in a charmed OpenStack install. During
much of the process there will be components that have not yet been satisfied,
which will cause error-like messages to be displayed in the output of the
:command:`juju status` command. Do not be alarmed. Indeed, these are
opportunities to learn about the interdependencies of the various pieces of
software. Messages such as **Missing relation** and **blocked** will vanish
once the appropriate applications and relations (integrations) have been added
and processed.

.. tip::

   One convenient way to monitor the installation progress is to have command
   :command:`watch -n 5 -c juju status --color` running in a separate terminal.

Deploy OpenStack
----------------

Assuming you have precisely followed the instructions on the :doc:`Install Juju
<install-juju>` page, you should now have a Juju controller called
'maas-controller' and an empty Juju model called 'openstack'. Change to that
context now:

.. code-block:: none

   juju switch maas-controller:openstack

In the following sections, the various OpenStack components will be added to
the 'openstack' model. Each application will be installed from the online
`Charmhub`_ and many will have configuration options specified via a YAML file.

.. note::

   You do not need to wait for a Juju command to complete before issuing
   further ones. However, it can be very instructive to see the effect one
   command has on the current state of the cloud.

Ceph OSD
~~~~~~~~

The ceph-osd application is deployed to four nodes with the `ceph-osd`_ charm.

The names of the block devices backing the OSDs is dependent upon the hardware
on the MAAS nodes. All possible devices (across all the nodes) that are to be
used for Ceph storage should be included in the value for the ``osd-devices``
option (space-separated). Here, we'll be using the same devices on each node:
``/dev/sda``, ``/dev/sdb``, ``/dev/sdc``, and ``/dev/sdd``. File
``ceph-osd.yaml`` contains the configuration:

.. code-block:: yaml

   ceph-osd:
     osd-devices: /dev/sda /dev/sdb /dev/sdc /dev/sdd

To deploy the application we'll make use of the 'compute' tag that we placed on
each of these nodes on the :doc:`Install MAAS <install-maas>` page:

.. code-block:: none

   juju deploy -n 4 --channel reef/stable --config ceph-osd.yaml --constraints tags=compute ceph-osd

If a message from a ceph-osd unit like "Non-pristine devices detected" appears
in the output of :command:`juju status` you will need to use actions
``zap-disk`` and ``add-disk`` that come with the ceph-osd charm. The
``zap-disk`` action is destructive in nature. Only use it if you want to purge
the disk of all data and signatures for use by Ceph.

.. note::

   Since ceph-osd was deployed on four nodes and there are only four nodes
   available in this environment, the usage of the 'compute' tag is not
   strictly necessary. A tag can help if there are a surplus of nodes however.

Nova Compute
~~~~~~~~~~~~

The nova-compute application is deployed to three nodes with the
`nova-compute`_ charm. File ``nova-compute.yaml`` contains the configuration:

.. code-block:: yaml

   nova-compute:
     config-flags: default_ephemeral_format=ext4
     enable-live-migration: true
     enable-resize: true
     migration-auth-type: ssh
     virt-type: qemu

The nodes must be targeted by machine ID since there are no more free Juju
machines (MAAS nodes) available. This means we're placing multiple services on
our nodes. We've chosen machines 1, 2, and 3. To deploy:

.. code-block:: none

   juju deploy -n 3 --to 1,2,3 --channel 2023.2/stable --config nova-compute.yaml nova-compute

.. note::

   The 'nova-compute' charm is designed to support one image format type per
   application at any given time. Changing format (see charm option
   ``libvirt-image-backend``) while existing instances are using the prior
   format will require manual image conversion for each instance. See bug `LP
   #1826888`_.

MySQL InnoDB Cluster
~~~~~~~~~~~~~~~~~~~~

MySQL InnoDB Cluster always requires at least three database units. The
mysql-innodb-cluster application is deployed to three nodes with the
`mysql-innodb-cluster`_ charm. They will be containerised on machines 0, 1, and
2. To deploy:

.. code-block:: none

   juju deploy -n 3 --to lxd:0,lxd:1,lxd:2 --channel 8.0/stable mysql-innodb-cluster

Vault
~~~~~

Vault is necessary for managing the TLS certificates that will enable encrypted
communication between cloud applications. The vault application will be
containerised on machine 3 with the `vault`_ charm. To deploy:

.. code-block:: none

   juju deploy --to lxd:3 --channel 1.8/stable vault

This is the first application to be joined with the cloud database that was set
up in the previous section. The process is:

#. create an application-specific instance of mysql-router with the
   `mysql-router`_ subordinate charm
#. add a relation between the mysql-router instance and the database
#. add a relation between the mysql-router instance and the application

The combination of steps 2 and 3 joins the application to the cloud database.

Here are the corresponding commands for Vault:

.. code-block:: none

   juju deploy --channel 8.0/stable mysql-router vault-mysql-router
   juju integrate vault-mysql-router:db-router mysql-innodb-cluster:db-router
   juju integrate vault-mysql-router:shared-db vault:shared-db

Vault must now be initialised and unsealed. The vault charm will also need to
be authorised to carry out certain tasks. These steps are covered in the `vault
charm documentation`_. Perform them now.

Provide Vault with a CA certificate so it can issue certificates to cloud API
services. This is covered on the :ref:`Managing TLS certificates
<add_ca_certificate>` page. Do this now.

Once the above is completed the Unit section output to command :command:`juju
status` should look similar to this:

.. code-block:: console

   Unit                     Workload  Agent  Machine  Public address  Ports     Message
   ceph-osd/0               blocked   idle   0        10.246.115.11             Missing relation: monitor
   ceph-osd/1*              blocked   idle   1        10.246.114.47             Missing relation: monitor
   ceph-osd/2               blocked   idle   2        10.246.114.17             Missing relation: monitor
   ceph-osd/3               blocked   idle   3        10.246.114.16             Missing relation: monitor
   mysql-innodb-cluster/0   active    idle   0/lxd/0  10.246.115.16             Unit is ready: Mode: R/O, Cluster is ONLINE and can tolerate up to ONE failure.
   mysql-innodb-cluster/1*  active    idle   1/lxd/0  10.246.115.14             Unit is ready: Mode: R/W, Cluster is ONLINE and can tolerate up to ONE failure.
   mysql-innodb-cluster/2   active    idle   2/lxd/0  10.246.115.15             Unit is ready: Mode: R/O, Cluster is ONLINE and can tolerate up to ONE failure.
   nova-compute/0           blocked   idle   1        10.246.114.47             Missing relations: image, messaging
   nova-compute/1           blocked   idle   2        10.246.114.17             Missing relations: messaging, image
   nova-compute/2*          blocked   idle   3        10.246.114.16             Missing relations: messaging, image
   vault/1*                 active    idle   3/lxd/1  10.246.115.17   8200/tcp  Unit is ready (active: true, mlock: disabled)
     vault-mysql-router/0*  active    idle            10.246.115.17             Unit is ready

Cloud applications are TLS-enabled via the ``vault:certificates`` relation.
Below we start with the cloud database. Although the latter has a self-signed
certificate, it is recommended to use the one signed by Vault's CA:

.. code-block:: none

   juju integrate mysql-innodb-cluster:certificates vault:certificates

.. _neutron_networking:

Neutron networking
~~~~~~~~~~~~~~~~~~

Neutron networking is implemented with four applications:

* neutron-api
* neutron-api-plugin-ovn (subordinate)
* ovn-central
* ovn-chassis (subordinate)

File ``neutron.yaml`` contains the configuration necessary (only two of them
require configuration):

.. code-block:: yaml

   ovn-chassis:
     bridge-interface-mappings: br-ex:enp1s0
     ovn-bridge-mappings: physnet1:br-ex
   neutron-api:
     neutron-security-groups: true
     flat-network-providers: physnet1

The ``bridge-interface-mappings`` setting impacts the OVN Chassis and refers to
a mapping of OVS bridge to network interface. As described in the :ref:`Create
OVS bridge <ovs_bridge>` section on the :doc:`Install MAAS <install-maas>`
page, for this example it is 'br-ex:enp1s0'.

.. note::

   To use hardware addresses (as opposed to an interface name common to all
   four nodes) the ``bridge-interface-mappings`` option can be expressed in
   this way (substitute in your own values):

   .. code-block:: yaml

      bridge-interface-mappings: >-
        br-ex:52:54:00:03:01:01
        br-ex:52:54:00:03:01:02
        br-ex:52:54:00:03:01:03
        br-ex:52:54:00:03:01:04

The ``flat-network-providers`` setting enables the Neutron flat network
provider used in this example scenario and gives it the name of 'physnet1'. The
flat network provider and its name will be referenced when we :ref:`Set up
public networking <public_networking>` on the next page.

The ``ovn-bridge-mappings`` setting maps the data-port interface to the flat
network provider.

The main OVN application is ovn-central and it requires at least three units.
They will be containerised on machines 0, 1, and 2 with the `ovn-central`_
charm. To deploy:

.. code-block:: none

   juju deploy -n 3 --to lxd:0,lxd:1,lxd:2 --channel 23.09/stable ovn-central

The neutron-api application will be containerised on machine 1 with the
`neutron-api`_ charm:

.. code-block:: none

   juju deploy --to lxd:1 --channel 2023.2/stable --config neutron.yaml neutron-api

Deploy the subordinate charm applications with the `neutron-api-plugin-ovn`_
and `ovn-chassis`_ charms:

.. code-block:: none

   juju deploy --channel 2023.2/stable neutron-api-plugin-ovn
   juju deploy --channel 23.09/stable --config neutron.yaml ovn-chassis

Add the necessary relations:

.. code-block:: none

   juju integrate neutron-api-plugin-ovn:neutron-plugin neutron-api:neutron-plugin-api-subordinate
   juju integrate neutron-api-plugin-ovn:ovsdb-cms ovn-central:ovsdb-cms
   juju integrate ovn-chassis:ovsdb ovn-central:ovsdb
   juju integrate ovn-chassis:nova-compute nova-compute:neutron-plugin
   juju integrate neutron-api:certificates vault:certificates
   juju integrate neutron-api-plugin-ovn:certificates vault:certificates
   juju integrate ovn-central:certificates vault:certificates
   juju integrate ovn-chassis:certificates vault:certificates

Join neutron-api to the cloud database:

.. code-block:: none

   juju deploy --channel 8.0/stable mysql-router neutron-api-mysql-router
   juju integrate neutron-api-mysql-router:db-router mysql-innodb-cluster:db-router
   juju integrate neutron-api-mysql-router:shared-db neutron-api:shared-db

Keystone
~~~~~~~~

The keystone application will be containerised on machine 0 with the
`keystone`_ charm. To deploy:

.. code-block:: none

   juju deploy --to lxd:0 --channel 2023.2/stable keystone

Join keystone to the cloud database:

.. code-block:: none

   juju deploy --channel 8.0/stable mysql-router keystone-mysql-router
   juju integrate keystone-mysql-router:db-router mysql-innodb-cluster:db-router
   juju integrate keystone-mysql-router:shared-db keystone:shared-db

Two additional relations can be added at this time:

.. code-block:: none

   juju integrate keystone:identity-service neutron-api:identity-service
   juju integrate keystone:certificates vault:certificates

RabbitMQ
~~~~~~~~

The rabbitmq-server application will be containerised on machine 2 with the
`rabbitmq-server`_ charm. To deploy:

.. code-block:: none

   juju deploy --to lxd:2 --channel 3.9/stable rabbitmq-server

Two relations can be added at this time:

.. code-block:: none

   juju integrate rabbitmq-server:amqp neutron-api:amqp
   juju integrate rabbitmq-server:amqp nova-compute:amqp

At this time the Unit section output to command :command:`juju status` should
look similar to this:

.. code-block:: console

   Unit                           Workload  Agent  Machine  Public address  Ports           Message
   ceph-osd/0                     blocked   idle   0        10.246.115.11                   Missing relation: monitor
   ceph-osd/1*                    blocked   idle   1        10.246.114.47                   Missing relation: monitor
   ceph-osd/2                     blocked   idle   2        10.246.114.17                   Missing relation: monitor
   ceph-osd/3                     blocked   idle   3        10.246.114.16                   Missing relation: monitor
   keystone/0*                    active    idle   0/lxd/2  10.246.114.72   5000/tcp        Unit is ready
     keystone-mysql-router/0*     active    idle            10.246.114.72                   Unit is ready
   mysql-innodb-cluster/0         active    idle   0/lxd/0  10.246.115.16                   Unit is ready: Mode: R/O, Cluster is ONLINE and can tolerate up to ONE
   failure.
   mysql-innodb-cluster/1*        active    idle   1/lxd/0  10.246.115.14                   Unit is ready: Mode: R/W, Cluster is ONLINE and can tolerate up to ONE
   failure.
   mysql-innodb-cluster/2         active    idle   2/lxd/0  10.246.115.15                   Unit is ready: Mode: R/O, Cluster is ONLINE and can tolerate up to ONE
   failure.
   neutron-api/0*                 active    idle   1/lxd/2  10.246.114.71   9696/tcp        Unit is ready
     neutron-api-mysql-router/0*  active    idle            10.246.114.71                   Unit is ready
     neutron-api-plugin-ovn/0*    active    idle            10.246.114.71                   Unit is ready
   nova-compute/0                 blocked   idle   1        10.246.114.47                   Missing relations: image
     ovn-chassis/3                active    idle            10.246.114.47                   Unit is ready
   nova-compute/1                 blocked   idle   2        10.246.114.17                   Missing relations: image
     ovn-chassis/0                active    idle            10.246.114.17                   Unit is ready
   nova-compute/2*                blocked   idle   3        10.246.114.16                   Missing relations: image
     ovn-chassis/2*               active    idle            10.246.114.16                   Unit is ready
   ovn-central/0                  active    idle   0/lxd/1  10.246.115.18   6641-6642/tcp   Unit is ready (northd: active)
   ovn-central/1                  active    idle   1/lxd/1  10.246.115.20   6641-6642/tcp   Unit is ready
   ovn-central/2*                 active    idle   2/lxd/1  10.246.115.19   6641-6642/tcp   Unit is ready (leader: ovnnb_db, ovnsb_db)
   rabbitmq-server/0*             active    idle   2/lxd/2  10.246.114.73   5672,15672/tcp  Unit is ready
   vault/1*                       active    idle   3/lxd/1  10.246.115.17   8200/tcp        Unit is ready (active: true, mlock: disabled)
     vault-mysql-router/0*        active    idle            10.246.115.17                   Unit is ready

Nova cloud controller
~~~~~~~~~~~~~~~~~~~~~

The nova-cloud-controller application, which includes nova-scheduler, nova-api,
and nova-conductor services, will be containerised on machine 3 with the
`nova-cloud-controller`_ charm. File ``ncc.yaml`` contains the configuration:

.. code-block:: yaml

   nova-cloud-controller:
     network-manager: Neutron

To deploy:

.. code-block:: none

   juju deploy --to lxd:3 --channel 2023.2/stable --config ncc.yaml nova-cloud-controller

Join nova-cloud-controller to the cloud database:

.. code-block:: none

   juju deploy --channel 8.0/stable mysql-router ncc-mysql-router
   juju integrate ncc-mysql-router:db-router mysql-innodb-cluster:db-router
   juju integrate ncc-mysql-router:shared-db nova-cloud-controller:shared-db

.. note::

   To keep :command:`juju status` output compact the expected
   ``nova-cloud-controller-mysql-router`` application name has been shortened
   to ``ncc-mysql-router``.

Five additional relations can be added at this time:

.. code-block:: none

   juju integrate nova-cloud-controller:identity-service keystone:identity-service
   juju integrate nova-cloud-controller:amqp rabbitmq-server:amqp
   juju integrate nova-cloud-controller:neutron-api neutron-api:neutron-api
   juju integrate nova-cloud-controller:cloud-compute nova-compute:cloud-compute
   juju integrate nova-cloud-controller:certificates vault:certificates

Placement
~~~~~~~~~

The placement application will be containerised on machine 3 with the
`placement`_ charm. To deploy:

.. code-block:: none

   juju deploy --to lxd:3 --channel 2023.2/stable placement

Join placement to the cloud database:

.. code-block:: none

   juju deploy --channel 8.0/stable mysql-router placement-mysql-router
   juju integrate placement-mysql-router:db-router mysql-innodb-cluster:db-router
   juju integrate placement-mysql-router:shared-db placement:shared-db

Three additional relations can be added at this time:

.. code-block:: none

   juju integrate placement:identity-service keystone:identity-service
   juju integrate placement:placement nova-cloud-controller:placement
   juju integrate placement:certificates vault:certificates

OpenStack dashboard
~~~~~~~~~~~~~~~~~~~

The openstack-dashboard application (Horizon) will be containerised on machine
2 with the `openstack-dashboard`_ charm. To deploy:

.. code-block:: none

   juju deploy --to lxd:2 --channel 2023.2/stable openstack-dashboard

Join openstack-dashboard to the cloud database:

.. code-block:: none

   juju deploy --channel 8.0/stable mysql-router dashboard-mysql-router
   juju integrate dashboard-mysql-router:db-router mysql-innodb-cluster:db-router
   juju integrate dashboard-mysql-router:shared-db openstack-dashboard:shared-db

.. note::

   To keep :command:`juju status` output compact the expected
   ``openstack-dashboard-mysql-router`` application name has been shortened to
   ``dashboard-mysql-router``.

Two additional relations are required:

.. code-block:: none

   juju integrate openstack-dashboard:identity-service keystone:identity-service
   juju integrate openstack-dashboard:certificates vault:certificates

Glance
~~~~~~

The glance application will be containerised on machine 3 with the `glance`_
charm. To deploy:

.. code-block:: none

   juju deploy --to lxd:3 --channel 2023.2/stable glance

Join glance to the cloud database:

.. code-block:: none

   juju deploy --channel 8.0/stable mysql-router glance-mysql-router
   juju integrate glance-mysql-router:db-router mysql-innodb-cluster:db-router
   juju integrate glance-mysql-router:shared-db glance:shared-db

Four additional relations can be added at this time:

.. code-block:: none

   juju integrate glance:image-service nova-cloud-controller:image-service
   juju integrate glance:image-service nova-compute:image-service
   juju integrate glance:identity-service keystone:identity-service
   juju integrate glance:certificates vault:certificates

At this time the Unit section output to command :command:`juju status` should
look similar to this:

.. code-block:: console

   Unit                           Workload  Agent  Machine  Public address  Ports           Message
   ceph-osd/0                     blocked   idle   0        10.246.115.11                   Missing relation: monitor
   ceph-osd/1*                    blocked   idle   1        10.246.114.47                   Missing relation: monitor
   ceph-osd/2                     blocked   idle   2        10.246.114.17                   Missing relation: monitor
   ceph-osd/3                     blocked   idle   3        10.246.114.16                   Missing relation: monitor
   glance/0*                      active    idle   3/lxd/4  10.246.114.77   9292/tcp        Unit is ready
     glance-mysql-router/0*       active    idle            10.246.114.77                   Unit is ready
   keystone/0*                    active    idle   0/lxd/2  10.246.114.72   5000/tcp        Unit is ready
     keystone-mysql-router/0*     active    idle            10.246.114.72                   Unit is ready
   mysql-innodb-cluster/0         active    idle   0/lxd/0  10.246.115.16                   Unit is ready: Mode: R/O, Cluster is ONLINE and can tolerate up to ONE
   failure.
   mysql-innodb-cluster/1*        active    idle   1/lxd/0  10.246.115.14                   Unit is ready: Mode: R/W, Cluster is ONLINE and can tolerate up to ONE
   failure.
   mysql-innodb-cluster/2         active    idle   2/lxd/0  10.246.115.15                   Unit is ready: Mode: R/O, Cluster is ONLINE and can tolerate up to ONE
   failure.
   neutron-api/0*                 active    idle   1/lxd/2  10.246.114.71   9696/tcp        Unit is ready
     neutron-api-mysql-router/0*  active    idle            10.246.114.71                   Unit is ready
     neutron-api-plugin-ovn/0*    active    idle            10.246.114.71                   Unit is ready
   nova-cloud-controller/0*       active    idle   3/lxd/2  10.246.114.74   8774-8775/tcp   Unit is ready
     ncc-mysql-router/0*          active    idle            10.246.114.74                   Unit is ready
   nova-compute/0                 active    idle   1        10.246.114.47                   Unit is ready
     ovn-chassis/3                active    idle            10.246.114.47                   Unit is ready
   nova-compute/1                 active    idle   2        10.246.114.17                   Unit is ready
     ovn-chassis/0                active    idle            10.246.114.17                   Unit is ready
   nova-compute/2*                active    idle   3        10.246.114.16                   Unit is ready
     ovn-chassis/2*               active    idle            10.246.114.16                   Unit is ready
   openstack-dashboard/0*         active    idle   2/lxd/3  10.246.114.76   80,443/tcp      Unit is ready
     dashboard-mysql-router/0*    active    idle            10.246.114.76                   Unit is ready
   ovn-central/0                  active    idle   0/lxd/1  10.246.115.18   6641-6642/tcp   Unit is ready (northd: active)
   ovn-central/1                  active    idle   1/lxd/1  10.246.115.20   6641-6642/tcp   Unit is ready
   ovn-central/2*                 active    idle   2/lxd/1  10.246.115.19   6641-6642/tcp   Unit is ready (leader: ovnnb_db, ovnsb_db)
   placement/0*                   active    idle   3/lxd/3  10.246.114.75   8778/tcp        Unit is ready
     placement-mysql-router/0*    active    idle            10.246.114.75                   Unit is ready
   rabbitmq-server/0*             active    idle   2/lxd/2  10.246.114.73   5672,15672/tcp  Unit is ready
   vault/1*                       active    idle   3/lxd/1  10.246.115.17   8200/tcp        Unit is ready (active: true, mlock: disabled)
     vault-mysql-router/0*        active    idle            10.246.115.17                   Unit is ready

Ceph monitor
~~~~~~~~~~~~

The ceph-mon application will be containerised on machines 0, 1, and 2 with the
`ceph-mon`_ charm. File ``ceph-mon.yaml`` contains the configuration:

.. code-block:: yaml

   ceph-mon:
     expected-osd-count: 4
     monitor-count: 3

The above informs the MON cluster that it is comprised of three nodes and that
it should expect at least four OSDs (disks).

To deploy:

.. code-block:: none

   juju deploy -n 3 --to lxd:0,lxd:1,lxd:2 --channel reef/stable --config ceph-mon.yaml ceph-mon

Three relations can be added at this time:

.. code-block:: none

   juju integrate ceph-mon:osd ceph-osd:mon
   juju integrate ceph-mon:client nova-compute:ceph
   juju integrate ceph-mon:client glance:ceph

For the above relations,

* The ``nova-compute:ceph`` relation makes Ceph the storage backend for Nova
  non-bootable disk images. The nova-compute charm option
  ``libvirt-image-backend`` must be set to 'rbd' for this to take effect.

* The ``glance:ceph`` relation makes Ceph the storage backend for Glance.

Cinder
~~~~~~

The cinder application will be containerised on machine 1 with the `cinder`_
charm. File ``cinder.yaml`` contains the configuration:

.. code-block:: yaml

   cinder:
     block-device: None
     glance-api-version: 2

To deploy:

.. code-block:: none

   juju deploy --to lxd:1 --channel 2023.2/stable --config cinder.yaml cinder

Join cinder to the cloud database:

.. code-block:: none

   juju deploy --channel 8.0/stable mysql-router cinder-mysql-router
   juju integrate cinder-mysql-router:db-router mysql-innodb-cluster:db-router
   juju integrate cinder-mysql-router:shared-db cinder:shared-db

Five additional relations can be added at this time:

.. code-block:: none

   juju integrate cinder:cinder-volume-service nova-cloud-controller:cinder-volume-service
   juju integrate cinder:identity-service keystone:identity-service
   juju integrate cinder:amqp rabbitmq-server:amqp
   juju integrate cinder:image-service glance:image-service
   juju integrate cinder:certificates vault:certificates

The above ``glance:image-service`` relation will enable Cinder to consume the
Glance API (e.g. making Cinder able to perform volume snapshots of Glance
images).

Like Glance, Cinder will use Ceph as its storage backend (hence ``block-device:
None`` in the configuration file). This will be implemented via the
`cinder-ceph`_ subordinate charm:

.. code-block:: none

   juju deploy --channel 2023.2/stable cinder-ceph

Three relations need to be added:

.. code-block:: none

   juju integrate cinder-ceph:storage-backend cinder:storage-backend
   juju integrate cinder-ceph:ceph ceph-mon:client
   juju integrate cinder-ceph:ceph-access nova-compute:ceph-access

Ceph RADOS Gateway
~~~~~~~~~~~~~~~~~~

The Ceph RADOS Gateway will be deployed to offer an S3 and Swift compatible
HTTP gateway. This is an alternative to using OpenStack Swift.

The ceph-radosgw application will be containerised on machine 0 with the
`ceph-radosgw`_ charm. To deploy:

.. code-block:: none

   juju deploy --to lxd:0 --channel reef/stable ceph-radosgw

A single relation is needed:

.. code-block:: none

   juju integrate ceph-radosgw:mon ceph-mon:radosgw

.. COMMENT (still: Feb 14, 2023)
   At the time of writing a jammy-aware ntp charm was not available.
   NTP
   ~~~

   The final component is an NTP client to keep the time on each cloud node
   synchronised. This is done with the `ntp`_ subordinate charm. To deploy:

   .. code-block:: none

      juju deploy ntp

   The below relation will add an ntp unit alongside each ceph-osd unit, and
   thus on each of the four cloud nodes:

   .. code-block:: none

      juju integrate ceph-osd:juju-info ntp:juju-info

.. _test_openstack:

Final results and next steps
----------------------------

Once all the applications have been deployed and the relations between them
have been added we need to wait for the output of :command:`juju status` to
settle. The final results should be devoid of any error-like messages. Example
output (including relations) for a successful cloud deployment is given
:ref:`here <install_openstack_juju_status>`.

You have successfully deployed OpenStack using Juju and MAAS. The next step is
to render the cloud functional for users. This will involve setting up
networks, images, and a user environment. Go to :doc:`Configure OpenStack
<configure-openstack>` now.

.. LINKS
.. _Charmhub: https://charmhub.io
.. _Deploying applications: https://juju.is/docs/juju/manage-applications#heading--deploy-an-application
.. _Deploying to specific machines: https://juju.is/docs/juju/placement-directive
.. _Managing integrations: https://juju.is/docs/juju/manage-integrations
.. _vault charm documentation: https://opendev.org/openstack/charm-vault/src/branch/stable/1.8/src/README.md#post-deployment-tasks

.. CHARMS
.. _ceph-mon: https://charmhub.io/ceph-mon
.. _ceph-osd: https://charmhub.io/ceph-osd
.. _ceph-radosgw: https://charmhub.io/ceph-radosgw
.. _cinder: https://charmhub.io/cinder
.. _cinder-ceph: https://charmhub.io/cinder-ceph
.. _glance: https://charmhub.io/glance
.. _keystone: https://charmhub.io/keystone
.. _mysql-innodb-cluster: https://charmhub.io/mysql-innodb-cluster
.. _mysql-router: https://charmhub.io/mysql-router
.. _neutron-gateway: https://charmhub.io/neutron-gateway
.. _neutron-api: https://charmhub.io/neutron-api
.. _neutron-api-plugin-ovn: https://charmhub.io/neutron-api-plugin-ovn
.. _neutron-openvswitch: https://charmhub.io/neutron-openvswitch
.. _nova-cloud-controller: https://charmhub.io/nova-cloud-controller
.. _nova-compute: https://charmhub.io/nova-compute
.. _ntp: https://charmhub.io/ntp
.. _openstack-dashboard: https://charmhub.io/openstack-dashboard
.. _ovn-central: https://charmhub.io/ovn-central
.. _ovn-chassis: https://charmhub.io/ovn-chassis
.. _placement: https://charmhub.io/placement
.. _rabbitmq-server: https://charmhub.io/rabbitmq-server
.. _vault: https://charmhub.io/vault

.. BUGS
.. _LP #1826888: https://bugs.launchpad.net/charm-deployment-guide/+bug/1826888
