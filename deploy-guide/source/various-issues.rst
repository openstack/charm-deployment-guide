:orphan:

==============
Various issues
==============

This page documents various issues (software limitations/bugs) that may apply
to a Charmed OpenStack cloud. These are still-valid issues that have arisen
during the development cycles of past OpenStack Charms releases. The most
recently discovered issues are documented in the `Release Notes`_ of the latest
version of the OpenStack Charms.

The items on this page are distinct from those found on the following pages:

* the dedicated `Upgrade issues`_ page
* the `Special charm procedures`_ page

Lack of FQDN for containers on physical MAAS nodes may affect running services
------------------------------------------------------------------------------

When Juju deploys to a LXD container on a physical MAAS node, the container is
not informed of its FQDN. The services running in the container will therefore
be unable to determine the FQDN on initial deploy and on reboot.

Adverse effects are service dependent. This issue is tracked in bug `LP
#1896630`_ in an OVN and Octavia context. Several workarounds are documented in
the bug.

Adding Glance storage backends
------------------------------

When a storage backend is added to Glance a service restart may be necessary in
order for the new backend to be registered. This issue is tracked in bug `LP
#1914819`_.

.. LINKS
.. _Release Notes: https://docs.openstack.org/charm-guide/latest/release-notes.html
.. _Upgrade issues: upgrade-issues.html
.. _Special charm procedures: upgrade-special.html

.. BUGS
.. _LP #1896630: https://bugs.launchpad.net/charm-layer-ovn/+bug/1896630
.. _LP #1914819: https://bugs.launchpad.net/charm-glance/+bug/1914819
