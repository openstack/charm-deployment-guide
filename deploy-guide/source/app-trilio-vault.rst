=======================================
Appendix U: TrilioVault Data Protection
=======================================

Overview
--------

As of the 20.05 release, the OpenStack charms support deployment of
TrilioVault, a data protection solution that integrates with
OpenStack.

TrilioVault allows end-users to independently backup and restore
point-in-time snapshots of their own workloads.

.. note::

   TrilioVault is not part of the OpenStack project and is a commercially
   supported propriety product.  For more details visit `trilio.io`_.

Prerequisites
-------------

 - Ubuntu 18.04 LTS
 - OpenStack Queens, Stein or Train
 - NFS server for storage of snapshots

.. note::

   TrilioVault requires a license for operation. For more details visit
   `trilio.io`_.

.. warning::

   TrilioVault only supports Ubuntu 18.04 LTS.

Deployment
----------

The TrilioVault solution consists of three core services:

 - TrilioVault Workload Manager: this service provides the main API
   for TrilioVault and is used to create and manage snapshots
   and restores.
 - TrilioVault Data Mover API: this service provides the API for
   managing the TrilioVault Data Mover services and is used
   by the TrilioVault Workload Manager.
 - TrilioVault Data Mover: deployed alongside Nova Compute services
   this service is responsible for managing the snapshot process for
   instances running on the OpenStack Cloud.

TrilioVault also provides an OpenStack Dashboard plugin providing a Web UI
for end users and cloud administrators.

TrilioVault may be added to an OpenStack cloud using the following example
overlay bundle:

.. code-block:: yaml

   series: bionic
   applications:
     trilio-data-mover:
       charm: cs:~openstack-charmers/trilio-data-mover
       options:
         triliovault-pkg-source: "deb [trusted=yes] https://apt.fury.io/triliovault-python3/ /"
     trilio-dm-api:
       charm: cs:~openstack-charmers/trilio-dm-api
       num_units: 1
       options:
         openstack-origin: cloud:bionic-train
         triliovault-pkg-source: "deb [trusted=yes] https://apt.fury.io/triliovault-python3/ /"
     trilio-horizon-plugin:
       charm: cs:~openstack-charmers-next/trilio-horizon-plugin
       options:
         triliovault-pkg-source: "deb [trusted=yes] https://apt.fury.io/triliovault-python3/ /"
     trilio-wlm:
       charm: cs:~openstack-charmers/trilio-wlm
       num_units: 1
       options:
         openstack-origin: cloud:bionic-train
         triliovault-pkg-source: "deb [trusted=yes] https://apt.fury.io/triliovault-python3/ /"
   relations:
     - - trilio-horizon-plugin:dashboard-plugin
       - openstack-dashboard:dashboard-plugin
     - - trilio-dm-api:identity-service
       - keystone:identity-service
     - - trilio-dm-api:shared-db
       - mysql:shared-db
     - - trilio-dm-api:amqp
       - rabbitmq-server:amqp
     - - trilio-data-mover:amqp
       - rabbitmq-server:amqp
     - - trilio-data-mover:juju-info
       - nova-compute:juju-info
     - - trilio-wlm:shared-db
       - mysql:shared-db
     - - trilio-wlm:amqp
       - rabbitmq-server:amqp
     - - trilio-wlm:identity-service
       - keystone:identity-service

NFS
---

After deployment completes the TrilioVault Data Mover and Workload Manager
applications will be in a blocked state (see :command:`juju status`). Both
must be configured with a valid NFS share on the NFS server used in the
deployment via configuration:

.. code-block:: none

   juju config trilio-wlm nfs-shares=10.40.3.20:/srv/triliovault
   juju config trilio-data-mover nfs-shares=10.40.3.20:/srv/triliovault

Both services must be configured with the same NFS share.

Authorisation
-------------

The TrilioVault service account must be granted the authorisation to access
resources from across users and projects to perform backups. This will require
passing the cloud admin password (setup by the keystone application) to the
``create-cloud-admin-trust`` action:

.. code-block:: none

   juju run-action trilio-wlm/leader create-cloud-admin-trust password=cloudadminpassword

Licensing
---------

Finally, the TrilioVault deployment must be licensed. This is completed by
uploading the license file from Trilio as a resource and then executing the
``create-license`` action:

.. code-block:: none

   juju attach trilio-wlm license=mycorp_tv.lic
   juju run-action trilio-wlm/leader create-license

The trilio-wlm and trilio-data-mover applications should be in the 'active'
state and ready for use.

.. LINKS
.. _trilio.io: https://www.trilio.io/triliovault/openstack/
