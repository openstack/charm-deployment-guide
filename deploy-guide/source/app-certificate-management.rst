====================================
Managing TLS certificates with Vault
====================================

Overview
--------

The preferred way to provide your charmed OpenStack deployment with
certificates for enabling transport layer security (TLS) is to add a
certificate authority to your model. The charms consume the certificates
through the `tls-certificates relation`_ and we do our validation using the
`Vault charm`_.

Enabling Vault Certificate Management
-------------------------------------

OpenStack charms providing an API service have a new 'certificates' relation.
Adding this relation will trigger the OpenStack charm to request
certificates and keys from vault. Once vault has provided these the charm
will install them and switch to listening on https, the catalog will also be
updated.

.. code:: bash

    juju add-relation keystone:certificates vault:certificates
    juju add-relation glance:certificates vault:certificates
    juju add-relation cinder:certificates vault:certificates
    juju add-relation nova-cloud-controller:certificates vault:certificates
    juju add-relation neutron-api:certificates vault:certificates
    ...

Adding a Certificate Authority (CA) certificate to Vault
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For Vault to be able to issue certificates on your behalf you must equip it
with a CA certificate.

You can either add your own intermediate CA certificate to Vault or have Vault
generate a self-signed root CA certificate for you.

Generate self-signed root CA certifitcate
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To have Vault generate a self-signed root CA certificate for you:

.. code-block:: none

   juju run-action --wait vault/leader generate-root-ca

Add your own intermediate CA certificate
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Currently, the only supported workflow is for Vault to generate a Certificate
Signing Request (CSR) for an intermediate CA. This CSR then needs to be signed
by an external CA. The resulting signed intermediate CA certificate is then
uploaded to Vault along with any certificates to support the certificate chain.

Retrieve CSR from Vault
.......................

Run the *get-csr* action against the lead unit of the vault application:

.. code:: bash

    $ juju run-action vault/0 get-csr
    Action queued with id: 0495b6ce-09d8-4e57-8d21-efa13794034a
    $ juju show-action-output 0495b6ce-09d8-4e57-8d21-efa13794034a
    results:
      output: |-
        -----BEGIN CERTIFICATE REQUEST-----
        MIICijCCAXICAQAwRTFDMEEGA1UEAxM6VmF1bHQgSW50ZXJtZWRpYXRlIENlcnRp
        ZmljYXRlIEF1dGhvcml0eSAoY2hhcm0tcGtpLWxvY2FsKTCCASIwDQYJKoZIhvcN
        AQEBBQADggEPADCCAQoCggEBAKEkTFyX2SgzDDUMvJNnoptdAb9AAiDZzc3U/aaX
        g55TmAQ5jIfYbXb/39O+81iWD8esWGzTkg9YyzfPUBwcF3FrsyyEPFjiFRhTEATl
        6W2U5tA981hKiEScF2BvMm4PJZM/1ND12yoIsg45n1AGUfY8GShqtKKNXvAR3TEj
        mYWQ35QhVFnsvh9hc5TUORRwlKy74FnCBDfuWuIGph/Ge4GRctAv0MUUsfqCv9DZ
        +xMHeE8IdyqMakTJ2iUXktbl0khSwbb5KUT9NOzC1mDPZPBICqSmhXQ1In9FR5ic
        VTsVOdHShaN4T4qjrvWI4+S5CJynDnSvDeUWPYu1lOM24H0CAwEAAaAAMA0GCSqG
        SIb3DQEBCwUAA4IBAQCHZoVqKJ0TCaYimHT2VGqklomGuQymxqkKIrBHQD4oe/Ts
        6nXXZaUWVPQF96o+Au4oz2R/l/UF4a89Z2xyCN95tys/QJv0Hw93MeN7cIGUZ8p+
        nOpb7IX5ZqWEKu1g+2cTQ0mxa1hGb+rbO0LtArbhWVQjrkqKdY26O78Ct76eutfO
        1uKYXMbbUlXH8Qb/MyfRt30nPqyLKz4bIqH0fm670s/MiK8zoJ8DwI3NpkzB0cfG
        qzYxRu078uWcenueDBa8HqvBQNT6meZ68y+eesXrIM4EY3dk4YSb5A5QSeqUAWjK
        Fp36v30jKNJ29rs8vduytRu6bz7bI+qZ1hCJbzQQ
        -----END CERTIFICATE REQUEST-----
    status: completed
    timing:
      completed: 2018-06-07 10:21:17 +0000 UTC
      enqueued: 2018-06-07 10:21:13 +0000 UTC
      started: 2018-06-07 10:21:13 +0000 UTC


Retrieve the CSR from the action output and place it in a file, removing any
leading whitespace.

Sign CSR
........

The exact command from signing the CSR will depend on the setup of the
external CA. Below is an example:

.. code:: bash

    openssl ca -config openssl.cnf -extensions v3_intermediate_ca -days 3650 \
        -notext -md sha256 -in csr_file -out /tmp/vault-charm-int.pem -batch \
        -passin pass:secretpassword

*If the signing is rejected due to mismatched O or OU or C etc then rerun the
get-csr actions and specify the mismatched items*

Upload signed CSR and root CA cert to vault
...........................................

(Where /tmp/root-ca.pem is the root ca cert)

.. code:: bash

    juju run-action vault/0 upload-signed-csr \
        pem="$(cat /tmp/vault-charm-int.pem | base64)" \
        root-ca="$(cat /tmp/root-ca.pem | base64)" \
        allowed-domains='openstack.local'

.. note::

    The certificates provided via the 'pem' parameter must be a PEM bundle
    containing the signed certificate, any intermediate CA certs external
    to Vault and the root CA cert.  Without this information Vault cannot
    verify the trust chain and will reject the provided certificate - see
    `RFC5280`_ for more details about certificate paths and trust.

    If external intermediate CAs are in use the root-ca PEM must also
    be a PEM bundle including certs for all intermediate CAs and the root
    CA.

    For more details about the format of certificate PEM bundles see
    `RFC7468`_.

Vault issues certificates
~~~~~~~~~~~~~~~~~~~~~~~~~

Vault will now issue certificates to all clients that have requested them.
This process will trigger the api charms to request endpoint updates from
keystone to reflect that they are now using https. This can be a lengthy
process, so monitor keystone units and wait for them to become idle.

.. code:: bash


    watch -d juju status keystone

Test
~~~~

Where /tmp/root-ca.pem is the root CA cert:

.. code:: bash

    source novarc # make sure you have https in OS_AUTH_URL

    echo "Testing: keystone"
    openstack --os-cacert /tmp/root-ca.pem catalog list
    echo "Testing: nova-cloud-controller"
    openstack --os-cacert /tmp/root-ca.pem server list
    echo "Testing: cinder"
    openstack --os-cacert /tmp/root-ca.pem volume list
    echo "Testing: neutron"
    openstack --os-cacert /tmp/root-ca.pem network list
    echo "Testing: image"
    openstack --os-cacert /tmp/root-ca.pem image list
    deactivate

Reissuing certificates
~~~~~~~~~~~~~~~~~~~~~~

The vault charm has an *reissue-certificates* action. Running the action
will cause vault to issue new certificates for all charm clients. The action
must be run on the lead unit.

.. code:: bash

   juju run-action vault/0 reissue-certificates

.. LINKS
.. _RFC5280: https://tools.ietf.org/html/rfc5280#section-3.2
.. _RFC7468: https://tools.ietf.org/html/rfc7468#section-5
.. _tls-certificates relation: https://github.com/juju-solutions/interface-tls-certificates#readme
.. _Vault charm: https://jaas.ai/vault/
