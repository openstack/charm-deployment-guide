===========================
TrilioVault Data Protection
===========================

Overview
--------

`TrilioVault`_ is a data protection solution that integrates with OpenStack. It
allows end-users to backup and restore point-in-time snapshots of their own
workloads (cloud instances). TrilioVault is implemented via the `Trilio
charms`_.

.. note::

   TrilioVault is not part of the OpenStack project. It is a commercially
   supported propriety product.

Prerequisites
-------------

* Ubuntu 18.04 LTS or 20.04 LTS
* OpenStack Queens, Stein, Train or Ussuri
* an NFS server for snapshot storage
* a license (see the project's homepage)

Deployment
----------

The TrilioVault solution consists of three core services:

* TrilioVault Workload Manager: the main API - used to manage snapshots (e.g.
  creation and restores). This is implemented with the trilio-wlm charm.

* TrilioVault Data Mover: deployed alongside OpenStack Nova - responsible for
  managing the instance snapshot process at the cloud level. This is
  implemented with the trilio-data-mover charm.

* TrilioVault Data Mover API: the API for managing the TrilioVault Data Mover
  services - used by the Workload Manager. This is implemented with the
  trilio-dm-api charm.

An OpenStack Dashboard plugin is also provided, allowing for snapshot
management to take place via a Web UI. This is implemented with the
trilio-horizon-plugin charm.

An overlay bundle is used to deploy to an existing OpenStack cloud. An example
is provided below.

.. note::

   Ensure that the value for ``openstack-origin`` matches the currently
   deployed OpenStack release.

   The ``ceph-mon:client`` relation is only needed if the cloud is already
   configured to use Ceph-backed VM images (via either Cinder or Nova).

.. code-block:: yaml

   series: bionic
   applications:
     trilio-wlm:
       charm: cs:~openstack-charmers/trilio-wlm
       num_units: 1
       options:
         openstack-origin: cloud:bionic-train
         triliovault-pkg-source: 'deb [trusted=yes] https://apt.fury.io/triliodata-4-1/ /'
     trilio-data-mover:
       charm: cs:~openstack-charmers/trilio-data-mover
       options:
         triliovault-pkg-source: 'deb [trusted=yes] https://apt.fury.io/triliodata-4-1/ /'
     trilio-dm-api:
       charm: cs:~openstack-charmers/trilio-dm-api
       num_units: 1
       options:
         openstack-origin: cloud:bionic-train
         triliovault-pkg-source: 'deb [trusted=yes] https://apt.fury.io/triliodata-4-1/ /'
     trilio-horizon-plugin:
       charm: cs:~openstack-charmers/trilio-horizon-plugin
       options:
         triliovault-pkg-source: 'deb [trusted=yes] https://apt.fury.io/triliodata-4-1/ /'
   relations:
     - - trilio-horizon-plugin:dashboard-plugin
       - openstack-dashboard:dashboard-plugin
     - - trilio-dm-api:identity-service
       - keystone:identity-service
     - - trilio-dm-api:shared-db
       - percona-cluster:shared-db
     - - trilio-dm-api:amqp
       - rabbitmq-server:amqp
     - - trilio-data-mover:amqp
       - rabbitmq-server:amqp
     - - trilio-data-mover:juju-info
       - nova-compute:juju-info
     - - trilio-wlm:shared-db
       - percona-cluster:shared-db
     - - trilio-wlm:amqp
       - rabbitmq-server:amqp
     - - trilio-wlm:identity-service
       - keystone:identity-service
     - - trilio-data-mover:ceph
       - ceph-mon:client
     - - trilio-data-mover:shared-db
       - percona-cluster:shared-db

.. note::

   The trilio-wlm and trilio-dm-api charms must be deployed with
   ``openstack-origin`` >= 'cloud:bionic-train' - even for Queens deployments.
   These parts of the TrilioVault deployment are Python 3 only and have
   dependency version requirements that are only supported from Train onwards.

NFS
---

Once the deployment completes the trilio-wlm and trilio-data-mover applications
will be in a blocked state (see :command:`juju status`). To rectify this, both
applications must be configured with a valid NFS share (on the provided NFS
server). For example:

.. code-block:: none

   juju config trilio-wlm nfs-shares=10.40.3.20:/srv/triliovault
   juju config trilio-data-mover nfs-shares=10.40.3.20:/srv/triliovault

Both services must be configured with the same NFS share.

Authorisation
-------------

The TrilioVault service account must be granted the authorisation to access
resources from across users and projects to perform backups. This will involve
providing it with the cloud's admin password (set up by the keystone
application). This is done with the trilio-wlm charm's
``create-cloud-admin-trust`` action:

.. code-block:: none

   juju run-action trilio-wlm/leader create-cloud-admin-trust password=cloudadminpassword

Licensing
---------

The TrilioVault deployment must be licensed. This is done by uploading the
license file (attaching it as a charm resource) and running the trilio-wlm
charm's ``create-license`` action:

.. code-block:: none

   juju attach trilio-wlm license=mycorp_tv.lic
   juju run-action trilio-wlm/leader create-license

The trilio-wlm and trilio-data-mover applications should now be in the 'active'
state and ready for use.

.. LINKS
.. _TrilioVault: https://www.trilio.io/triliovault-for-openstack-2/
.. _Trilio charms: https://opendev.org/openstack?tab=&sort=recentupdate&q=charm-trilio
