Appendix N: Policy Overrides
============================

Overview
++++++++

This appendix explains the purpose of policy overrides and shows how to enable
them.

.. important::

    Overrides are available on a per-charm basis and will be noted in a charm's
    README. Always consult the charm documentation prior to enabling this
    feature.

Background
++++++++++

The Policy Overrides feature provides a mechanism for operators to modify the
policies within OpenStack services in order to manage the permissions of the
tenants and users of the system. It is NOT intended that this facility is used
to manage the permissions of the services themselves, and steps are taken to
prevent this where possible.  The charms will continue to manage the policies
of the service users directly (e.g. keystone, glance, nova) using the default
policy files within the charms themselves.

The preferred approach has always been to use charm options to enable a more
controlled approach to modifying the policy of a service by providing templates
that the options modify - i.e. a controlled approach that always results in
consistent policy files.

However, over time, it has become apparent that there will always be an aspect
of the policy of a service that an operator wants to tweak or change that the
charm authors hadn't considered, and the cycle time to introduce the option
exceeds the expectations of the delay that an operator is willing to tolerate.

Therefore, it has been recognised that there are aspects of the configuration
files that the charms control need to be more fluidly modifiable by operators.
These tend to be orthogonal to the act of deployment and instead are to do with
operating the cloud.

So, the objective is to provide operators with a mechanism to override the
policy defaults without (hopefully) breaking the cloud. The policy defaults are
either coded in the service via "policy-in-code" and/or via a default policy
YAML file provided by the charm/service.

Facility offered
++++++++++++++++

The general facility offered is the ability to:

- Create YAML files that contain any rules that are allowed in policy files for
  each service.
- The YAML files are placed into a zip file.
- The ZIP file will be attached/uploaded to the charm as a `Juju resource
  <https://jaas.ai/docs/juju-resources>`_.
- A config item named ``use-policyd-override`` is set to ``True``.

This will cause the charm to unzip the attached file, do some validation
checks, and then drop the file(s) in the ``/etc/<service-name>/policy.d/``
directory.  The OpenStack service will then use these overrides to add to,
change, or remove the default charm- or package-determined policies and thus
provide the permissions that the operator requires.

Some charms may wish to provide a template system with the policy overrides.
In this case the override file may include template variables that the charm
will substitute with data that the charm has from config, the enviroment, or
through relation data.  This will be documented within the charms' README file.

Policies across the OpenStack services
++++++++++++++++++++++++++++++++++++++

The policy override system described here is *per charm*.  Thus if several
services require policy overrides then a policy override resource ZIP file
needs to be created for each charm and applied to the service.

Requirements
++++++++++++

Policy overrides are charm dependent, so the individual charm README files
should be consulted.  However, the following general issues may arise:

- The ZIP file is not properly formatted.  Check that a pkunzip program can
  open and test the enclosed files.
- There are no files in the zip file that have an extension of ``.yaml``, or
  ``.yml``.
- There are duplicate named files after the ZIP file has been processed.
  Directories in the zip file are "flattened" such that all of the files appear
  as a simple list.  Each of these flattened filename is lower-cased.  At this
  point any duplicates will cause a failure.  This processing is done so that
  the policy.d directory for the OpenStack service is a simple list of
  unambiguous files.
- An identified YAML file in the zip file is not formatted correctly; as it
  can't be loaded, it won't be written to the policy.d override folder.
- The template substitution function errors.
- A blacklisted key has been used in the policy override.  Individual charms
  may choose to disallow the override of particular rules in the policy file.
  In this case the policy file will be rejected.

Any problems with the overrides will be indicated in the output for ``juju
status``. See next section `Applying overrides`_.

.. note:: The hook (install, upgrade, config-changed) will not fail if the
          policy override is broken.  The policy overrides will not be
          installed, and the status line will indicate a failure.  It is not a
          charm breakage if the policy overrides are not installed as the
          overrides should be orthogonal to the operation of the charm.  i.e.
          they are to do with tenants and user permissions, and not the
          operation of the underlying services.

Applying overrides
++++++++++++++++++

Policy overrides for an OpenStack service are applied by attaching the ZIP file
to the corresponding charm and then enabling them by passing an option to the
charm. These are both done via Juju commands.

The file containing policy override files is attached to the charm in this way:

    juju attach-resource <charm-name> policyd-override=<overrides.zip>

The policy override is enabled in the charm using:

    juju config <charm-name> use-policyd-override=true

Juju status
+++++++++++

The status of the overrides for a Juju application is shown in the output for
the ``juju status`` command. When overrides are successful the text ``PO:``
(for Policy Overrides) will be prefixed to the application's status message.
When they are unsuccessful ``PO: (broken)`` will be used.

Unsuccessful overrides imply that **none** of the default policies have been
modified. In this case, the operator should either fix and re-attach them to
the charm or disable the overrides entirely (i.e. set ``use-policyd-overrides``
to 'false').

Information on broken overrides will appear in the logs.

Examples
++++++++

This area contains examples of policy override usage.

Showing extended server attributes
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This example involves changing the default policy affecting the
nova-cloud-controller application.

Ordinarily, when a non-admin user requests details for a cloud instance some
fields are not shown. This is because some information is deemed inappropriate
or too sensitive for the regular user. For instance, this is the (partial)
default output to the :command:`openstack server show` command:

.. code-block:: console

   echo $OS_USERNAME
   User1

   openstack server show 9167b3e9-c653-43fc-858a-2d6f6da36daa

   +-----------------------------+----------------------------------------------------------+
   | Field                       | Value                                                    |
   +-----------------------------+----------------------------------------------------------+
   | OS-DCF:diskConfig           | MANUAL                                                   |
   | OS-EXT-AZ:availability_zone | nova                                                     |
   | OS-EXT-STS:power_state      | Running                                                  |
   | OS-EXT-STS:task_state       | None                                                     |
   | OS-EXT-STS:vm_state         | active                                                   |
   | OS-SRV-USG:launched_at      | 2019-12-11T23:09:47.000000                               |
   | OS-SRV-USG:terminated_at    | None                                                     |

Compare that output to what an admin sees:

.. code-block:: console

   echo $OS_USERNAME
   admin

   openstack server show 9167b3e9-c653-43fc-858a-2d6f6da36daa

   +-------------------------------------+--------------------------------------------------+
   | Field                               | Value                                            |
   +-------------------------------------+--------------------------------------------------+
   | OS-DCF:diskConfig                   | MANUAL                                           |
   | OS-EXT-AZ:availability_zone         | nova                                             |
   | OS-EXT-SRV-ATTR:host                | virt-node-01.maas                                |
   | OS-EXT-SRV-ATTR:hypervisor_hostname | virt-node-01.maas                                |
   | OS-EXT-SRV-ATTR:instance_name       | instance-00000001                                |
   | OS-EXT-STS:power_state              | Running                                          |
   | OS-EXT-STS:task_state               | None                                             |
   | OS-EXT-STS:vm_state                 | active                                           |
   | OS-SRV-USG:launched_at              | 2019-12-11T23:09:47.000000                       |
   | OS-SRV-USG:terminated_at            | None                                             |

The admin user has three extra fields that are categorised as *extended server
attributes*:

.. code-block:: console

   | OS-EXT-SRV-ATTR:host                | virt-node-01.maas                                |
   | OS-EXT-SRV-ATTR:hypervisor_hostname | virt-node-01.maas                                |
   | OS-EXT-SRV-ATTR:instance_name       | instance-00000001                                |

For some environments, such as an internal company cloud, the benefits of
providing this information to users may outweigh any perceived concerns. For
example, users will know immediately whether an announced hypervisor
maintenance procedure will affect their running instances, providing that the
announcement includes the hypervisor name.

To make this happen the default policy affecting the `Nova API`_ will need to
be overridden to include the owner of the instance as well as the admin. The
policy "target" that controls these particular fields is
``os_compute_api:os-extended-server-attributes``.

The final policy statement is placed in a file, say,
``nova-server-attributes.yaml``:

.. code-block:: ini

   {
       #"os_compute_api:os-extended-server-attributes": "rule:admin_api"
       "os_compute_api:os-extended-server-attributes": "rule:admin_or_owner"
   }

The default statement is left as a comment in order to provide some extra
context.

Compress the file, attach it as a resource to the nova-cloud-controller
application, and enable the override:

.. code-block:: console

   zip nova-server-attributes.zip nova-server-attributes.yaml
   juju attach-resource nova-cloud-controller policyd-override=nova-server-attributes.zip
   juju config nova-cloud-controller use-policyd-override=true

Any non-admin user should now have access to three extra fields when querying
the instances that they own with the :command:`openstack server show` command.

More extended attributes can be displayed through the use of option
``--os-compute-api-version``. For example:

.. code-block:: console

   openstack --os-compute-api-version 2.3 server show 9167b3e9-c653-43fc-858a-2d6f6da36daa

See the upstream documentation on `Show Server Details`_.

.. LINKS
.. _Nova API: https://docs.openstack.org/nova/latest/configuration/policy.html
.. _Show Server Details: https://docs.openstack.org/api-ref/compute/?expanded=show-server-details-detail#show-server-details
