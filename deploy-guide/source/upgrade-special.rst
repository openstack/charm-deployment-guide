========================
Special charm procedures
========================

The OpenStack charms are designed to accommodate every supported series and
OpenStack release wherever possible. However, upgrades (of either of the three
types) may occasionally introduce unavoidable challenges. For instance, it
could be that a charm is replaced by an entirely new charm on a new series or a
new charm is needed to accommodate a new service on a new OpenStack release.
These kinds of special procedures are documented here:

* :doc:`percona-cluster charm: series upgrade to focal <percona-series-upgrade-to-focal>`
* :doc:`placement charm: OpenStack upgrade to Train <placement-charm-upgrade-to-train>`
* :doc:`ceph charm: migration to ceph-mon and ceph-osd <app-ceph-migration>`
