================================
Special charm upgrade procedures
================================

The OpenStack charms are designed to accommodate every supported series and
OpenStack release wherever possible. However, upgrades may occasionally
introduce unavoidable challenges for a deployed charm. For instance, it could
be that a charm is replaced by an entirely new charm on the new series. This
can happen due to development policy concerning the charms themselves or due to
reasons independent of the charms (e.g. the workload software is no longer
supported on the new operating system). Any core OpenStack charms affected in
this way will be documented here:

* :doc:`percona-cluster charm: series upgrade to focal <percona-series-upgrade-to-focal>`
* :doc:`ceph charm: migration to ceph-mon and ceph-osd <app-ceph-migration>`
