:orphan:

==============
Various issues
==============

This page documents various issues (software limitations/bugs) that may apply
to a Charmed OpenStack cloud. These are still-valid issues that have arisen
during the development cycles of past OpenStack Charms releases. The most
recently discovered issues are documented in the `Release Notes`_ of the latest
version of the OpenStack Charms.

The items on this page are distinct from those found on the following pages:

* the dedicated `Upgrade issues`_ page
* the `Special charm procedures`_ page

Lack of FQDN for containers on physical MAAS nodes may affect running services
------------------------------------------------------------------------------

When Juju deploys to a LXD container on a physical MAAS node, the container is
not informed of its FQDN. The services running in the container will therefore
be unable to determine the FQDN on initial deploy and on reboot.

Adverse effects are service dependent. This issue is tracked in bug `LP
#1896630`_ in an OVN and Octavia context. Several workarounds are documented in
the bug.

Barbican DB migration
---------------------

With Focal Ussuri, running command ``barbican-manage db upgrade`` against a
barbican application that is backed by a MySQL InnoDB Cluster will lead to a
failure (see bug `LP #1899104`_). This was discovered while resolving bug `LP
#1827690`_.

The package bug only affects Focal Ussuri and is not present in Victoria, nor
is it present when using (Bionic) Percona Cluster as the back-end DB.

Ceph iSCSI on Ubuntu 20.10
--------------------------

The ceph-iscsi charm can't be deployed on Ubuntu 20.10 (Groovy) due to a Python
library issue. See bug `LP #1904199`_ for details.

Adding Glance storage backends
------------------------------

When a storage backend is added to Glance a service restart may be necessary in
order for the new backend to be registered. This issue is tracked in bug `LP
#1914819`_.

OVS to OVN migration procedure on Ubuntu 20.10
----------------------------------------------

When performed on Ubuntu 20.10 (Groovy), the procedure for migrating an
OpenStack cloud from ML2+OVS to ML2+OVN may require an extra step due to Open
vSwitch bug `LP #1852221`_.

Following the procedure in the `Migration from Neutron ML2+OVS to ML2+OVN`_
section of the deploy guide, the workaround is to restart the ``ovs-vswitchd``
service after resuming the ovn-chassis charm in step 15:

.. code-block:: none

   juju run-action --wait neutron-openvswitch/0 cleanup
   juju run-action --wait ovn-chassis/0 resume
   juju run --unit ovn-chassis/0 'systemctl restart ovs-vswitchd'

.. LINKS
.. _Release Notes: https://docs.openstack.org/charm-guide/latest/release-notes.html
.. _Upgrade issues: upgrade-issues.html
.. _Special charm procedures: upgrade-special.html
.. _Migration from Neutron ML2+OVS to ML2+OVN: https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/app-ovn.html#migration-from-neutron-ml2-ovs-to-ml2-ovn

.. BUGS
.. _LP #1896630: https://bugs.launchpad.net/charm-layer-ovn/+bug/1896630
.. _LP #1899104: https://bugs.launchpad.net/ubuntu/+source/barbican/+bug/1899104
.. _LP #1827690: https://bugs.launchpad.net/charm-barbican/+bug/1827690
.. _LP #1904199: https://bugs.launchpad.net/charm-ceph-iscsi/+bug/1904199
.. _LP #1914819: https://bugs.launchpad.net/charm-glance/+bug/1914819
.. _LP #1852221: https://bugs.launchpad.net/ubuntu/+source/openvswitch/+bug/1852221
