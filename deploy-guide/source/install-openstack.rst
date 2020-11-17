=================
Install OpenStack
=================

In the :doc:`previous section <install-juju>`, we installed Juju and created a
Juju controller and model. We are now going to use Juju to install OpenStack
itself. There are two methods to choose from:

#. **By individual charm**. This method provides a solid understanding of how
   Juju works and of how OpenStack is put together. Choose this option if you
   have never installed OpenStack with Juju.

#. **By charm bundle**. This method provides an automated means to install
   OpenStack. Choose this option if you are familiar with how OpenStack is
   built with Juju.

The current page is devoted to method #1. See :doc:`Deploying OpenStack from a
bundle <install-openstack-bundle>` for method #2.

.. important::

   Irrespective of install method, once the cloud is deployed, the following
   management practices related to charm versions and machine series are
   recommended:

   #. The entire suite of charms used to manage the cloud should be upgraded to
      the latest stable charm revision before any major change is made to the
      cloud (e.g. migrating to new charms, upgrading cloud services, upgrading
      machine series). See `Charm upgrades`_ for details.

   #. The Juju machines that comprise the cloud should all be running the same
      series (e.g. 'bionic' or 'focal', but not a mix of the two). See `Series
      upgrade`_ for details.

Despite the length of this page, only three distinct Juju commands will be
employed: :command:`juju deploy`, :command:`juju add-unit`, and :command:`juju
add-relation`. You may want to review these pertinent sections of the Juju
documentation before continuing:

* `Deploying applications`_
* `Deploying to specific machines`_
* `Managing relations`_

.. TODO
   Cloud topology section goes here (modelled on openstack-base README)

This page will show how to install a minimal non-HA OpenStack cloud. See
:doc:`OpenStack high availability <app-ha>` for guidance on that subject.

OpenStack release
-----------------

.. TEMPLATE
   As the guide's :doc:`Overview <index>` section states, OpenStack Ussuri will
   be deployed atop Ubuntu 18.04 LTS (Bionic) cloud nodes. In order to achieve
   this a cloud archive release of 'cloud:bionic-train' will be used during the
   install of each OpenStack application. Note that some applications are not
   part of the OpenStack project per se and therefore do not apply
   (exceptionally, Ceph applications do use this method). Not using a more
   recent OpenStack release in this way will result in a Queens deployment
   (i.e. Queens is in the Ubuntu package archive for Bionic).

As the guide's :doc:`Overview <index>` section states, OpenStack Ussuri will be
deployed atop Ubuntu 20.04 LTS (Focal) cloud nodes. In order to achieve this
the default package archive ("distro") for the cloud nodes will be used during
the install of each OpenStack application. Note that some applications are not
part of the OpenStack project per se and therefore do not apply (exceptionally,
Ceph applications do use this method).

See :ref:`Perform the upgrade <perform_the_upgrade>` in the :doc:`OpenStack
Upgrades <app-upgrade-openstack>` appendix for more details on cloud
archive releases and how they are used when upgrading OpenStack.

.. important::

   The chosen OpenStack release may impact the installation and configuration
   instructions. **This guide assumes that OpenStack Ussuri is being
   deployed.**

Installation progress
---------------------

There are many moving parts involved in a charmed OpenStack install. During
much of the process there will be components that have not yet been satisfied,
which will cause error-like messages to be displayed in the output of the
:command:`juju status` command. Do not be alarmed. Indeed, these are
opportunities to learn about the interdependencies of the various pieces of
software. Messages such as **Missing relation** and **blocked** will vanish
once the appropriate applications and relations have been added and processed.

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
`Charm store`_ and many will have configuration options specified via a YAML
file.

.. note::

   You do not need to wait for a Juju command to complete before issuing
   further ones. However, it can be very instructive to see the effect one
   command has on the current state of the cloud.

Ceph OSD
~~~~~~~~

The ceph-osd application is deployed to four nodes with the `ceph-osd`_ charm.
The name of the block devices backing the OSDs is dependent upon the hardware
on the nodes. All possible devices across the nodes should be given as the
value for the ``osd-devices`` option (space-separated). Here, we'll be using
the same device on each cloud node: ``/dev/sdb``. File ``ceph-osd.yaml``
contains the configuration.

.. code-block:: yaml

   ceph-osd:
     osd-devices: /dev/sdb
     source: distro

To deploy the application we'll make use of the 'compute' tag we placed on each
of these nodes on the :doc:`Install MAAS <install-maas>` page.

.. code-block:: none

   juju deploy -n 4 --config ceph-osd.yaml --constraints tags=compute ceph-osd

If a message from a ceph-osd unit like "Non-pristine devices detected" appears
in the output of :command:`juju status` you will need to use actions
``zap-disk`` and ``add-disk`` that come with the 'ceph-osd' charm. The
``zap-disk`` action is destructive in nature. Only use it if you want to purge
the disk of all data and signatures for use by Ceph.

.. note::

   Since ceph-osd was deployed on four nodes and there are only four nodes
   available in this environment, the usage of the 'compute' tag is not
   strictly necessary.

Nova compute
~~~~~~~~~~~~

The nova-compute application is deployed to one node with the `nova-compute`_
charm. We'll then scale-out the application to two other machines. File
``nova-compute.yaml`` contains the configuration:

.. code-block:: yaml

   nova-compute:
     enable-live-migration: true
     enable-resize: true
     migration-auth-type: ssh
     openstack-origin: distro

The initial node must be targeted by machine since there are no more free Juju
machines (MAAS nodes) available. This means we're placing multiple services on
our nodes. We've chosen machines 1, 2, and 3:

.. code-block:: none

   juju deploy -n 3 --to 1,2,3 --config nova-compute.yaml nova-compute

.. note::

   The 'nova-compute' charm is designed to support one image format type per
   application at any given time. Changing format (see charm option
   ``libvirt-image-backend``) while existing instances are using the prior
   format will require manual image conversion for each instance. See bug `LP
   #1826888`_.

Swift storage
~~~~~~~~~~~~~

The swift-storage application is deployed to three nodes (machines 0, 2, and
3) with the `swift-storage`_ charm. File ``swift-storage.yaml`` contains the
configuration:

.. code-block:: yaml

   swift-storage:
     block-device: sdc
     overwrite: "true"
     openstack-origin: distro

This configuration points to block device ``/dev/sdc``. Adjust according to
your available hardware. In a production environment, avoid using a loopback
device.

Deploy to the three machines:

.. code-block:: none

   juju deploy -n 3 --to 0,2,3 --config swift-storage.yaml swift-storage

MySQL InnoDB Cluster
~~~~~~~~~~~~~~~~~~~~

MySQL InnoDB Cluster always requires at least three database units. They will
be containerised on machines 0, 1, and 2:

.. code-block:: none

   juju deploy -n 3 --to lxd:0,lxd:1,lxd:2 mysql-innodb-cluster

Vault
~~~~~

Vault is necessary for managing the TLS certificates that will enable encrypted
communication between cloud applications.

Deploy it in this way:

.. code-block:: none

   juju deploy --to lxd:3 vault

This is the first application to be joined with the cloud database that was set
up in the previous section. The process is:

#. create an application-specific instance of mysql-router (a subordinate)
#. add a relation between that mysql-router instance and the database
#. add a relation between the application and the mysql-router instance

The combination of steps 2 and 3 joins the application to the cloud database.

Here are the corresponding commands for Vault:

.. code-block:: none

   juju deploy mysql-router vault-mysql-router
   juju add-relation vault-mysql-router:db-router mysql-innodb-cluster:db-router
   juju add-relation vault-mysql-router:shared-db vault:shared-db

Vault now needs to be initialised and unsealed. The vault charm will also need
to be authorised to carry out certain tasks. These steps are covered on the
`Vault`_ page. Perform them now.

Once the above is completed the Unit section output to command :command:`juju
status` should look similar to this:

.. code-block:: console

   Unit                     Workload  Agent  Machine  Public address  Ports     Message
   ceph-osd/0*              blocked   idle   0        10.0.0.206                Missing relation: monitor
   ceph-osd/1               blocked   idle   1        10.0.0.208                Missing relation: monitor
   ceph-osd/2               blocked   idle   2        10.0.0.209                Missing relation: monitor
   ceph-osd/3               blocked   idle   3        10.0.0.213                Missing relation: monitor
   mysql-innodb-cluster/0*  active    idle   0/lxd/0  10.0.0.211                Unit is ready: Mode: R/W
   mysql-innodb-cluster/1   active    idle   1/lxd/0  10.0.0.212                Unit is ready: Mode: R/O
   mysql-innodb-cluster/2   active    idle   2/lxd/0  10.0.0.214                Unit is ready: Mode: R/O
   nova-compute/0*          blocked   idle   1        10.0.0.208                Missing relations: image, messaging
   nova-compute/1           blocked   idle   2        10.0.0.209                Missing relations: image, messaging
   nova-compute/2           blocked   idle   3        10.0.0.213                Missing relations: messaging, image
   swift-storage/0*         blocked   idle   0        10.0.0.206                Missing relations: proxy
   swift-storage/1          blocked   idle   2        10.0.0.209                Missing relations: proxy
   swift-storage/2          blocked   idle   3        10.0.0.213                Missing relations: proxy
   vault/0*                 active    idle   3/lxd/0  10.0.0.217      8200/tcp  Unit is ready (active: true, mlock: disabled)
     vault-mysql-router/0*  active    idle            10.0.0.217                Unit is ready

.. _neutron_networking:

Neutron networking
~~~~~~~~~~~~~~~~~~

Neutron networking is implemented with four applications:

* neutron-api
* neutron-api-plugin-ovn (subordinate)
* ovn-central
* ovn-chassis (subordinate)

File ``neutron.yaml`` contains the configuration necessary for three of them:

.. code-block:: yaml

   ovn-chassis:
     bridge-interface-mappings: br-ex:eth1
     ovn-bridge-mappings: physnet1:br-ex
   neutron-api:
     neutron-security-groups: true
     flat-network-providers: physnet1
     openstack-origin: distro
   ovn-central:
     source: distro

The ``bridge-interface-mappings`` setting refers to a network interface that
the OVN Chassis will bind to. In the above example it is 'eth1' and it should
be an unused interface. In MAAS this interface must be given an IP mode of
'Unconfigured' (see `Post-commission configuration`_ in the MAAS
documentation). All four nodes should have this interface to ensure that any
node is able to accommodate OVN Chassis.

The ``flat-network-providers`` setting enables the Neutron flat network
provider used in this example scenario and gives it the name of 'physnet1'. The
flat network provider and its name will be referenced when we :ref:`Set up
public networking <public_networking>` on the next page.

The ``ovn-bridge-mappings`` setting maps the data-port interface to the flat
network provider.

The main OVN application is ovn-central and it requires at least three units.
They will be containerised on machines 0, 1, and 2:

.. code-block:: none

   juju deploy -n 3 --to lxd:0,lxd:1,lxd:2 --config neutron.yaml ovn-central

The neutron-api application will be containerised on machine 1:

.. code-block:: none

   juju deploy --to lxd:1 --config neutron.yaml neutron-api

Deploy the subordinate charm applications:

.. code-block:: none

   juju deploy neutron-api-plugin-ovn
   juju deploy --config neutron.yaml ovn-chassis

Add the necessary relations:

.. code-block:: none

   juju add-relation neutron-api-plugin-ovn:neutron-plugin neutron-api:neutron-plugin-api-subordinate
   juju add-relation neutron-api-plugin-ovn:ovsdb-cms ovn-central:ovsdb-cms
   juju add-relation ovn-chassis:ovsdb ovn-central:ovsdb
   juju add-relation ovn-chassis:nova-compute nova-compute:neutron-plugin
   juju add-relation neutron-api:certificates vault:certificates
   juju add-relation neutron-api-plugin-ovn:certificates vault:certificates
   juju add-relation ovn-central:certificates vault:certificates
   juju add-relation ovn-chassis:certificates vault:certificates

Join neutron-api to the cloud database:

.. code-block:: none

   juju deploy mysql-router neutron-api-mysql-router
   juju add-relation neutron-api-mysql-router:db-router mysql-innodb-cluster:db-router
   juju add-relation neutron-api-mysql-router:shared-db neutron-api:shared-db

Keystone
~~~~~~~~

The keystone application will be containerised on machine 0.

To deploy:

.. code-block:: none

   juju deploy --to lxd:0 --config openstack-origin=distro keystone

Join keystone to the cloud database:

.. code-block:: none

   juju deploy mysql-router keystone-mysql-router
   juju add-relation keystone-mysql-router:db-router mysql-innodb-cluster:db-router
   juju add-relation keystone-mysql-router:shared-db keystone:shared-db

Two additional relations can be added at this time:

.. code-block:: none

   juju add-relation keystone:identity-service neutron-api:identity-service
   juju add-relation keystone:certificates vault:certificates

RabbitMQ
~~~~~~~~

The rabbitmq-server application will be containerised on machine 2 with the
`rabbitmq-server`_ charm:

.. code-block:: none

   juju deploy --to lxd:2 rabbitmq-server

Two relations can be added at this time:

.. code-block:: none

   juju add-relation rabbitmq-server:amqp neutron-api:amqp
   juju add-relation rabbitmq-server:amqp nova-compute:amqp

At this time the Unit section output to command :command:`juju status` should
look similar to this:

.. code-block:: console

   Unit                           Workload  Agent  Machine  Public address  Ports              Message
   ceph-osd/0*                    blocked   idle   0        10.0.0.206                         Missing relation: monitor
   ceph-osd/1                     blocked   idle   1        10.0.0.208                         Missing relation: monitor
   ceph-osd/2                     blocked   idle   2        10.0.0.209                         Missing relation: monitor
   ceph-osd/3                     blocked   idle   3        10.0.0.213                         Missing relation: monitor
   keystone/0*                    active    idle   0/lxd/2  10.0.0.223      5000/tcp           Unit is ready
     keystone-mysql-router/0*     active    idle            10.0.0.223                         Unit is ready
   mysql-innodb-cluster/0*        active    idle   0/lxd/0  10.0.0.211                         Unit is ready: Mode: R/W
   mysql-innodb-cluster/1         active    idle   1/lxd/0  10.0.0.212                         Unit is ready: Mode: R/O
   mysql-innodb-cluster/2         active    idle   2/lxd/0  10.0.0.214                         Unit is ready: Mode: R/O
   neutron-api/0*                 active    idle   1/lxd/2  10.0.0.220      9696/tcp           Unit is ready
     neutron-api-mysql-router/0*  active    idle            10.0.0.220                         Unit is ready
     neutron-api-plugin-ovn/0*    active    idle            10.0.0.220                         Unit is ready
   nova-compute/0*                blocked   idle   1        10.0.0.208                         Missing relations: image
     ovn-chassis/1                active    idle            10.0.0.208                         Unit is ready
   nova-compute/1                 blocked   idle   2        10.0.0.209                         Missing relations: image
     ovn-chassis/0*               active    idle            10.0.0.209                         Unit is ready
   nova-compute/2                 blocked   idle   3        10.0.0.213                         Missing relations: image
     ovn-chassis/2                active    idle            10.0.0.213                         Unit is ready
   ovn-central/0*                 active    idle   0/lxd/1  10.0.0.218      6641/tcp,6642/tcp  Unit is ready (leader: ovnnb_db, ovnsb_db northd: active)
   ovn-central/1                  active    idle   1/lxd/1  10.0.0.221      6641/tcp,6642/tcp  Unit is ready
   ovn-central/2                  active    idle   2/lxd/1  10.0.0.219      6641/tcp,6642/tcp  Unit is ready
   rabbitmq-server/0*             active    idle   2/lxd/2  10.0.0.222      5672/tcp           Unit is ready
   swift-storage/0*               blocked   idle   0        10.0.0.206                         Missing relations: proxy
   swift-storage/1                blocked   idle   2        10.0.0.209                         Missing relations: proxy
   swift-storage/2                blocked   idle   3        10.0.0.213                         Missing relations: proxy
   vault/0*                       active    idle   3/lxd/0  10.0.0.217      8200/tcp           Unit is ready (active: true, mlock: disabled)
     vault-mysql-router/0*        active    idle            10.0.0.217                         Unit is ready

Nova cloud controller
~~~~~~~~~~~~~~~~~~~~~

The nova-cloud-controller application, which includes nova-scheduler, nova-api,
and nova-conductor services, will be containerised on machine 0 with the
`nova-cloud-controller`_ charm. File ``nova-cloud-controller.yaml`` contains
the configuration:

.. code-block:: yaml

   nova-cloud-controller:
     network-manager: Neutron
     openstack-origin: distro

To deploy:

.. code-block:: none

   juju deploy --to lxd:3 --config nova-cloud-controller.yaml nova-cloud-controller

Join nova-cloud-controller to the cloud database:

.. code-block:: none

   juju deploy mysql-router ncc-mysql-router
   juju add-relation ncc-mysql-router:db-router mysql-innodb-cluster:db-router
   juju add-relation ncc-mysql-router:shared-db nova-cloud-controller:shared-db

.. note::

   To keep :command:`juju status` output compact the expected
   ``nova-cloud-controller-mysql-router`` application name has been shortened
   to ``ncc-mysql-router``.

Five additional relations can be added at this time:

.. code-block:: none

   juju add-relation nova-cloud-controller:identity-service keystone:identity-service
   juju add-relation nova-cloud-controller:amqp rabbitmq-server:amqp
   juju add-relation nova-cloud-controller:neutron-api neutron-api:neutron-api
   juju add-relation nova-cloud-controller:cloud-compute nova-compute:cloud-compute
   juju add-relation nova-cloud-controller:certificates vault:certificates

Placement
~~~~~~~~~

The placement application will be containerised on machine 2 with the
`placement`_ charm.

To deploy:

.. code-block:: none

   juju deploy --to lxd:3 --config openstack-origin=distro placement

Join placement to the cloud database:

.. code-block:: none

   juju deploy mysql-router placement-mysql-router
   juju add-relation placement-mysql-router:db-router mysql-innodb-cluster:db-router
   juju add-relation placement-mysql-router:shared-db placement:shared-db

Three additional relations can be added at this time:

.. code-block:: none

   juju add-relation placement:identity-service keystone:identity-service
   juju add-relation placement:placement nova-cloud-controller:placement
   juju add-relation placement:certificates vault:certificates

OpenStack dashboard
~~~~~~~~~~~~~~~~~~~

The openstack-dashboard application (Horizon) will be containerised on machine
1 with the `openstack-dashboard`_ charm.

To deploy:

.. code-block:: none

   juju deploy --to lxd:1 --config openstack-origin=distro openstack-dashboard

Join openstack-dashboard to the cloud database:

.. code-block:: none

   juju deploy mysql-router dashboard-mysql-router
   juju add-relation dashboard-mysql-router:db-router mysql-innodb-cluster:db-router
   juju add-relation dashboard-mysql-router:shared-db openstack-dashboard:shared-db

.. note::

   To keep :command:`juju status` output compact the expected
   ``openstack-dashboard-mysql-router`` application name has been shortened to
   ``dashboard-mysql-router``.

Two additional relations are required:

.. code-block:: none

   juju add-relation openstack-dashboard:identity-service keystone:identity-service
   juju add-relation openstack-dashboard:certificates vault:certificates

Glance
~~~~~~

The glance application will be containerised on machine 2 with the `glance`_
charm.

To deploy:

.. code-block:: none

   juju deploy --to lxd:3 --config openstack-origin=distro glance

Join glance to the cloud database:

.. code-block:: none

   juju deploy mysql-router glance-mysql-router
   juju add-relation glance-mysql-router:db-router mysql-innodb-cluster:db-router
   juju add-relation glance-mysql-router:shared-db glance:shared-db

Four additional relations can be added at this time:

.. code-block:: none

   juju add-relation glance:image-service nova-cloud-controller:image-service
   juju add-relation glance:image-service nova-compute:image-service
   juju add-relation glance:identity-service keystone:identity-service
   juju add-relation glance:certificates vault:certificates

At this time the Unit section output to command :command:`juju status` should
look similar to this:

.. code-block:: console

   Unit                           Workload  Agent  Machine  Public address  Ports              Message
   ceph-osd/0*                    blocked   idle   0        10.0.0.206                         Missing relation: monitor
   ceph-osd/1                     blocked   idle   1        10.0.0.208                         Missing relation: monitor
   ceph-osd/2                     blocked   idle   2        10.0.0.209                         Missing relation: monitor
   ceph-osd/3                     blocked   idle   3        10.0.0.213                         Missing relation: monitor
   glance/0*                      active    idle   3/lxd/3  10.0.0.224      9292/tcp           Unit is ready
     glance-mysql-router/0*       active    idle            10.0.0.224                         Unit is ready
   keystone/0*                    active    idle   0/lxd/2  10.0.0.223      5000/tcp           Unit is ready
     keystone-mysql-router/0*     active    idle            10.0.0.223                         Unit is ready
   mysql-innodb-cluster/0*        active    idle   0/lxd/0  10.0.0.211                         Unit is ready: Mode: R/W
   mysql-innodb-cluster/1         active    idle   1/lxd/0  10.0.0.212                         Unit is ready: Mode: R/O
   mysql-innodb-cluster/2         active    idle   2/lxd/0  10.0.0.214                         Unit is ready: Mode: R/O
   neutron-api/0*                 active    idle   1/lxd/2  10.0.0.220      9696/tcp           Unit is ready
     neutron-api-mysql-router/0*  active    idle            10.0.0.220                         Unit is ready
     neutron-api-plugin-ovn/0*    active    idle            10.0.0.220                         Unit is ready
   nova-cloud-controller/0*       active    idle   3/lxd/1  10.0.0.216      8774/tcp,8775/tcp  Unit is ready
     ncc-mysql-router/0*          active    idle            10.0.0.216                         Unit is ready
   nova-compute/0*                active    idle   1        10.0.0.208                         Unit is ready
     ovn-chassis/1                active    idle            10.0.0.208                         Unit is ready
   nova-compute/1                 active    idle   2        10.0.0.209                         Unit is ready
     ovn-chassis/0*               active    idle            10.0.0.209                         Unit is ready
   nova-compute/2                 active    idle   3        10.0.0.213                         Unit is ready
     ovn-chassis/2                active    idle            10.0.0.213                         Unit is ready
   openstack-dashboard/0*         active    idle   1/lxd/3  10.0.0.210      80/tcp,443/tcp     Unit is ready
     dashboard-mysql-router/0*    active    idle            10.0.0.210                         Unit is ready
   ovn-central/0*                 active    idle   0/lxd/1  10.0.0.218      6641/tcp,6642/tcp  Unit is ready (leader: ovnnb_db, ovnsb_db northd: active)
   ovn-central/1                  active    idle   1/lxd/1  10.0.0.221      6641/tcp,6642/tcp  Unit is ready
   ovn-central/2                  active    idle   2/lxd/1  10.0.0.219      6641/tcp,6642/tcp  Unit is ready
   placement/0*                   active    idle   3/lxd/2  10.0.0.215      8778/tcp           Unit is ready
     placement-mysql-router/0*    active    idle            10.0.0.215                         Unit is ready
   rabbitmq-server/0*             active    idle   2/lxd/2  10.0.0.222      5672/tcp           Unit is ready
   swift-storage/0*               blocked   idle   0        10.0.0.206                         Missing relations: proxy
   swift-storage/1                blocked   idle   2        10.0.0.209                         Missing relations: proxy
   swift-storage/2                blocked   idle   3        10.0.0.213                         Missing relations: proxy
   vault/0*                       active    idle   3/lxd/0  10.0.0.217      8200/tcp           Unit is ready (active: true, mlock: disabled)
     vault-mysql-router/0*        active    idle            10.0.0.217                         Unit is ready

Ceph monitor
~~~~~~~~~~~~

The ceph-mon application will be containerised on machines 0, 1, and 2 with the
`ceph-mon`_ charm.

To deploy:

.. code-block:: none

   juju deploy -n 3 --to lxd:0,lxd:1,lxd:2 --config source=distro ceph-mon

Three relations can be added at this time:

.. code-block:: none

   juju add-relation ceph-mon:osd ceph-osd:mon
   juju add-relation ceph-mon:client nova-compute:ceph
   juju add-relation ceph-mon:client glance:ceph

For the above relations,

* The nova-compute:ceph relation makes Ceph the storage backend for Nova
  non-bootable disk images. The nova-compute charm option
  ``libvirt-image-backend`` must be set to 'rbd' for this to take effect.

* The glance:ceph relation makes Ceph the storage backend for Glance.

Cinder
~~~~~~

The cinder application will be containerised on machine 1 with the `cinder`_
charm. File ``cinder.yaml`` contains the configuration:

.. code-block:: yaml

   cinder:
     glance-api-version: 2
     block-device: None
     openstack-origin: distro

To deploy:

.. code-block:: none

   juju deploy --to lxd:1 --config cinder.yaml cinder

Join cinder to the cloud database:

.. code-block:: none

   juju deploy mysql-router cinder-mysql-router
   juju add-relation cinder-mysql-router:db-router mysql-innodb-cluster:db-router
   juju add-relation cinder-mysql-router:shared-db cinder:shared-db

Four additional relations can be added at this time:

.. code-block:: none

   juju add-relation cinder:cinder-volume-service nova-cloud-controller:cinder-volume-service
   juju add-relation cinder:identity-service keystone:identity-service
   juju add-relation cinder:amqp rabbitmq-server:amqp
   juju add-relation cinder:image-service glance:image-service

The above glance:image-service relation will enable Cinder to consume the
Glance API (e.g. making Cinder able to perform volume snapshots of Glance
images).

Like Glance, Cinder will use Ceph as its storage backend (hence ``block-device:
None`` in the configuration file). This will be implemented via the
`cinder-ceph`_ subordinate charm:

.. code-block:: none

   juju deploy cinder-ceph

Four relations need to be added:

.. code-block:: none

   juju add-relation cinder-ceph:storage-backend cinder:storage-backend
   juju add-relation cinder-ceph:ceph ceph-mon:client
   juju add-relation cinder-ceph:ceph-access nova-compute:ceph-access
   juju add-relation cinder:certificates vault:certificates

Swift proxy
~~~~~~~~~~~

The swift-proxy application will be containerised on machine 3 with the
`swift-proxy`_ charm. File ``swift-proxy.yaml`` contains the configuration:

.. code-block:: yaml

   swift-proxy:
     zone-assignment: auto
     swift-hash: "<uuid>"

Swift proxy needs to be supplied with a unique identifier (UUID). Generate one
with the :command:`uuid -v 4` command (you may need to first install the
``uuid`` deb package) and insert it into the file.

To deploy:

.. code-block:: none

   juju deploy --to lxd:3 --config swift-proxy.yaml swift-proxy

Two relations are needed:

.. code-block:: none

   juju add-relation swift-proxy:swift-storage swift-storage:swift-storage
   juju add-relation swift-proxy:identity-service keystone:identity-service

NTP
~~~

The final component is an NTP client to keep the time on each cloud node
synchronised. This is done with the `ntp`_ subordinate charm:

.. code-block:: none

   juju deploy ntp

The below relation will add an ntp unit alongside each ceph-osd unit, and
thus on each of the four cloud nodes:

.. code-block:: none

   juju add-relation ceph-osd:juju-info ntp:juju-info

.. _test_openstack:

Final results and dashboard access
----------------------------------

Once all the applications have been deployed and the relations between them
have been added we need to wait for the output of :command:`juju status` to
settle. The final results should be devoid of any error-like messages. Example
output (including relations) for a successful cloud deployment is given
:ref:`here <install_openstack_juju_status>`.

One milestone in the deployment of OpenStack is the first login to the Horizon
dashboard. You will need its IP address and the admin password.

Obtain the address in this way:

.. code-block:: none

   juju status --format=yaml openstack-dashboard | grep public-address | awk '{print $2}' | head -1

The password is queried from Keystone:

.. code-block:: none

   juju run --unit keystone/0 leader-get admin_passwd

In this example, the address is '10.0.0.210' and the password is
'kohy6shoh3diWav5'.

The dashboard URL then becomes:

**http://10.0.0.210/horizon**

And the credentials are:

| Domain: **admin_domain**
| User Name: **admin**
| Password: **kohy6shoh3diWav5**
|

Once logged in you should see something like this:

.. figure:: ./media/install-openstack_horizon.png
   :scale: 70%
   :alt: Horizon dashboard

To enable instance console access from within Horizon:

.. code-block:: none

   juju config nova-cloud-controller console-access-protocol=novnc

Next steps
----------

You have successfully deployed OpenStack using both Juju and MAAS. The next
step is to render the cloud functional for users. This will involve setting up
networks, images, and a user environment. Go to :doc:`Configure OpenStack
<configure-openstack>` now.

.. LINKS
.. _OpenStack Charms: https://docs.openstack.org/charm-guide/latest/openstack-charms.html
.. _Charm upgrades: app-upgrade-openstack#charm-upgrades
.. _Series upgrade: app-series-upgrade
.. _Charm store: https://jaas.ai/store
.. _Post-commission configuration: https://maas.io/docs/commission-nodes#heading--post-commission-configuration
.. _Deploying applications: https://jaas.ai/docs/deploying-applications
.. _Deploying to specific machines: https://jaas.ai/docs/deploying-advanced-applications#heading--deploying-to-specific-machines
.. _Managing relations: https://jaas.ai/docs/relations
.. _Vault: app-vault.html

.. CHARMS
.. _ceph-mon: https://jaas.ai/ceph-mon
.. _ceph-osd: https://jaas.ai/ceph-osd
.. _cinder: https://jaas.ai/cinder
.. _cinder-ceph: https://jaas.ai/cinder-ceph
.. _glance: https://jaas.ai/glance
.. _keystone: https://jaas.ai/keystone
.. _neutron-gateway: https://jaas.ai/neutron-gateway
.. _neutron-api: https://jaas.ai/neutron-api
.. _neutron-openvswitch: https://jaas.ai/neutron-openvswitch
.. _nova-cloud-controller: https://jaas.ai/nova-cloud-controller
.. _nova-compute: https://jaas.ai/nova-compute
.. _ntp: https://jaas.ai/ntp
.. _openstack-dashboard: https://jaas.ai/openstack-dashboard
.. _percona-cluster: https://jaas.ai/percona-cluster
.. _placement: https://jaas.ai/placement
.. _rabbitmq-server: https://jaas.ai/rabbitmq-server
.. _swift-proxy: https://jaas.ai/swift-proxy
.. _swift-storage: https://jaas.ai/swift-storage

.. BUGS
.. _LP #1826888: https://bugs.launchpad.net/charm-deployment-guide/+bug/1826888
