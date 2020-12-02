=================
Appendix C: Vault
=================

Overview
++++++++

Vault provides a secure storage and certificate management service.
Integrating Vault into an OpenStack deployment involves a number of
post-deployment steps.

.. note::

   For actual deployment steps see the `vault charm`_.

Vault client
~~~~~~~~~~~~

Vault is needed as a client in order to manage the Vault deployment. Install it
on the host where the Juju client resides:

.. code:: none

   sudo snap install vault

Initialise Vault
~~~~~~~~~~~~~~~~

Identify the vault unit by setting the ``VAULT_ADDR`` environment variable
based on the IP address of the unit. This can be discovered from :command:`juju
status` output (column 'Public address'). Here well use '10.0.0.126':

.. code:: none

   export VAULT_ADDR="http://10.0.0.126:8200"

Initialise Vault by specifying the number of unseal keys that should get
generated as well as the number of unseal keys that are needed in order to
complete the unseal process. Below we will specify five and three,
respectively:

.. code:: none

   vault operator init -key-shares=5 -key-threshold=3

Sample output:

.. code:: console

   Unseal Key 1: XONSc5Ku8HJu+ix/zbzWhMvDTiPpwWX0W1X/e/J1Xixv
   Unseal Key 2: J/fQCPvDeMFJT3WprfPy17gwvyPxcvf+GV751fTHUoN/
   Unseal Key 3: +bRfX5HMISegsODqNZxvNcupQp/kYQuhsQ2XA+GamjY4
   Unseal Key 4: FMRTPJwzykgXFQOl2XTupw2lfgLOXbbIep9wgi9jQ2ls
   Unseal Key 5: 7rrxiIVQQWbDTJPMsqrZDKftD6JxJi6vFOlyC0KSabDB

   Initial Root Token: s.ezlJjFw8ZDZO6KbkAkm605Qv

   Vault initialized with 5 key shares and a key threshold of 3. Please securely
   distribute the key shares printed above. When the Vault is re-sealed,
   restarted, or stopped, you must supply at least 3 of these keys to unseal it
   before it can start servicing requests.

   Vault does not store the generated master key. Without at least 3 key to
   reconstruct the master key, Vault will remain permanently sealed!

   It is possible to generate new unseal keys, provided you have a quorum of
   existing unseal keys shares. See "vault operator rekey" for more information.

Besides displaying the five unseal keys the output also includes an "initial
root token". This token is used to access the Vault API.

.. warning::

   It is not possible to unseal Vault without the unseal keys, nor is it
   possible to manage Vault without the intial root token. **Store this
   information in a safe place immediately**.

Unseal Vault
~~~~~~~~~~~~

Unseal the vault unit using the requisite number of unique keys (three in this
example):

.. code:: none

   vault operator unseal XONSc5Ku8HJu+ix/zbzWhMvDTiPpwWX0W1X/e/J1Xixv
   vault operator unseal FMRTPJwzykgXFQOl2XTupw2lfgLOXbbIep9wgi9jQ2ls
   vault operator unseal 7rrxiIVQQWbDTJPMsqrZDKftD6JxJi6vFOlyC0KSabDB

In an HA environment repeat the unseal process for each unit. Prior to
unsealing a different unit change the value of the ``VAULT_ADDR`` variable so
that it points to that unit.

.. note::

   Maintenance work on the cloud may require vault units to be paused and later
   resumed. A resumed vault unit will be sealed and will therefore require
   unsealing. See the `Managing power events`_ section for details.

Proceed to the next step once all units have been unsealed.

Authorise the vault charm
~~~~~~~~~~~~~~~~~~~~~~~~~

The vault charm must be authorised to access the Vault deployment in order to
create storage backends (for secrets) and roles (to allow other applications to
access Vault for encryption key storage).

Generate a root token with a limited lifetime (10 minutes here) using the
initial root token:

.. code:: none

   export VAULT_TOKEN=s.ezlJjFw8ZDZO6KbkAkm605Qv
   vault token create -ttl=10m

Sample output:

.. code:: console

   Key                  Value
   ---                  -----
   token                s.QMhaOED3UGQ4MeH3fmGOpNED
   token_accessor       nApB972Dp2lnTTIF5VXQqnnb
   token_duration       10m
   token_renewable      true
   token_policies       ["root"]
   identity_policies    []
   policies             ["root"]

This temporary token ('token') is then used to authorise the charm:

.. code:: none

   juju run-action --wait vault/leader authorize-charm token=s.QMhaOED3UGQ4MeH3fmGOpNED

After the action completes execution, the vault unit(s) will become active and
any pending requests for secrets storage will be processed for consuming
applications.

Here is sample status output for an unsealed three-unit Vault cluster:

.. code:: console

   vault/0*                 active    idle   0/lxd/1  10.0.0.126      8200/tcp  Unit is ready (active: false, mlock: disabled)
     vault-hacluster/0*     active    idle            10.0.0.126                Unit is ready and clustered
     vault-mysql-router/0*  active    idle            10.0.0.126                Unit is ready
   vault/1                  active    idle   1/lxd/1  10.0.0.130      8200/tcp  Unit is ready (active: true, mlock: disabled)
     vault-hacluster/2      active    idle            10.0.0.130                Unit is ready and clustered
     vault-mysql-router/2   active    idle            10.0.0.130                Unit is ready
   vault/2                  active    idle   2/lxd/1  10.0.0.132      8200/tcp  Unit is ready (active: false, mlock: disabled)
     vault-hacluster/1      active    idle            10.0.0.132                Unit is ready and clustered
     vault-mysql-router/1   active    idle            10.0.0.132                Unit is ready

Managing TLS certificates
~~~~~~~~~~~~~~~~~~~~~~~~~

Vault can be used to manage a deployment's TLS certificates, either by basing
them on a self-signed CA certificate (that Vault can generate by itself) or on
a third-party CA certificate that you can upload to Vault.

Vault is the recommended way to use TLS in Charmed OpenStack. This topic is
covered on the `Certificate lifecycle management`_ page.

.. important::

   The OVN charms require TLS certificates to be managed by Vault.

.. LINKS
.. _vault charm: https://jaas.ai/vault
.. _Certificate lifecycle management: app-certificate-management.html
.. _Managing power events: app-managing-power-events.html#vault
