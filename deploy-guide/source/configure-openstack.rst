===================
Configure OpenStack
===================

Overview
--------

In the :doc:`previous section <install-openstack>`, we installed OpenStack. We
are now going to configure OpenStack with the intent of making it consumable by
regular users. Configuration will be performed by both the admin user and the
non-admin user.

Domains, projects, users, and roles are a vital part of OpenStack operations.
For the non-admin case, we'll create a single domain with a single project and
single user.

The tasks on this page should be performed on the host where the Juju client is
installed.

Install the OpenStack clients
-----------------------------

You'll need the OpenStack clients in order to manage your cloud from the
command line. Install them now:

.. code-block:: none

   sudo snap install openstackclients --classic

Create the admin user environment
---------------------------------

To gain control of the cloud the Keystone administrator password and the root
CA certificate are needed. This information can be most easily obtained by
using files created and maintained for this purpose. They can be found in the
`openstack-bundles`_ repository.

Download the repository and source the ``openrc`` file:

.. code-block:: none

   git clone https://github.com/openstack-charmers/openstack-bundles ~/openstack-bundles
   source ~/openstack-bundles/stable/openstack-base/openrc

.. note::

   For informational purposes, sourcing the file will result in the execution
   of these two commands (to obtain the CA certificate and password):

   .. code-block:: none

      juju run -m openstack --unit vault/leader 'leader-get root-ca'
      juju run -m openstack --unit keystone/leader 'leader-get admin_passwd'

The admin user environment should also now be set up. Verify this:

.. code-block:: none

   env | grep OS_

Sample output:

.. code-block:: console

   OS_REGION_NAME=RegionOne
   OS_AUTH_VERSION=3
   OS_CACERT=/home/ubuntu/snap/openstackclients/common/root-ca.crt
   OS_AUTH_URL=https://10.0.0.162:5000/v3
   OS_PROJECT_DOMAIN_NAME=admin_domain
   OS_AUTH_PROTOCOL=https
   OS_USERNAME=admin
   OS_AUTH_TYPE=password
   OS_USER_DOMAIN_NAME=admin_domain
   OS_PROJECT_NAME=admin
   OS_PASSWORD=aegoaquoo1veZae6
   OS_IDENTITY_API_VERSION=3

Perform actions as the admin user
---------------------------------

The actions in this section should be performed as user 'admin'.

Confirm the user environment
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

One way that you can confirm that the admin environment is set correctly is by
querying for cloud endpoints:

.. code-block:: none

   openstack endpoint list --interface admin

The output will look similar to this:

.. code-block:: console

   +----------------------------------+-----------+--------------+--------------+---------+-----------+------------------------------------------+
   | ID                               | Region    | Service Name | Service Type | Enabled | Interface | URL                                      |
   +----------------------------------+-----------+--------------+--------------+---------+-----------+------------------------------------------+
   | 172dc2610f2a46cbbf64919a7b414266 | RegionOne | cinderv3     | volumev3     | True    | admin     | https://10.0.0.171:8776/v3/$(tenant_id)s |
   | 60466514cde4401eaa810301bddb1d2c | RegionOne | glance       | image        | True    | admin     | https://10.0.0.167:9292                  |
   | 70be9abb201748078b6d91ff803ede86 | RegionOne | cinderv2     | volumev2     | True    | admin     | https://10.0.0.171:8776/v2/$(tenant_id)s |
   | 835f368961744d3aa62b0b7ead24c5c4 | RegionOne | placement    | placement    | True    | admin     | https://10.0.0.165:8778                  |
   | 9478c33a71994f9daa4d79a5630f1784 | RegionOne | neutron      | network      | True    | admin     | https://10.0.0.161:9696                  |
   | bcff6b5d81474cb9884b8161865b1394 | RegionOne | keystone     | identity     | True    | admin     | https://10.0.0.162:35357/v3              |
   | cb4dcb58607448c7981ddae79e8ca92d | RegionOne | nova         | compute      | True    | admin     | https://10.0.0.164:8774/v2.1             |
   +----------------------------------+-----------+--------------+--------------+---------+-----------+------------------------------------------+

If the endpoints aren't visible, it's likely your environment variables aren't
set correctly.

.. note::

   The helper files will set the Keystone endpoint variable ``OS_AUTH_URL`` to
   use HTTPS if TLS is detected anywhere in the cloud. This will always be the
   case due to the OVN requirement for TLS. If Keystone is not TLS-enabled (for
   some reason) you will need to manually reset the above variable to use HTTP.

Create an image and flavor
~~~~~~~~~~~~~~~~~~~~~~~~~~

Import a boot image into Glance to create server instances with. Here we import
a Focal amd64 image:

.. code-block:: none

   curl http://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img
      --output ~/cloud-images/focal-amd64.img

Now import the image and call it 'focal-amd64':

.. code-block:: none

   openstack image create --public --container-format bare \
      --disk-format qcow2 --file ~/cloud-images/focal-amd64.img \
      focal-amd64

Create at least one flavor to define a hardware profile for new instances. Here
we create one called 'm1.small':

.. code-block:: none

   openstack flavor create --ram 2048 --disk 20 --ephemeral 20 m1.small

Make sure that your MAAS nodes can accommodate the flavor's resources.

.. _public_networking:

Set up public networking
~~~~~~~~~~~~~~~~~~~~~~~~

Create an external public (shared) network, here called 'ext_net'. We use the
'flat' network provider type and its provider 'physnet1' that were set up
during the :ref:`Neutron networking <neutron_networking>` step on the previous
page:

.. code-block:: none

   openstack network create --external --share \
      --provider-network-type flat --provider-physical-network physnet1 \
      ext_net

Create the subnet, here called 'ext_subnet', for the above network. The values
used are based on the local environment. For instance, recall that our MAAS
subnet is '10.0.0.0/24':

.. code-block:: none

   openstack subnet create --network ext_net --no-dhcp \
      --gateway 10.0.0.1 --subnet-range 10.0.0.0/24 \
      --allocation-pool start=10.0.0.10,end=10.0.0.200 \
      ext_subnet

.. important::

   The addresses in the public subnet allocation pool are managed within
   OpenStack but they also reside on the subnet managed by MAAS. It is
   important to tell MAAS to never use this address range. This is done via a
   `Reserved IP range`_ in MAAS.

Create the non-admin user environment
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Create a new domain, project, and user. Here we'll use 'domain1', 'project1',
and 'user1' respectively. You will be prompted to provide the new user's
password:

.. code-block:: none

   openstack domain create domain1
   openstack project create --domain domain1 project1
   openstack user create --domain domain1 --project project1 --password-prompt user1

Sample output from the last command:

.. code-block:: console

   User Password:********
   Repeat User Password:********
   +---------------------+----------------------------------+
   | Field               | Value                            |
   +---------------------+----------------------------------+
   | default_project_id  | 47c42bfc695c4efcba92ab2345336265 |
   | domain_id           | 884c9966c24f4db291e2b89b27ce692b |
   | enabled             | True                             |
   | id                  | 8b16e5335976418e99bf0b798e83e413 |
   | name                | User1                            |
   | options             | {}                               |
   | password_expires_at | None                             |
   +---------------------+----------------------------------+

We'll use the user's ID to assign her the 'Member' role:

.. code-block:: none

   openstack role add --user 8b16e5335976418e99bf0b798e83e413 \
      --project project1 Member

Create an OpenStack user authentication file for user 'user1'. All we're
missing is the Keystone URL, which we can get from the current user 'admin'
environment:

.. code-block:: none

   echo $OS_AUTH_URL

The output for the last command for this example is
**https://10.0.0.162:5000/v3**.

The contents of the file, say ``project1-rc``, will therefore look like this
(assuming the user password is 'ubuntu'):

.. code-block:: ini

   export OS_AUTH_URL=https://10.0.0.162:5000/v3
   export OS_USER_DOMAIN_NAME=domain1
   export OS_USERNAME=user1
   export OS_PROJECT_DOMAIN_NAME=domain1
   export OS_PROJECT_NAME=project1
   export OS_PASSWORD=ubuntu

Source the file to become the non-admin user:

.. code-block:: none

   source project1-rc
   echo $OS_USERNAME

The output for the last command should be **user1**.

Perform actions as the non-admin user
-------------------------------------

The actions in this section should be performed as user 'user1'.

Set the user environment
~~~~~~~~~~~~~~~~~~~~~~~~

Perform a cloud query to ensure the user environment is functioning correctly:

.. code-block:: none

   openstack image list
   +--------------------------------------+-------------+--------+
   | ID                                   | Name        | Status |
   +--------------------------------------+-------------+--------+
   | 82517c74-1226-4dab-8a6b-59b4fe07f681 | focal-amd64 | active |
   +--------------------------------------+-------------+--------+

The image that was previously imported by the admin user should be returned.

Set up private networking
~~~~~~~~~~~~~~~~~~~~~~~~~

In order to get a fixed IP address to access any created instances we need a
project-specific network with a private subnet. We'll also need a router to
link this network to the public network created earlier.

The non-admin user now creates a private internal network called 'user1-net'
and an accompanying subnet called 'user1-subnet' (the DNS server is pointing to
the MAAS server at 10.0.0.2):

.. code-block:: none

   openstack network create --internal user1_net

   openstack subnet create --network user1_net --dns-nameserver 10.0.0.2 \
      --gateway 192.168.0.1 --subnet-range 192.168.0/24 \
      --allocation-pool start=192.168.0.10,end=192.168.0.200 \
      int_subnet

Now a router called 'user1_router' is created, added to the subnet, and told to
use the public external network as its gateway network:

.. code-block:: none

   openstack router create user1_router
   openstack router set --external-gateway ext_net user1_router
   openstack router add subnet user1_router user1_subnet

Configure SSH and security groups
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

An SSH keypair needs to be imported into the cloud in order to access your
instances.

Generate one first if you do not yet have one. This command creates a
passphraseless keypair (remove the `-N` option to avoid that):

.. code-block:: none

   ssh-keygen -q -N '' -f ~/cloud-keys/user1-key

To import a keypair:

.. code-block:: none

   openstack keypair create --public-key ~/cloud-keys/user1-key.pub user1

Security groups will need to be configured to allow the passing of SSH traffic.
You can alter the default group rules or create a new group with its own rules.
We do the latter by creating a group called 'Allow_SSH':

.. code-block:: none

   openstack security group create --description 'Allow SSH' Allow_SSH
   openstack security group rule create --proto tcp --dst-port 22 Allow_SSH

Create and access an instance
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Create a Focal amd64 instance called 'focal-1':

.. code-block:: none

   openstack server create --image focal-amd64 --flavor m1.small \
      --key-name user1 --network user1_net --security-group Allow_SSH \
      focal-1

Request and assign a floating IP address to the new instance:

.. code-block:: none

   FLOATING_IP=$(openstack floating ip create -f value -c floating_ip_address ext_net)
   openstack server add floating ip focal-1 $FLOATING_IP

Ask for a listing of all instances within the context of the current project
('project1'):

.. code-block:: none

   openstack server list

Sample output:

.. code-block:: console

   +--------------------------------------+---------+--------+-------------------------------------+-------------+----------+
   | ID                                   | Name    | Status | Networks                            | Image       | Flavor   |
   +--------------------------------------+---------+--------+-------------------------------------+-------------+----------+
   | 687b96d0-ab22-459b-935b-a9d0b7e9964c | focal-1 | ACTIVE | user1_net=192.168.0.154, 10.0.0.187 | focal-amd64 | m1.small |
   +--------------------------------------+---------+--------+-------------------------------------+-------------+----------+

The first address listed is in the private network and the second one is in the
public network:

You can monitor the booting of the instance with this command:

.. code-block:: none

   openstack console log show focal-1

The instance is ready when the output contains:

.. code-block:: console

   .
   .
   .
   Ubuntu 20.04.2 LTS focal-1 ttyS0

   focal-1 login:

Connect to the instance in this way:

.. code-block:: none

   ssh -i ~/cloud-keys/user1-key ubuntu@$FLOATING_IP

Next steps
----------

You now have a functional OpenStack cloud managed by MAAS-backed Juju.

Go on to read the many Charmed OpenStack topics in this guide or consider the
`OpenStack Administrator Guides`_ for upstream OpenStack administrative help.

.. LINKS
.. _openstack-bundles: https://github.com/openstack-charmers/openstack-bundles
.. _Reserved IP range: https://maas.io/docs/concepts-and-terms#heading--ip-ranges
.. _OpenStack Administrator Guides: http://docs.openstack.org/user-guide-admin/content
