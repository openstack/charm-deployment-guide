==========================================
Appendix V: Bridge interface configuration
==========================================

Overview
--------

This appendix explores physical interface configuration options available for
OVS and OVN charms.

Available configuration options
-------------------------------

For both OVS and OVN charms there is a need to provide physical interfaces for
vSwitch bridges in order to be able to use VLAN and flat provider networks in
Neutron:

* OVS charms with ``data-port`` config:
  * ``neutron-openvswitch``;
  * ``neutron-gateway``;
* OVN charms with ``bridge-interface-mappings`` config:
  * ``ovn-chassis``;
  * ``ovn-dedicated-chassis``.

Utilising one interface for multiple purposes
---------------------------------------------

It is common to use a single network interface for providing network access to
different types of workloads. Workloads such as OpenStack API services and Ceph
may use VLAN interfaces at the host level or via VLAN-specific container
bridges. Neutron services may rely on OVS bridges with physical network
interfaces or bonds used as virtual switch uplinks. The following diagram shows
how this can be achieved

.. code-block:: console

   +----------------+      +-----------------+
   |      LXD       |      |VM or router port|
   | container port |      +-----------------+
   +----------------+              ||
           ||              +-----------------+
           ||              |   br-int (OVS)  |
   +----------------+      +-----------------+
   |  br-bond1.100  |              ||
   |  Linux bridge  |      +-----------------+
   | with L3 config |      |br-provider (OVS)|
   +----------------+      +-----------------+
           ||                      ||
   +----------------+      +-----------------+
   |    bond1.100   |      |      bond1      |
   |  no L3 config  |      |  no L3 config   |
   +----------------+      +-----------------+
                              ||         ||
                           +------+   +------+
                           | hwe0 |   | hwe1 |
                           +------+   +------+

``bond1`` configured at the Linux kernel level can have VLAN interfaces
such as ``bond1.100`` which will make all traffic tagged with VLAN 100 to be
forwarded to ``bond1.100`` instead of ``bond1`` so it will not reach
``br-provider``. In this example, all untagged and tagged traffic, except for
VLAN 100, will be forwarded to ``br-provider`` via ``bond1``.

Any VLAN interface explicitly configured on top of ``bond1`` will make its
VLAN unusable in Neutron through the associated provider bridge because the
inbound traffic for that VLAN will always be intercepted by a ``bond1.<vid>``
interface at the kernel level.

.. warning::

   ``bond1`` must not have any L3 configuration for this setup to work.

This allows VLAN provider networks to be used for a range of VLANs dedicated
for use with Neutron in conjunction with some VLANs dedicated to host workloads.


Charm configuration examples
----------------------------

The following configuration assumes the setup mentioned above and that there
are no VLAN tenant networks - only VLAN provider networks (thus the
``vlan-ranges`` option only includes a physnet name).

For OVS deployments the charm configuration would look like this:

.. code-block:: none

   juju config neutron-api vlan-ranges='dcfabric'
   juju config neutron-openvswitch data-port='br-provider:bond1' bridge-mappings='dcfabric:br-provider' vlan-ranges='dcfabric'
   juju config neutron-gateway data-port='br-provider:bond1' bridge-mappings='dcfabric:br-provider' vlan-ranges='dcfabric'

Whereas for OVN deployments:

.. code-block:: none

   juju config neutron-api vlan-ranges='dcfabric'
   juju config ovn-chassis bridge-interface-mappings='br-provider:bond1' ovn-bridge-mappings='dcfabric:br-provider'
   juju config ovn-dedicated-chassis bridge-interface-mappings='br-provider:bond1' ovn-bridge-mappings='dcfabric:br-provider'

To configure a VLAN provider network the following command can be used with any
segment ID other than 100 as bond1.100 is present:

.. code-block:: none

   # --external is only needed for setups targeted at using floating IPs.
   openstack network create --external --provider-network-type vlan --provider-physical-network dcfabric --provider-segment 99
