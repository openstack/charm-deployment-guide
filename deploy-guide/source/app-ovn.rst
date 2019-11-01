Appendix O: Open Virtual Network (OVN)
======================================

Overview
++++++++

As of the 19.10 charm release, with OpenStack Train or later, support for
integration with Open Virtual Network (OVN) is available.

OVN charms:

* neutron-api-plugin-ovn

* ovn-central

* ovn-chassis

* ovn-dedicated-chassis

.. warning::

    The OVN charms are considered preview charms, still in development.

Deployment
++++++++++

OVN makes use of Public Key Infrastructure (PKI) to authenticate and authorize
control plane communication.  The charm requires a Certificate Authority to be
present in the model as represented by the ``certificates`` relation.

Follow the instructions for deployment and configuration of Vault in
`Appendix E <./app-certificate-management.html>`_.

OVN can then be deployed:

.. code::

    juju config neutron-api manage-neutron-plugin-legacy-mode=false

    juju deploy cs:~openstack-charmers/neutron-api-plugin-ovn
    juju deploy cs:~openstack-charmers/ovn-central -n 3 --config source=cloud:bionic-train
    juju deploy cs:~openstack-charmers/ovn-chassis

    juju add-relation neutron-api-plugin-ovn:certificates vault:certificates
    juju add-relation neutron-api-plugin-ovn:neutron-plugin \
        neutron-api:neutron-plugin-api-subordinate
    juju add-relation neutron-api-plugin-ovn:ovsdb-cms ovn-central:ovsdb-cms
    juju add-relation ovn-central:certificates vault:certificates
    juju add-relation ovn-chassis:ovsdb ovn-central:ovsdb
    juju add-relation ovn-chassis:certificates vault:certificates
    juju add-relation ovn-chassis:nova-compute nova-compute:neutron-plugin

The OVN components used for the data plane is deployed by the ``ovn-chassis``
subordinate charm.  A subordinate charm is deployed together with a principle
charm, ``nova-compute`` in the example above.

If you require a dedicated software gateway you may deploy the data plane
components as a principle charm through the use of the
`ovn-dedicated-chassis <https://jaas.ai/u/openstack-charmers/ovn-dedicated-chassis/>`_ charm.

.. note::

    You can also make use of `an OVN overlay bundle <https://raw.githubusercontent.com/openstack-charmers/openstack-bundles/master/development/overlays/openstack-base-ovn.yaml>`_ in conjunction with `the openstack base bundle <https://raw.githubusercontent.com/openstack-charmers/openstack-bundles/master/development/openstack-base-bionic-train/bundle.yaml>`_.

Configuration
+++++++++++++

OVN integrates with OpenStack through an ML2 driver as provided by
`networking-ovn <https://docs.openstack.org/networking-ovn/latest/>`_.  General
Neutron configuration is still done through the `neutron-api <https://jaas.ai/neutron-api/>`_
charm, and the subset of configuration specific to OVN is done through the
`neutron-api-plugin-ovn <https://jaas.ai/u/openstack-charmers/neutron-api-plugin-ovn/>`_ charm.

Internal DNS resolution
~~~~~~~~~~~~~~~~~~~~~~~

OVN supports Neutron internal DNS resolution.  To configure this:

.. code::

    juju config neutron-api enable-ml2-dns=true
    juju config neutron-api dns-domain=openstack.example.
    juju config neutron-api-plugin-api dns-servers="1.1.1.1 8.8.8.8"

.. note::

    The value for the ``dns-domain`` configuration option must
    not be set to 'openstack.local.' as that will effectively disable the
    feature.

    It is also important to end the string with a '.' (dot).

When you set ``enable-ml2-dns`` to 'true' and set a value for ``dns-domain``,
Neutron will add details such as instance name and DNS domain name to each
individual Neutron port associated with instances. ``networking-ovn`` will
then read instance name and DNS domain name from ports and populate the
``DNS`` table of the Northbound and Southbound databases:

.. code::

    # ovn-sbctl list DNS
    _uuid               : 2e149fa8-d27f-4106-99f5-a08f60c443bf
    datapaths           : [b25ed99a-89f1-49cc-be51-d215aa6fb073]
    external_ids        : {dns_id="4c79807e-0755-4d17-b4bc-eb57b93bf78d"}

    records             : {"c-1"="192.0.2.239", "c-1.openstack.example"="192.0.2.239"}

On the chassis, OVN creates flow rules to redirect UDP port 53 packets (DNS)
to the local ``ovn-controller`` process:

.. code::

    cookie=0xdeaffed, duration=77.575s, table=22, n_packets=0, n_bytes=0, idle_age=77, priority=100,udp6,metadata=0x2,tp_dst=53 actions=controller(userdata=00.00.00.06.00.00.00.00.00.01.de.10.00.00.00.64,pause),resubmit(,23)
    cookie=0xdeaffed, duration=77.570s, table=22, n_packets=0, n_bytes=0, idle_age=77, priority=100,udp,metadata=0x2,tp_dst=53 actions=controller(userdata=00.00.00.06.00.00.00.00.00.01.de.10.00.00.00.64,pause),resubmit(,23)

The local ``ovn-controller`` process then decides if it should respond to the
DNS query directly or if it needs to be forwarded to the real DNS server.

External connectivity
~~~~~~~~~~~~~~~~~~~~~

Interface and network to bridge mapping is done through the
`ovn-chassis <https://jaas.ai/u/openstack-charmers/ovn-chassis/>`_ charm.

Networks for use with external Layer3 connectivity should have mappings on
chassis located in the vicinity of the datacenter border gateways. Having two
or more chassis with mappings for a Layer3 network will have OVN automatically
configure highly available routers with liveness detection provided by the
Bidirectional Forwarding Detection (BFD) protocol.

Chassis without direct external mapping to a external Layer3 network will
forward traffic through a tunnel to one of the chassis acting as a gateway for
that network.

.. note::

    It is not necessary nor recommended to add mapping for external
    Layer3 networks to all chassis.  Doing so will create a scaling problem at
    the physical network layer that needs to be resolved with globally shared
    Layer2 (does not scale) or tunneling at the top-of-rack switch layer (adds
    complexity) and is generally not a recommended configuration.

Example configuration:

.. code:: bash

    juju config neutron-api flat-network-providers=physnet1
    juju config ovn-chassis ovn-bridge-mappings=physnet1:br-provider
    juju config ovn-chassis \
        interface-bridge-mappings='00:00:5e:00:00:42:br-provider \
                                   00:00:5e:00:00:51:br-provider'
    openstack network create --external --share --provider-network-type flat \
                             --provider-physical-network physnet1 ext-net
    openstack subnet create --network ext-net \
                            --subnet-range 192.0.2.0/24 \
                            --no-dhcp --gateway 192.0.2.1 \
                            ext

Networks for use with external Layer2 connectivity should have mappings present
on all chassis with potential to host the consuming payload.

Usage
+++++

Create networks, routers and subnets through the OpenStack API or CLI as you
normally would.

The ``networking-ovn`` driver will translate the OpenStack network constructs
into high level logical rules in the OVN Northbound database.

The ``ovn-northd`` daemon in turn translates this into data in the Southbound
database.

The local ``ovn-controller`` daemon on each chassis consumes these rules and
programs flows in the local Open vSwitch database.

Information queries
+++++++++++++++++++

.. note::

    Future versions of the charms will provide information-gathering in the
    form of actions and/or through updates to the ``juju status`` command.

OVSDB Cluster status
~~~~~~~~~~~~~~~~~~~~

.. code::

    juju run --application ovn-central 'ovs-appctl -t \
        /var/run/openvswitch/ovnnb_db.ctl cluster/status OVN_Northbound'
    juju run --application ovn-central 'ovs-appctl -t \
        /var/run/openvswitch/ovnsb_db.ctl cluster/status OVN_Southbound'

Querying DBs
~~~~~~~~~~~~

.. code::

    juju run --unit ovn-central/leader 'ovn-nbctl show'
    juju run --unit ovn-central/leader 'ovn-sbctl show'
    juju run --unit ovn-central/leader 'ovn-sbctl lflow-list'

Data plane flow tracing
~~~~~~~~~~~~~~~~~~~~~~~

.. code::

    juju run --unit ovn-chassis/1 'ovs-vsctl show'
    juju run --unit ovn-chassis/1 'ovs-ofctl dump-flows br-int'
    juju run --unit ovn-chassis/1 'sudo ovs-appctl -t ovs-vswitchd \
        ofproto/trace br-provider \
        in_port=enp3s0f0,icmp,nw_src=192.0.2.1,nw_dst=192.0.2.100'
