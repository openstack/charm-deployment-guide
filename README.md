# OpenStack Charms Deployment Guide

[![tags][image-badge-cdg]][image-link-openstack-tags]

The [OpenStack Charms Deployment Guide][cdg] is the main source of information
for the usage of the [OpenStack Charms][openstack-charms].

## Building

To build the guide run this simple command:

    tox

The resulting HTML files will be found in the `deploy-guide/build/html`
directory. These can be opened individually with a web browser or hosted by a
local web server.

## Contributing

Documentation issues can be filed on [Launchpad][lp-bugs-cdg].

This repository is under version control and is managed via the [OpenStack
Gerrit system][gerrit-openstack] (see the [OpenDev Developer’s
Guide][opendev-dev-guide]). For specific guidance on working with the
documentation hosted on [docs.openstack.org][link] please read the [OpenStack
Documentation Contributor Guide][openstack-doc-guide].

<!-- LINKS -->

[image-badge-cdg]: http://governance.openstack.org/badges/charm-deployment-guide.svg
[image-link-openstack-tags]: http://governance.openstack.org/reference/tags/index.html
[cdg]: https://docs.openstack.org/project-deploy-guide/charm-deployment-guide
[openstack-charms]: https://launchpad.net/openstack-charms
[lp-bugs-cdg]: https://bugs.launchpad.net/charm-deployment-guide/+filebug
[gerrit-openstack]: https://review.openstack.org
[opendev-dev-guide]: https://docs.openstack.org/infra/manual/developers.html
[openstack-doc-guide]: https://docs.openstack.org/doc-contrib-guide/index.html
[link]: https://docs.openstack.org
