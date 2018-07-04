Appendix C: Vault
==============================

Overview
++++++++

Vault provides a secure storage and certificate management service.
Integrating vault into an OpenStack deployment involves a few
post-deployment steps which have been encapsulated in charm actions.

Vault
+++++

Deploy vault
~~~~~~~~~~~~

First deploy the vault charm along with supporting services:

.. code:: bash

    juju deploy --to lxd:0 vault
    juju add-relation vault percona-cluster

.. note::

    When running on hardware or in KVM, the vault charm will configure
    vault to enable mlock (memory locking) which ensures that vault
    won't get swapped out to disk, potentially compromising secrets.
    Using mlock is not supported in LXD containers and will be
    automatically disabled.

Vault can make use of the existing Percona XtraDB Cluster deployed to
support the rest of the OpenStack applications or could be deployed
with a separate instance if required.  All data stored is encrypted by
vault using its master encryption key.

Vault will deploy and startup in an un-initialized state;  For production
deployments the unseal keys used to manage the Vault master key used for
encryption should be managed from outside of the Juju model hosting vault and
the OpenStack Charms.

.. note::

   Production deployments you should also secure vault using the ssl-*
   configuration options provided by the charm; this ensures that communication
   with the Vault REST API is always secure as data on the network is also
   encrypted using TLS encryption.

Initialize and unseal vault
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Install the vault snap and configure the VAULT_ADDR environment variable:

.. code:: bash

    sudo snap install vault
    export VAULT_ADDR="https://<IP of vault unit>:8200"

and then initialize the vault deployment:

.. code:: bash

    vault operator init -key-shares=5 -key-threshold=3

you should get a response like:

.. code:: bash

    Unseal Key 1: XqeOza3SY6f4L6xfuk6f8JumrEF7cak9mUXCCPRXzs4B
    Unseal Key 2: djvVAAste0F5iSe43nmBs2ZX5r+wUqHe4UfUrcprWkyM
    Unseal Key 3: iSXHBdTNIKrbd3JIEI+n+q7j04Q4HPsQOHgk7apupttT
    Unseal Key 4: J8jzdUi0HOWgx8Ig75Mu8uVQTXkw6yNv2kMhjeNz2/5B
    Unseal Key 5: xTJkk3guA9Mq3CYfDhIBevSWrD1CIer6q70HVACMbduc

    Initial Root Token: ebded15e-c908-5d3a-1df0-1e7e7218c162

    Vault initialized with 5 key shares and a key threshold of 3. Please securely
    distribute the key shares printed above. When the Vault is re-sealed,
    restarted, or stopped, you must supply at least 3 of these keys to unseal it
    before it can start servicing requests.

    Vault does not store the generated master key. Without at least 3 key to
    reconstruct the master key, Vault will remain permanently sealed!

    It is possible to generate new unseal keys, provided you have a quorum of
    existing unseal keys shares. See "vault rekey" for more information.

This response details the 5 keys that can be used to unseal the vault deployment
and an initial root token for accessing the Vault API.

.. warning::

    Do not lose the unseal keys! It's impossible to unseal
    vault without them which must be completed after any restart
    of the vault daemon as part of ongoing maintenance.

.. warning::

    Do not lose the root token! Without it the vault deployment will
    be inaccessible.

Each vault unit must be individually unsealed, so if there are multiple vault
units repeat the unseal process below for each unit changing the VAULT_ADDR
environment variable each time to point at the individual units.

.. code:: bash

    vault operator unseal XqeOza3SY6f4L6xfuk6f8JumrEF7cak9mUXCCPRXzs4B
    vault operator unseal djvVAAste0F5iSe43nmBs2ZX5r+wUqHe4UfUrcprWkyM
    vault operator unseal iSXHBdTNIKrbd3JIEI+n+q7j04Q4HPsQOHgk7apupttT


Authorize vault charm
~~~~~~~~~~~~~~~~~~~~~

vault is now ready for use - however the charm needs to be authorized
using a root token to be able to create secrets storage back-ends and
roles to allow other applications to access vault for encryption key
storage.

First generate a one-shot root token with a limited TTL using the
initial root token for this purpose:

.. code:: bash

   export VAULT_TOKEN=ebded15e-c908-5d3a-1df0-1e7e7218c162
   vault token create -ttl=10m

you should get a response like:

.. code:: bash

    Key                Value
    ---                -----
    token              03ceadf5-529d-6a64-0cfd-1e341b1dacb1
    token_accessor     17390537-2012-51dc-93d0-9cc26ba953eb
    token_duration     10m
    token_renewable    true
    token_policies     [root]

This token can then be used to setup access for the charm to
Vault:

.. code:: bash

    juju run-action vault/0 authorize-charm token=03ceadf5-529d-6a64-0cfd-1e341b1dacb1

After the action completes execution, the vault unit will go active
and any pending requests for secrets storage will be processed for
consuming applications.

Enabling HA
~~~~~~~~~~~

The vault charm supports deployment in HA configurations; this requires
the use of etcd to provide HA storage to vault units, with access to
vault being provided a virtual IP or DNS-HA hostname.

The etcd application needs to support etcd3 so ensure it is using the latest
snap channel which supports it:

.. code:: bash

    juju deploy --to lxd:0 vault
    juju add-unit --to lxd:1 vault
    juju add-unit --to lxd:2 vault
    juju config vault vip=10.20.30.1
    juju deploy hacluster vault-hacluster
    juju add-relation vault vault-hacluster

    juju deploy --config channel=3.1/stable --to lxd:0 etcd
    juju add-unit --to lxd:1 etcd
    juju add-unit --to lxd:2 etcd

    juju deploy --to lxd:0 easyrsa  # required for TLS certs for etcd

    juju add-relation etcd easyrsa
    juju add-relation etcd vault
    juju add-relation vault percona-cluster

Only a single vault unit is 'active' at any point in time (reflected in juju
status output). Other vault units will proxy incoming API requests to the
active vault unit over a secure cluster connection between units.

.. note::

    When deploying vault in HA configurations, all vault units must be
    unsealed using the unseal keys generated during initialization
    in order to unlock the master key. This is performed externally
    to the charm using the Vault API.
