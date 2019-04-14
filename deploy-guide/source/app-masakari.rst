Appendix L: Automated Instance Recovery
=======================================

Overview
++++++++

As of the 19.04 charm release, with OpenStack Stein and later, masakari can be
a deployed to provide automated instance recovery for guests using shared
storage. Masakari responds to two different failures types, individual guest
failure and the loss of an entire compute node.

.. warning::

    The Masakari charms are for use in development environments or PoC work
    they should not be treated as production ready yet.

STONITH
+++++++

It is important that guests using shared storage cannot continue to run in the
event of a compute node becoming isolated. The risk being that masakari
attempts to bring the same guest up on a new compute node when the old one
is still running which could lead to data corruption. To ensure that this does
not occur stonith can be setup for the compute nodes. For stonith to be
configured the **maas_url** and **maas_credentials** config option must be
set in the hacluster charm related to the masakari charm. Also the
**enable-stonith** config option should be set to **True** in the
pacemaker-remote charm.

Deployment
++++++++++

Three new charms are needeed to deploy this solution: masakari,
masakari-monitors and pacemaker-remote. The masakari charm provides api
services and is a principal or standalone charm. The masakari-monitors charm is
deployed as a subordinate to the nova-compute charm as it monitors
nova-compute directly and sends messages to the masakari API charm. The
pacemaker-remote charm is also a subordinate to the nova-compute charm and is
required to monitor the compute nodes health.

Below is an overlay which can be used to add masakari to an existing
deployment:

.. code::

    machines:
      '0':
        series: bionic
      '1':
        series: bionic
      '2':
        series: bionic
      '3':
        series: bionic
    relations:
    - - nova-compute:juju-info
      - masakari-monitors:container
    - - masakari:ha
      - hacluster:ha
    - - keystone:identity-credentials
      - masakari-monitors:identity-credentials
    - - nova-compute:juju-info
      - pacemaker-remote:juju-info
    - - hacluster:pacemaker-remote
      - pacemaker-remote:pacemaker-remote
    - - masakari:identity-service
      - keystone:identity-service
    - - masakari:shared-db
      - mysql:shared-db
    - - masakari:amqp
      - rabbitmq-server:amqp
    series: bionic
    applications:
      masakari-monitors:
        charm: cs:masakari-monitors
      hacluster:
        charm: cs:hacluster
        options:
          maas_url: <INSERT MAAS URL>
          maas_credentials: <INSERT MAAS API KEY>
      pacemaker-remote:
        charm: cs:pacemaker-remote
        options:
          enable-stonith: True
          enable-resources: False
      masakari:
        charm: cs:masakari
        series: bionic
        num_units: 3
        options:
          openstack-origin: cloud:bionic-stein
          vip: <INSERT VIP(S)>
        bindings:
          public: public
          admin: admin
          internal: internal
          shared-db: internal
          amqp: internal
        to:
        - 'lxd:1'
        - 'lxd:2'
        - 'lxd:3'

.. warning::

    The bundle above with need customising to correct maas_url,
    maas_credentials and vip settings. The machine mappings will almost
    certainly need updating too.

To use the overlay with an existing model remember to use the
**--map-machines** switch to juju

.. code::

    $ juju deploy base.yaml --overlay masakari-overlay.yaml --map-machines=existing

Configuring Masakari
++++++++++++++++++++

In Masakari the compute nodes are grouped into failover segments. In the event
of a failure guests are moved onto other nodes within the same segment. Which
compute node is chosen to house the evacuated guests is determined by the
recovery method of that segment.

'AUTO' Recovery Method
----------------------

With auto recovery the guests are relocated to any of the available nodes in
the same segment. The problem with this approach is that there is no guarantee
that resources will be available to accommodate guests from a failed compute
node.

To configure a group of compute hosts for auto recovery, first create a segment
with the recovery method set to auto:

.. code::

    $ openstack segment create segment1 auto COMPUTE
    +-----------------+--------------------------------------+
    | Field           | Value                                |
    +-----------------+--------------------------------------+
    | created_at      | 2019-04-12T13:59:50.000000           |
    | updated_at      | None                                 |
    | uuid            | 691b8ef3-7481-48b2-afb6-908a98c8a768 |
    | name            | segment1                             |
    | description     | None                                 |
    | id              | 1                                    |
    | service_type    | COMPUTE                              |
    | recovery_method | auto                                 |
    +-----------------+--------------------------------------+


Next the hypervisors need to be added into the segment, these should be
referenced by their unqualified hostname:

.. code::

    $ openstack segment host create tidy-goose COMPUTE SSH 691b8ef3-7481-48b2-afb6-908a98c8a768
    +---------------------+--------------------------------------+
    | Field               | Value                                |
    +---------------------+--------------------------------------+
    | created_at          | 2019-04-12T14:18:24.000000           |
    | updated_at          | None                                 |
    | uuid                | 11b85c9d-2b97-4b83-b773-0e9565e407b5 |
    | name                | tidy-goose                           |
    | type                | COMPUTE                              |
    | control_attributes  | SSH                                  |
    | reserved            | False                                |
    | on_maintenance      | False                                |
    | failover_segment_id | 691b8ef3-7481-48b2-afb6-908a98c8a768 |
    +---------------------+--------------------------------------+

Repeat above for all remaining hypervisors:


.. code::

    $ openstack segment host list 691b8ef3-7481-48b2-afb6-908a98c8a768
    +--------------------------------------+------------+---------+--------------------+----------+----------------+--------------------------------------+
    | uuid                                 | name       | type    | control_attributes | reserved | on_maintenance | failover_segment_id                  |
    +--------------------------------------+------------+---------+--------------------+----------+----------------+--------------------------------------+
    | 75afadbb-67cc-47b2-914e-e3bf848028e4 | frank-colt | COMPUTE | SSH                | False    | False          | 691b8ef3-7481-48b2-afb6-908a98c8a768 |
    | 11b85c9d-2b97-4b83-b773-0e9565e407b5 | tidy-goose | COMPUTE | SSH                | False    | False          | 691b8ef3-7481-48b2-afb6-908a98c8a768 |
    | f1e9b0b4-3ac9-4f07-9f83-5af2f9151109 | model-crow | COMPUTE | SSH                | False    | False          | 691b8ef3-7481-48b2-afb6-908a98c8a768 |
    +--------------------------------------+------------+---------+--------------------+----------+----------------+--------------------------------------+

'RESERVED_HOST' Recovery Method
-------------------------------

With reserved_host recovery compute hosts are allocated as reserved which
allows an operator to guarantee there is sufficient capacity available for any
guests in need of evacuation.

Firstly create a segment with the reserved_host recovery method:

.. code::

    $ openstack segment create segment1 reserved_host COMPUTE -c uuid -f value
    2598f8aa-3612-4731-9716-e126ca6cc280


Add a host using the --reserved switch to indicate that it will act as a
standby:

.. code::

    $ openstack segment host create model-crow --reserved True COMPUTE SSH 2598f8aa-3612-4731-9716-e126ca6cc280


Add the remaining hypervisors as before:

.. code::

    $ openstack segment host create frank-colt COMPUTE SSH 2598f8aa-3612-4731-9716-e126ca6cc280
    $ openstack segment host create tidy-goose COMPUTE SSH 2598f8aa-3612-4731-9716-e126ca6cc280


Listing the segment hosts shows that model-crow is a reserved host:

.. code::

    $ openstack segment host list 2598f8aa-3612-4731-9716-e126ca6cc280
    +--------------------------------------+------------+---------+--------------------+----------+----------------+--------------------------------------+
    | uuid                                 | name       | type    | control_attributes | reserved | on_maintenance | failover_segment_id                  |
    +--------------------------------------+------------+---------+--------------------+----------+----------------+--------------------------------------+
    | 4769e08c-ed52-440a-866e-832b977aa5e2 | tidy-goose | COMPUTE | SSH                | False    | False          | 2598f8aa-3612-4731-9716-e126ca6cc280 |
    | 90aedbd2-e03b-4dbd-b330-a1c848f300df | frank-colt | COMPUTE | SSH                | False    | False          | 2598f8aa-3612-4731-9716-e126ca6cc280 |
    | c77574cc-b6e7-440e-9c86-84e91981f15e | model-crow | COMPUTE | SSH                | True     | False          | 2598f8aa-3612-4731-9716-e126ca6cc280 |
    +--------------------------------------+------------+---------+--------------------+----------+----------------+--------------------------------------+

Finally disable the reserved host in nova so that it remains available for
failover:

.. code::

    $ openstack compute service set --disable model-crow nova-compute
    $ openstack compute service list
    +----+----------------+---------------------+----------+----------+-------+----------------------------+
    | ID | Binary         | Host                | Zone     | Status   | State | Updated At                 |
    +----+----------------+---------------------+----------+----------+-------+----------------------------+
    |  1 | nova-scheduler | juju-44b912-3-lxd-3 | internal | enabled  | up    | 2019-04-13T10:59:10.000000 |
    |  5 | nova-conductor | juju-44b912-3-lxd-3 | internal | enabled  | up    | 2019-04-13T10:59:08.000000 |
    |  7 | nova-compute   | tidy-goose          | nova     | enabled  | up    | 2019-04-13T10:59:11.000000 |
    |  8 | nova-compute   | frank-colt          | nova     | enabled  | up    | 2019-04-13T10:59:05.000000 |
    |  9 | nova-compute   | model-crow          | nova     | disabled | up    | 2019-04-13T10:59:12.000000 |
    +----+----------------+---------------------+----------+----------+-------+----------------------------+

When a compute node failure is detected, masakari will disable the failed node
and enable the reserve node in nova. After simulating a failure of frank-colt
the service list now looks like this:

.. code::

    $ openstack compute service list
    +----+----------------+---------------------+----------+----------+-------+----------------------------+
    | ID | Binary         | Host                | Zone     | Status   | State | Updated At                 |
    +----+----------------+---------------------+----------+----------+-------+----------------------------+
    |  1 | nova-scheduler | juju-44b912-3-lxd-3 | internal | enabled  | up    | 2019-04-13T11:05:20.000000 |
    |  5 | nova-conductor | juju-44b912-3-lxd-3 | internal | enabled  | up    | 2019-04-13T11:05:28.000000 |
    |  7 | nova-compute   | tidy-goose          | nova     | enabled  | up    | 2019-04-13T11:05:21.000000 |
    |  8 | nova-compute   | frank-colt          | nova     | disabled | down  | 2019-04-13T11:03:56.000000 |
    |  9 | nova-compute   | model-crow          | nova     | enabled  | up    | 2019-04-13T11:05:22.000000 |
    +----+----------------+---------------------+----------+----------+-------+----------------------------+

Since the reserved host has now been enabled and is hosting evacuated guests,
masakari has removed the reserved flag from it. Masakari has also placed the
failed node in maintenance mode.

.. code::

    $ openstack segment host list 2598f8aa-3612-4731-9716-e126ca6cc280
    +--------------------------------------+------------+---------+--------------------+----------+----------------+--------------------------------------+
    | uuid                                 | name       | type    | control_attributes | reserved | on_maintenance | failover_segment_id                  |
    +--------------------------------------+------------+---------+--------------------+----------+----------------+--------------------------------------+
    | 4769e08c-ed52-440a-866e-832b977aa5e2 | tidy-goose | COMPUTE | SSH                | False    | False          | 2598f8aa-3612-4731-9716-e126ca6cc280 |
    | 90aedbd2-e03b-4dbd-b330-a1c848f300df | frank-colt | COMPUTE | SSH                | False    | True           | 2598f8aa-3612-4731-9716-e126ca6cc280 |
    | c77574cc-b6e7-440e-9c86-84e91981f15e | model-crow | COMPUTE | SSH                | False    | False          | 2598f8aa-3612-4731-9716-e126ca6cc280 |
    +--------------------------------------+------------+---------+--------------------+----------+----------------+--------------------------------------+

‘AUTO_PRIORITY’ and ‘RH_PRIORITY’ Recovery Methods
--------------------------------------------------

These methods appear to chain the previous methods together. So, auto_priority
attempts to move the guest using the auto method first and if that fails it
tries the reserved_host method. rh_priority does the same thing but in the
reverse order. See
`Masakari Pike Release Note <https://docs.openstack.org/releasenotes/masakari/pike.html>`_  for details.

Individual Instance Recovery
----------------------------

Finally, to use the masakari feature which reacts to a single guest failing
rather than a whole hypervisor, the guest(s) need to be marked with a small
piece of metadata:

.. code::

    $ openstack server set --property HA_Enabled=True server_120419134342
