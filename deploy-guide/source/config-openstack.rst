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

Create the admin user environment
---------------------------------

Somehow we need to gain administrative control of OpenStack, the key piece of
which is the Keystone administrator password. This is achieved using our
default Juju administrative powers. Typically a script is used to generate this
OpenStack admin environment. Let this script be placed in a file called
``openrc`` whose contents is:

.. code-block:: bash

   OS_PARAMS=$(env | awk 'BEGIN {FS="="} /^OS_/ {print $1;}' | paste -sd ' ')
   for param in $_OS_PARAMS; do
       if [ "$param" = "OS_AUTH_PROTOCOL" ]; then continue; fi
       if [ "$param" = "OS_CACERT" ]; then continue; fi
       unset $param
   done
   unset _OS_PARAMS

   _keystone_ip=$(juju run $_juju_model_arg --unit keystone/leader 'unit-get private-address')
   _password=$(juju run $_juju_model_arg --unit keystone/leader 'leader-get admin_passwd')

   export OS_AUTH_URL=${OS_AUTH_PROTOCOL:-http}://${_keystone_ip}:5000/v3
   export OS_USERNAME=admin
   export OS_PASSWORD=${_password}
   export OS_USER_DOMAIN_NAME=admin_domain
   export OS_PROJECT_DOMAIN_NAME=admin_domain
   export OS_PROJECT_NAME=admin
   export OS_REGION_NAME=RegionOne
   export OS_IDENTITY_API_VERSION=3
   # Swift needs this:
   export OS_AUTH_VERSION=3
   # Gnocchi needs this
   export OS_AUTH_TYPE=password

Note that the origin of this file is the `openstack-bundles`_ repository.

Source the file to become the admin user:

.. code-block:: none

   source openrc
   echo $OS_USERNAME

The output for the last command should be **admin**.

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

   +----------------------------------+-----------+--------------+--------------+---------+-----------+----------------------------------------+
   | ID                               | Region    | Service Name | Service Type | Enabled | Interface | URL                                    |
   +----------------------------------+-----------+--------------+--------------+---------+-----------+----------------------------------------+
   | 0515d09c36dd4fd991a1b2aa448eb3cb | RegionOne | neutron      | network      | True    | admin     | http://10.0.0.7:9696                   |
   | 0abda66d8c414faea7e7485ea6e8ff80 | RegionOne | glance       | image        | True    | admin     | http://10.0.0.20:9292                  |
   | 46599b147a2e4ff79513d8a4c6a37a83 | RegionOne | cinderv2     | volumev2     | True    | admin     | http://10.0.0.24:8776/v2/$(tenant_id)s |
   | c046918276db46a7b9e0106d5102927f | RegionOne | cinderv3     | volumev3     | True    | admin     | http://10.0.0.24:8776/v3/$(tenant_id)s |
   | c2a70ec99ec6417988e57f093ff4888d | RegionOne | keystone     | identity     | True    | admin     | http://10.0.0.29:35357/v3              |
   | c79512b6f9774bb59f23b5b687ac286d | RegionOne | placement    | placement    | True    | admin     | http://10.0.0.11:8778                  |
   | e8fbd499be904832b8ffa55fcb9c6efb | RegionOne | nova         | compute      | True    | admin     | http://10.0.0.10:8774/v2.1             |
   +----------------------------------+-----------+--------------+--------------+---------+-----------+----------------------------------------+

If the endpoints aren't visible, it's likely your environment variables aren't
set correctly.

Create an image and flavor
~~~~~~~~~~~~~~~~~~~~~~~~~~

Import a boot image into Glance to create server instances with. Here we import
a Bionic amd64 image and call it 'bionic x86_64':

.. code-block:: none

   curl http://cloud-images.ubuntu.com/bionic/current/bionic-server-cloudimg-amd64.img | \
      openstack image create --public --container-format bare --disk-format qcow2 \
      --property architecture=x86_64 --property hw_disk_bus=virtio \
      --property hw_vif_model=virtio "bionic x86_64"

Create at least one flavor to define a hardware profile for new instances. Here
we create one called 'm1.micro':

.. code-block:: none

   openstack flavor create --ram 512 --disk 4 m1.micro

The above flavor is defined with minimum specifications for Ubuntu Server.
Adjust according to your needs.

.. _public_networking:

Set up public networking
~~~~~~~~~~~~~~~~~~~~~~~~

Create the external public network, here called 'Pub_Net'. We use the 'flat'
network provider type and its provider 'physnet1' that were set up during the
:ref:`Neutron networking <neutron_networking>` step on the previous page:

.. code-block:: none

   openstack network create Pub_Net --external --share --default \
      --provider-network-type flat --provider-physical-network physnet1

Create the subnet, here called 'Pub_Subnet', for the above network. The values
used are based on the local environment. For instance, recall that our MAAS
subnet is '10.0.0.0/21':

.. code-block:: none

   openstack subnet create Pub_Subnet --allocation-pool start=10.0.8.1,end=10.0.8.199 \
      --subnet-range 10.0.0.0/21 --no-dhcp --gateway 10.0.0.1 \
      --network Pub_Net

.. important::

   The addresses in the public subnet allocation pool are managed within
   OpenStack but they also reside on the subnet managed by MAAS. It is
   important to tell MAAS to never use this address range. This is done via a
   `Reserved IP range`_ in MAAS.

Create the non-admin user environment
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Create a new domain, project, and user. Here we'll use 'Domain1', 'Project1',
and 'User1' respectively. You will be prompted to provide the new user's
password.

.. code-block:: none

   openstack domain create Domain1
   openstack project create --domain Domain1 Project1
   openstack user create --domain Domain1 --project Project1 --password-prompt User1

Sample results:

.. code-block:: console

   User Password:********
   Repeat User Password:********
   +---------------------+----------------------------------+
   | Field               | Value                            |
   +---------------------+----------------------------------+
   | default_project_id  | 2962d44b73db4e1d884498b8ce000a69 |
   | domain_id           | 5080f063d9f84290a8233e16a0ff39a2 |
   | enabled             | True                             |
   | id                  | 1ea06b07c73149ca9c6753e07c30383a |
   | name                | User1                            |
   | options             | {}                               |
   | password_expires_at | None                             |
   +---------------------+----------------------------------+

Take note of the output. We'll need the user's ID in order to assign her the
'Member' role:

.. code-block:: none

   openstack role add --user 1ea06b07c73149ca9c6753e07c30383a \
      --project Project1 Member

Create an OpenStack user authentication file for user 'User1'. All we're
missing is the Keystone URL, which we can get from the current user 'admin'
environment:

.. code-block:: none

   echo $OS_AUTH_URL

The output for the last command for this example is
**http://10.0.0.23:5000/v3**.

The contents of the file, say ``Project1-rc``, will therefore look like this
(assuming the user password is 'ubuntu'):

.. code-block:: bash

   export OS_AUTH_URL=http://10.0.0.23:5000/v3
   export OS_USER_DOMAIN_NAME=Domain1
   export OS_USERNAME=User1
   export OS_PROJECT_DOMAIN_NAME=Domain1
   export OS_PROJECT_NAME=Project1
   export OS_PASSWORD=ubuntu

Source the file to become the non-admin user:

.. code-block:: none

   source Project1-rc
   echo $OS_USERNAME

The output for the last command should be **User1**.

Perform actions as the non-admin user
-------------------------------------

The actions in this section should be performed as user 'User1'.

Set the user environment
~~~~~~~~~~~~~~~~~~~~~~~~

Perform a cloud query to ensure the user environment is functioning correctly:

.. code-block:: none

   openstack image list
   +--------------------------------------+---------------+--------+
   | ID                                   | Name          | Status |
   +--------------------------------------+---------------+--------+
   | 429f79c7-9ed9-4873-b6da-41580acd2d5f | bionic x86_64 | active |
   +--------------------------------------+---------------+--------+

The image that was previously imported by the admin user should be returned.

Set up private networking
~~~~~~~~~~~~~~~~~~~~~~~~~

In order to get a fixed IP address to access any created instances we need a
project-specific network with a private subnet. We'll also need a router to
link this network to the public network created earlier.

The non-admin user now creates a private internal network called 'Network1'
and an accompanying subnet called 'Subnet1' (the DNS server is pointing to the
MAAS server at 10.0.0.3):

.. code-block:: none

   openstack network create Network1 --internal
   openstack subnet create Subnet1 \
      --allocation-pool start=192.168.0.10,end=192.168.0.199 \
      --subnet-range 192.168.0.0/24 \
      --gateway 192.168.0.1 --dns-nameserver 10.0.0.3 \
      --network Network1

Now a router called 'Router1' is created, added to the subnet, and told to use
the public network as its external gateway network:

.. code-block:: none

   openstack router create Router1
   openstack router add subnet Router1 Subnet1
   openstack router set Router1 --external-gateway Pub_Net

Configure SSH and security groups
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Instances are accessed via SSH. Import a public SSH key so that it can be
referenced at instance creation time and then installed in the 'ubuntu' user
account. An existing key can be used but here we first create a new keypair
called 'User1-key':

.. code-block:: none

   ssh-keygen -q -N '' -f ~/.ssh/User1-key
   openstack keypair create --public-key ~/.ssh/User1-key.pub User1-key

Security groups will need to be configured to at least allow the passing of SSH
traffic. You can alter the default group rules or create a new group with its
own rules. We do the latter by creating a group called 'Allow_SSH':

.. code-block:: none

   openstack security group create --description 'Allow SSH' Allow_SSH
   openstack security group rule create --proto tcp --dst-port 22 Allow_SSH

Create and access an instance
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Determine the network ID of private network 'Network1' and then create an
instance called 'bionic-1':

.. code-block:: none

   NET_ID=$(openstack network list | grep Network1 | awk '{ print $2 }')
   openstack server create --image 'bionic x86_64' --flavor m1.micro \
      --key-name User1-key --security-group Allow_SSH --nic net-id=$NET_ID \
      bionic-1

Request a floating IP address from the public network 'Pub_Net' and assign it
to a variable:

.. code-block:: none

   FLOATING_IP=$(openstack floating ip create -f value -c floating_ip_address Pub_Net)

Now add that floating IP address to the newly-created instance 'bionic-1':

.. code-block:: none

   openstack server add floating ip bionic-1 $FLOATING_IP

Ask for a listing of all instances within the context of the current project
('Project1'):

.. code-block:: none

   openstack server list

Sample output:

.. code-block:: console

   +--------------------------------------+----------+--------+-----------------------------------+---------------+----------+
   | ID                                   | Name     | Status | Networks                          | Image         | Flavor   |
   +--------------------------------------+----------+--------+-----------------------------------+---------------+----------+
   | 9167b3e9-c653-43fc-858a-2d6f6da36daa | bionic-1 | ACTIVE | Network1=192.168.0.131, 10.0.8.10 | bionic x86_64 | m1.micro |
   +--------------------------------------+----------+--------+-----------------------------------+---------------+----------+

The first address listed is in the private network and the second one is in the
public network:

You can monitor the booting of the instance with this command:

.. code-block:: none

   openstack console log show bionic-1

The instance is ready when the output contains:

.. code-block:: console

   .
   .
   .
   Ubuntu 18.04.3 LTS bionic-1 ttyS0

   bionic-1 login:

You can connect to the instance in this way:

.. code-block:: none

   ssh -i ~/.ssh/User1-key ubuntu@$FLOATING_IP

Next steps
----------

You now have a functional OpenStack cloud managed by MAAS-backed Juju and have
reached the end of the Charms Deployment Guide.

Just as we used MAAS as a backing cloud to Juju, an optional objective is to do
the same with the new OpenStack cloud. That is, you would add the OpenStack
cloud to Juju, add a set of credentials, create a Juju controller, and go on
to deploy charms. The resulting Juju machines will be running as OpenStack
instances! See `Using OpenStack with Juju`_ in the Juju documentation for
guidance.

.. LINKS
.. _openstack-bundles: https://github.com/openstack-charmers/openstack-bundles/blob/master/stable/shared/openrcv3_project
.. _Reserved IP range: https://maas.io/docs/concepts-and-terms#heading--ip-ranges
.. _Using OpenStack with Juju: https://jaas.ai/docs/openstack-cloud
