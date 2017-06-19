Prerequisites
-------------

Before you install and configure the charms service,
you must create a database, service credentials, and API endpoints.

#. To create the database, complete these steps:

   * Use the database access client to connect to the database
     server as the ``root`` user:

     .. code-block:: console

        $ mysql -u root -p

   * Create the ``charms`` database:

     .. code-block:: none

        CREATE DATABASE charms;

   * Grant proper access to the ``charms`` database:

     .. code-block:: none

        GRANT ALL PRIVILEGES ON charms.* TO 'charms'@'localhost' \
          IDENTIFIED BY 'CHARMS_DBPASS';
        GRANT ALL PRIVILEGES ON charms.* TO 'charms'@'%' \
          IDENTIFIED BY 'CHARMS_DBPASS';

     Replace ``CHARMS_DBPASS`` with a suitable password.

   * Exit the database access client.

     .. code-block:: none

        exit;

#. Source the ``admin`` credentials to gain access to
   admin-only CLI commands:

   .. code-block:: console

      $ . admin-openrc

#. To create the service credentials, complete these steps:

   * Create the ``charms`` user:

     .. code-block:: console

        $ openstack user create --domain default --password-prompt charms

   * Add the ``admin`` role to the ``charms`` user:

     .. code-block:: console

        $ openstack role add --project service --user charms admin

   * Create the charms service entities:

     .. code-block:: console

        $ openstack service create --name charms --description "charms" charms

#. Create the charms service API endpoints:

   .. code-block:: console

      $ openstack endpoint create --region RegionOne \
        charms public http://controller:XXXX/vY/%\(tenant_id\)s
      $ openstack endpoint create --region RegionOne \
        charms internal http://controller:XXXX/vY/%\(tenant_id\)s
      $ openstack endpoint create --region RegionOne \
        charms admin http://controller:XXXX/vY/%\(tenant_id\)s
