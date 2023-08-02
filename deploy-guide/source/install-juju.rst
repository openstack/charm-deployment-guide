============
Install Juju
============

In the :doc:`previous section <install-maas>`, we set up the base environment
in the form of a `MAAS`_ cluster. We are now going to implement `Juju`_ as a
management solution for that environment. The main goal will be the creation of
a Juju controller, the administrative node for a Juju-managed cloud.

Install Juju
------------

To install Juju:

.. code-block:: none

   sudo snap install juju --classic --channel 2.9

Add MAAS to Juju
----------------

Add the MAAS cluster so Juju will be able to manage it as a cloud. We'll do
this via a cloud definition file, such as ``maas-cloud.yaml``:

.. code-block:: yaml

   clouds:
     maas-one:
       type: maas
       auth-types: [oauth1]
       endpoint: http://10.246.112.3:5240/MAAS

We've called the cloud 'maas-one' and its endpoint is based on the IP address
of the MAAS system.

The cloud is added in this way:

.. code-block:: none

   juju add-cloud --client -f maas-cloud.yaml maas-one

View the updated list of clouds known to the current Juju client with the
:command:`juju clouds --client` command.

Add the MAAS credentials
------------------------

Add the MAAS credentials so Juju can interact with the newly added cloud.
We'll again use a file to import our information, such as ``maas-creds.yaml``:

.. code-block:: yaml

   credentials:
     maas-one:
       anyuser:
         auth-type: oauth1
         maas-oauth: LGJ8svffZZ5kSdeA8E:9kVM7jJpHGG6J9apk3:KE65tLnjpPuqVHZ6vb97T8VWfVB9tM3j

We've included the name of the cloud 'maas-one' and a new user 'anyuser'. The
long key is the MAAS API key for the MAAS 'admin' user. This key was placed in
file ``~/admin-api-key`` on the MAAS system during the :ref:`Install MAAS
<install_maas>` step on the previous page. It can also be obtained from the
'admin' user's profile in the web UI.

The credentials are added in this way:

.. code-block:: none

   juju add-credential --client -f maas-creds.yaml maas-one

View the updated list of credentials known to the current Juju client with the
:command:`juju credentials --client --show-secrets --format yaml` command.

Create the Juju controller
--------------------------

Create the controller (using the 'jammy' series) for the 'maas-one' cloud, and
call it 'maas-controller':

.. code-block:: none

   juju bootstrap --bootstrap-series=jammy --constraints tags=juju maas-one maas-controller

The ``--constraints`` option allows us to effectively select a node in the MAAS
cluster. Recall that we attached a tag of 'juju' to the lower-resourced MAAS
node during the :ref:`Tag nodes <tag_nodes>` step on the previous page.

The MAAS web UI will show the node being deployed. The whole process will take
about five minutes.

View the updated list of controllers known to the current Juju client with the
:command:`juju controllers` command.

Create the model
----------------

The OpenStack deployment will be placed in its own Juju model for
organisational purposes. Create the model 'openstack' and specify our desired
series of 'jammy':

.. code-block:: none

   juju add-model --config default-series=jammy openstack

The output of the :command:`juju status` command summarises the Juju aspect of
the environment. It should now look very similar to this:

.. code-block:: none

   Model      Controller       Cloud/Region      Version  SLA          Timestamp
   openstack  maas-controller  maas-one/default  2.9.43   unsupported  20:28:32Z

   Model "admin/openstack" is empty.

Next steps
----------

The next step is to use Juju to deploy OpenStack. This will involve deploying
the OpenStack applications and adding relations between them. Go to
:doc:`Install OpenStack <install-openstack>` now.

.. LINKS
.. _Juju: https://juju.is
.. _MAAS: https://maas.io
