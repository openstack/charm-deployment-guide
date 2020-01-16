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
      series (e.g. 'xenial' or 'bionic', but not a mix of the two). See `Series
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

OpenStack release
-----------------

As the guide's :doc:`Overview <index>` section states, OpenStack Train will be
deployed atop Ubuntu 18.04 LTS (Bionic) cloud nodes. In order to achieve this a
"cloud archive pocket" of 'cloud:bionic-train' will be used during the install
of each OpenStack application. Note that some applications are not part of the
OpenStack project per se and therefore do not apply (exceptionally, Ceph
applications do use this method). Not using a more recent OpenStack release in
this way will result in a Queens deployment (i.e. Queens is in the Ubuntu
package archive for Bionic).

See :ref:`Perform the upgrade <perform_the_upgrade>` in the :doc:`OpenStack
Upgrades <app-upgrade-openstack>` appendix for more details on the cloud
archive pocket and how it is used when upgrading OpenStack.

.. important::

   The chosen OpenStack release may impact the installation and configuration
   instructions. **This guide assumes that OpenStack Train is being deployed.**

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
`Charm store`_ and each will typically have configuration options specified via
its own YAML file.

.. note::

   You do not need to wait for a Juju command to complete before issuing
   further ones. However, it can be very instructive to see the effect one
   command has on the current state of the cloud.

Ceph OSD
~~~~~~~~

The ceph-osd application is deployed to four nodes with the `ceph-osd`_ charm.
The name of the block devices backing the OSDs is dependent upon the hardware
on the nodes. Here, we'll be using the same second drive on each cloud node:
``/dev/sdb``. File ``ceph-osd.yaml`` contains the configuration. If your
devices are not identical across the nodes you will need separate files (or
stipulate them on the command line):

.. code-block:: yaml

   ceph-osd:
     osd-devices: /dev/sdb
     source: cloud:bionic-train

To deploy the application we'll make use of the 'compute' tag we placed on each
of these nodes on the :doc:`Install MAAS <install-maas>` page.

.. code-block:: none

   juju deploy --constraints tags=compute --config ceph-osd.yaml -n 4 ceph-osd

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
``compute.yaml`` contains the configuration:

.. code-block:: yaml

   nova-compute:
     enable-live-migration: true
     enable-resize: true
     migration-auth-type: ssh
     openstack-origin=cloud:bionic-train

The initial node must be targeted by machine since there are no more free Juju
machines (MAAS nodes) available. This means we're placing multiple services on
our nodes. We've chosen machine 1:

.. code-block:: none

   juju deploy --to 1 --config compute.yaml nova-compute

Now scale-out to machines 2 and 3:

.. code-block:: none

   juju add-unit --to 2 nova-compute
   juju add-unit --to 3 nova-compute

.. note::

   The 'nova-compute' charm is designed to support one image format type per
   application at any given time. Changing format (see charm option
   ``libvirt-image-backend``) while existing instances are using the prior
   format will require manual image conversion for each instance. See bug `LP
   #1826888`_.

Swift storage
~~~~~~~~~~~~~

The swift-storage application is deployed to one node (machine 0) with the
`swift-storage`_ charm and then scaled-out to three other machines. File
``swift-storage.yaml`` contains the configuration:

.. code-block:: yaml

   swift-storage:
     block-device: sdc
     overwrite: "true"
     openstack-origin=cloud:bionic-train

This configuration points to block device ``/dev/sdc``. Adjust according to
your available hardware. In a production environment, avoid using a loopback
device.

Here are the four deploy commands for the four machines:

.. code-block:: none

   juju deploy --to 0 --config swift-storage.yaml swift-storage
   juju add-unit --to 1 swift-storage
   juju add-unit --to 2 swift-storage
   juju add-unit --to 3 swift-storage

.. _neutron_networking:

Neutron networking
~~~~~~~~~~~~~~~~~~

Neutron networking is implemented with three applications:

* neutron-gateway
* neutron-api
* neutron-openvswitch

File ``neutron.yaml`` contains the configuration for them:

.. code-block:: yaml

   neutron-gateway:
     data-port: br-ex:eth1
     bridge-mappings: physnet1:br-ex
     openstack-origin=cloud:bionic-train
   neutron-api:
     neutron-security-groups: true
     flat-network-providers: physnet1
     openstack-origin=cloud:bionic-train
   neutron-openvswitch:
     openstack-origin=cloud:bionic-train

The ``data-port`` setting refers to a network interface that Neutron Gateway
will bind to. In the above example it is 'eth1' and it should be an unused
interface. In MAAS this interface must be given an *IP mode* of 'Unconfigured'
(see `Post-commission configuration`_ in the MAAS documentation). Set all four
nodes in this way to ensure that any node is able to accommodate Neutron
Gateway.

The ``flat-network-providers`` setting enables the Neutron flat network
provider used in this example scenario and gives it the name of 'physnet1'. The
flat network provider and its name will be referenced when we :ref:`Set up
public networking <public_networking>` on the next page.

The ``bridge-mappings`` setting maps the data-port interface to the flat
network provider.

The neutron-gateway application will be deployed directly on machine 0:

.. code-block:: none

   juju deploy --to 0 --config neutron.yaml neutron-gateway

The neutron-api application will be deployed as a container on machine 1:

.. code-block:: none

   juju deploy --to lxd:1 --config neutron.yaml neutron-api

The neutron-openvswitch application will be deployed by means of a subordinate
charm (it will be installed on a machine once its relation is added):

.. code-block:: none

   juju deploy neutron-openvswitch --config neutron.yaml

Three relations need to be added:

.. code-block:: none

   juju add-relation neutron-api:neutron-plugin-api neutron-gateway:neutron-plugin-api
   juju add-relation neutron-api:neutron-plugin-api neutron-openvswitch:neutron-plugin-api
   juju add-relation neutron-openvswitch:neutron-plugin nova-compute:neutron-plugin

Percona cluster
~~~~~~~~~~~~~~~

The Percona XtraDB cluster is the OpenStack database of choice. The
percona-cluster application is deployed as a single LXD container on machine 0
with the `percona-cluster`_ charm. File ``mysql.yaml`` contains the
configuration:

.. code-block:: yaml

   mysql:
     max-connections: 20000

To deploy Percona while giving it an application name of 'mysql':

.. code-block:: none

   juju deploy --to lxd:0 --config mysql.yaml percona-cluster mysql

Only a single relation is needed:

.. code-block:: none

   juju add-relation neutron-api:shared-db mysql:shared-db

Keystone
~~~~~~~~

The keystone application is deployed as a single LXD container on machine 3.
File ``keystone.yaml`` contains the configuration:

.. code-block:: yaml

   keystone:
     openstack-origin=cloud:bionic-train

To deploy:

.. code-block:: none

   juju deploy --to lxd:3 --config keystone.yaml keystone

Then add these two relations:

.. code-block:: none

   juju add-relation keystone:shared-db mysql:shared-db
   juju add-relation keystone:identity-service neutron-api:identity-service

RabbitMQ
~~~~~~~~

The rabbitmq-server application is deployed as a single LXD container on
machine 0 with the `rabbitmq-server`_ charm. No additional configuration is
required. To deploy:

.. code-block:: none

   juju deploy --to lxd:0 rabbitmq-server

Four relations are needed:

.. code-block:: none

   juju add-relation rabbitmq-server:amqp neutron-api:amqp
   juju add-relation rabbitmq-server:amqp neutron-openvswitch:amqp
   juju add-relation rabbitmq-server:amqp nova-compute:amqp
   juju add-relation rabbitmq-server:amqp neutron-gateway:amqp

Nova cloud controller
~~~~~~~~~~~~~~~~~~~~~

The nova-cloud-controller application, which includes nova-scheduler, nova-api,
and nova-conductor services, is deployed as a single LXD container on machine 2
with the `nova-cloud-controller`_ charm. File ``controller.yaml`` contains the
configuration:

.. code-block:: yaml

   nova-cloud-controller:
     network-manager: Neutron
     openstack-origin=cloud:bionic-train

To deploy:

.. code-block:: none

   juju deploy --to lxd:2 --config controller.yaml nova-cloud-controller

Relations need to be added for six applications:

.. code-block:: none

   juju add-relation nova-cloud-controller:shared-db mysql:shared-db
   juju add-relation nova-cloud-controller:identity-service keystone:identity-service
   juju add-relation nova-cloud-controller:amqp rabbitmq-server:amqp
   juju add-relation nova-cloud-controller:quantum-network-service neutron-gateway:quantum-network-service
   juju add-relation nova-cloud-controller:neutron-api neutron-api:neutron-api
   juju add-relation nova-cloud-controller:cloud-compute nova-compute:cloud-compute

Placement
~~~~~~~~~

The placement application is deployed as a single LXD container on machine 2
with the `placement`_ charm. File ``placement.yaml`` contains the
configuration:

.. code-block:: yaml

   placement:
     openstack-origin=cloud:bionic-train

To deploy:

.. code-block:: none

   juju deploy --to lxd:2 --config placement.yaml placement

Relations need to be added for three applications:

.. code-block:: none

   juju add-relation placement:shared-db mysql:shared-db
   juju add-relation placement:identity-service keystone:identity-service
   juju add-relation placement:placement nova-cloud-controller:placement

OpenStack dashboard
~~~~~~~~~~~~~~~~~~~

The openstack-dashboard application (Horizon) is deployed as a single LXD
container on machine 3 with the `openstack-dashboard`_ charm. File
``dashboard.yaml`` contains the configuration:

.. code-block:: yaml

   openstack-dashboard:
     openstack-origin=cloud:bionic-train

To deploy:

.. code-block:: none

   juju deploy --to lxd:3 --config dashboard.yaml openstack-dashboard

A single relation is required:

.. code-block:: none

   juju add-relation openstack-dashboard:identity-service keystone:identity-service

Glance
~~~~~~

The glance application is deployed as a single container on machine 2 with the
`glance`_ charm. File ``glance.yaml`` contains the configuration:

.. code-block:: yaml

   openstack-dashboard:
     openstack-origin=cloud:bionic-train

To deploy:

.. code-block:: none

   juju deploy --to lxd:2 --config glance.yaml glance

Five relations are needed:

.. code-block:: none

   juju add-relation glance:image-service nova-cloud-controller:image-service
   juju add-relation glance:image-service nova-compute:image-service
   juju add-relation glance:shared-db mysql:shared-db
   juju add-relation glance:identity-service keystone:identity-service
   juju add-relation glance:amqp rabbitmq-server:amqp

Ceph monitor
~~~~~~~~~~~~

The ceph-mon application is deployed as a container on machines 1, 2, and 3
with the `ceph-mon`_ charm. File ``ceph-mon.yaml`` contains the configuration:

.. code-block:: yaml

   ceph-mon:
     source: cloud:bionic-train

To deploy:

.. code-block:: none

   juju deploy --to lxd:1 --config ceph-mon.yaml ceph-mon
   juju add-unit --to lxd:2 ceph-mon
   juju add-unit --to lxd:3 ceph-mon

Three relations are needed:

.. code-block:: none

   juju add-relation ceph-mon:osd ceph-osd:mon
   juju add-relation ceph-mon:client nova-compute:ceph
   juju add-relation ceph-mon:client glance:ceph

The last relation makes Ceph the backend for Glance.

Cinder
~~~~~~

The cinder application is deployed to a container on machine 1 with the
`cinder`_ charm. File ``cinder.yaml`` contains the configuration:

.. code-block:: yaml

   cinder:
     glance-api-version: 2
     block-device: None
     openstack-origin=cloud:bionic-train

To deploy:

.. code-block:: none

   juju deploy --to lxd:1 --config cinder.yaml cinder

Relations need to be added for five applications:

.. code-block:: none

   juju add-relation cinder:cinder-volume-service nova-cloud-controller:cinder-volume-service
   juju add-relation cinder:shared-db mysql:shared-db
   juju add-relation cinder:identity-service keystone:identity-service
   juju add-relation cinder:amqp rabbitmq-server:amqp
   juju add-relation cinder:image-service glance:image-service

In addition, like Glance, Cinder will use Ceph as its backend. This will be
implemented via the `cinder-ceph`_ subordinate charm:

.. code-block:: none

   juju deploy cinder-ceph

A relation is needed for both Cinder and Ceph:

.. code-block:: none

   juju add-relation cinder-ceph:storage-backend cinder:storage-backend
   juju add-relation cinder-ceph:ceph ceph-mon:client

Swift proxy
~~~~~~~~~~~

The swift-proxy application is deployed to a container on machine 0 with the
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

   juju deploy --to lxd:0 --config swift-proxy.yaml swift-proxy

Two relations are needed:

.. code-block:: none

   juju add-relation swift-proxy:swift-storage swift-storage:swift-storage
   juju add-relation swift-proxy:identity-service keystone:identity-service

NTP
~~~

The final component needed is an NTP client to keep everything synchronised.
This is done with the `ntp`_ subordinate charm:

.. code-block:: none

   juju deploy ntp

This single relation will add an ntp unit alongside each of the four ceph-osd
units:

.. code-block:: none

   juju add-relation ceph-osd:juju-info ntp:juju-info

.. _test_openstack:

Final results and dashboard access
----------------------------------

Once all the applications have been deployed and the relations between them
have been added we need to wait for the output of :command:`juju status` to
settle. The final results should be devoid of any error-like messages. If your
terminal supports colours then you should see only green (not amber nor red) .
Example (monochrome) output for a successful cloud deployment is given
:ref:`here <install_openstack_juju_status>`.

One milestone in the deployment of OpenStack is the first login to the Horizon
dashboard. You will need its IP address and the admin password.

Obtain the address in this way:

.. code-block:: none

   juju status --format=yaml openstack-dashboard | grep public-address | awk '{print $2}'

The password is queried from Keystone:

.. code-block:: none

   juju run --unit keystone/0 leader-get admin_passwd

In this example, the address is '10.0.0.14' and the password is
'kohy6shoh3diWav5'.

The dashboard URL then becomes:

**http://10.0.0.14/horizon**

And the credentials are:

| Domain: **admin_domain**
| User Name: **admin**
| Password: **kohy6shoh3diWav5**
|

Once logged in you should see something like this:

.. figure:: ./media/install-openstack_horizon.png
   :alt: Horizon dashboard

Next steps
----------

You have successfully deployed OpenStack using both Juju and MAAS. The next
step is to render the cloud functional for users. This will involve setting up
networks, images, and a user environment.

.. LINKS
.. _OpenStack Charms: https://docs.openstack.org/charm-guide/latest/openstack-charms.html
.. _Charm upgrades: app-upgrade-openstack#charm-upgrades
.. _Series upgrade: app-series-upgrade
.. _Charm store: https://jaas.ai/store
.. _Post-commission configuration: https://maas.io/docs/commission-nodes#heading--post-commission-configuration
.. _Deploying applications: https://jaas.ai/docs/deploying-applications
.. _Deploying to specific machines: https://jaas.ai/docs/deploying-advanced-applications#heading--deploying-to-specific-machines
.. _Managing relations: https://jaas.ai/docs/relations

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
