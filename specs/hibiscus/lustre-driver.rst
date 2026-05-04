..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================
Lustre share driver for Manila
================================

https://blueprints.launchpad.net/manila/+spec/lustre-driver

Lustre is the dominant parallel filesystem in high-performance computing,
running on the majority of the world's largest supercomputers and
increasingly serving as checkpoint and scratch storage for large-scale
AI/ML training. This spec proposes adding ``LUSTRE`` as a supported
share protocol in Manila, referring to the native Lustre client
protocol (as distinct from accessing Lustre via NFS or SMB gateways).
The driver maps Manila shares to Lustre subdirectories with project
quotas, and access rules to Lustre nodemaps for IP-based tenant
isolation.


Problem description
===================

Manila supports several share protocols (NFS, CIFS, CephFS, GlusterFS,
HDFS, MAPRFS) but has no support for Lustre, the most widely deployed
parallel filesystem in HPC (running on over 60% of the TOP500
supercomputers `[1]`_). All four major cloud providers (AWS FSx for
Lustre, Google Cloud Managed Lustre, Azure Managed Lustre, and Oracle
OCI File Storage with Lustre) now offer managed Lustre services.
OpenStack operators running Lustre alongside OpenStack have no
supported integration path through Manila.

At least six organizations have built manual Lustre-on-OpenStack
integrations (Wellcome Trust Sanger Institute `[2]`_, University of
Cambridge `[3]`_, Monash University `[4]`_, DKRZ `[5]`_, ETH Zurich
`[6]`_, and the FENIX European HPC consortium `[7]`_), all reinventing
the same patterns: subdirectory isolation, nodemap-based access
control, and manual quota management. A prior attempt at a Cinder
(block storage) driver for Lustre was abandoned because the block
model is the wrong abstraction for a shared POSIX filesystem.
Manila's shared filesystem model maps naturally to Lustre primitives.


Use Cases
=========

* An HPC cloud operator wants to offer self-service Lustre storage to
  tenants. Today they manually create directories, set quotas, and
  configure nodemaps for each project. With a Manila driver, tenants
  provision and manage their own Lustre shares through the standard
  Manila API.

* An AI/ML researcher running distributed GPU training needs isolated,
  quota-enforced shared storage for checkpoints. They create a Manila
  share with ``share_proto=LUSTRE`` and mount it on their training
  nodes without administrator intervention. Lustre is also
  increasingly used for persistent home directories, where
  researchers share the same filesystem across interactive sessions
  and training runs.

* A Kubernetes operator using the manila-csi-plugin wants to provision
  Lustre-backed PersistentVolumeClaims from standard StorageClass
  definitions. The Lustre driver enables this without custom
  integration.

* A deployer runs both a batch scheduler (e.g. Slurm) and OpenStack
  on the same Lustre filesystem. Manila manages the OpenStack tenant
  shares through separate nodemaps and project quotas while the batch
  scheduler manages HPC allocations on the same filesystem.


Proposed change
===============

Add ``LUSTRE`` to the set of supported share protocols in Manila and
implement a new share driver that operates in DHSS=False mode on a
pre-existing Lustre filesystem. The driver manages subdirectories,
project quotas, and nodemaps using standard Lustre CLI tools (``lfs``
and ``lctl``).

Lustre has three service types: the MGS (Management Service) holds
filesystem configuration including nodemaps, the MDS (Metadata
Service) handles the namespace and quotas, and the OSS (Object
Storage Service) stores file data. Nodemap commands
(``lctl nodemap_*``) must run on the MGS, and quota commands run on
the MDS. If the manila-share host is co-located with the MGS/MDS,
commands run locally via oslo.privsep. If the MGS or MDS is on a
separate host, the driver uses SSH, similar to how the LVM and ZFS
drivers administer remote storage nodes.

The mapping between Manila and Lustre concepts is:

* **Share** -- a subdirectory (fileset) under a configurable path
  prefix on the Lustre filesystem.
* **Share size** -- a Lustre project quota assigned to the
  subdirectory's project ID, enforced by the filesystem.
* **Access rule** -- a Lustre nodemap that maps client IP addresses
  (NIDs) to UID/GID squash settings, controlling who can access the
  subdirectory. The driver uses ``access_type='ip'`` because Lustre
  nodemaps operate at the network level. User-based access would
  require Kerberos integration, which is listed as future work.
* **Export location** -- a standard Lustre mount specifier in the
  format ``<IP>@<nid_type>:/<fsname>/<prefix>/<share_id>``.

The driver supports share creation, deletion, extend, shrink, and
IP-based access control (read-write and read-only). Capacity is
reported by parsing ``lfs df`` output.

The following configuration options are introduced in the driver's
backend section:

* ``lustre_share_export_ip`` -- IP or hostname used in export
  locations (typically the MGS NID or a DNS name resolving to it)
* ``lustre_mgs_ip`` -- MGS hostname for remote nodemap operations
  (optional; if unset, commands run locally via oslo.privsep)
* ``lustre_mds_ip`` -- MDS hostname for remote quota operations
  (optional; if unset, commands run locally via oslo.privsep)
* ``lustre_mount_point`` -- local mount point of the Lustre filesystem
* ``lustre_fs_name`` -- Lustre filesystem name
* ``lustre_share_path_prefix`` -- subdirectory prefix for Manila shares
* ``lustre_project_id_start`` / ``lustre_project_id_end`` -- project ID
  range for quota allocation
* ``lustre_nid_type`` -- LNet NID type (tcp, o2ib, etc.), used for
  constructing export location strings and NID ranges in nodemap
  rules. The driver is transport-agnostic; whether the underlying
  fabric uses TCP or InfiniBand (o2ib), and whether LNet routers are
  needed, is a deployment topology decision outside the driver's
  scope.

The driver requires Lustre 2.16 or later. This version introduced
RBAC nodemaps and root project quota enforcement, which prevent
tenants with root access inside VMs from escaping their quota
allocation.

Alternatives
------------

* **Continue manual integration.** Organizations can keep building
  custom glue between Lustre and OpenStack, leading to fragmented,
  unmaintainable, one-off solutions.

* **NFS gateway instead of native Lustre protocol.** Export Lustre
  via NFS-Ganesha and use existing NFS drivers. This introduces a
  single-point bottleneck, sacrifices Lustre's parallel I/O
  performance, and loses features like client-side striping and
  project quotas.

* **Cinder block driver.** Previously attempted by DDN and abandoned.
  The block model does not support shared multi-tenant POSIX access.

* **DHSS=True mode.** Creating entire Lustre filesystems per share is
  infeasible because Lustre filesystem provisioning involves deploying
  MGS, MDS, and OSS servers. This could be explored as future work
  using containerized or virtualized Lustre servers.

Data model impact
-----------------

No new database tables or columns are required. The only change is
adding ``LUSTRE`` to the ``SUPPORTED_SHARE_PROTOCOLS`` constant. The
driver uses Manila's existing ``private_storage`` mechanism to persist
per-share metadata such as the Lustre project ID.

REST API impact
---------------

No new API endpoints or microversions are introduced. The existing
share creation and management endpoints will accept
``share_proto='LUSTRE'`` as a valid protocol value when the deployer
adds ``LUSTRE`` to the ``enabled_share_protocols`` configuration
option.

Export locations for Lustre shares use the standard Lustre mount
specifier format, for example:
``10.0.0.1@tcp:/scratch/manila_shares/abc-123``.

Access rules use the existing ``access_type='ip'`` with
``access_level='rw'`` or ``'ro'``. No new access types are introduced.

Driver impact
-------------

A new self-contained driver is added. No changes to the
``ShareDriver`` base class, ``ShareManager``, or any existing drivers
are required.

The driver supports: share creation, deletion, extend, shrink,
IP-based access control (grant and revoke), capacity reporting, and
share ensure (re-verification on service restart).

Not supported in the initial implementation: snapshots, share
replication, manage/unmanage of existing shares, and share groups.
These can be added as future enhancements. Lustre does not have
native per-share snapshot support. Server-level snapshots (e.g., ZFS
or LVM snapshots of individual Lustre servers) are possible but are
not scoped to individual shares. Manila's backup API can be used for
data protection workflows once backup support is added to the driver.

Security impact
---------------

When running locally, the driver executes ``lfs`` and ``lctl``
commands with elevated privileges using oslo.privsep. When the MGS or
MDS is on a remote host, the driver uses SSH with key-based
authentication, configured via the ``lustre_mgs_ip`` and
``lustre_mds_ip`` options.

Lustre client mounts do not use credentials for authentication.
Access is controlled by nodemaps, which are NID-based (IP-based).
This is the same trust model as IP-based NFS access control already
used in Manila.

The ``root_prj_enable`` filesystem parameter (Lustre 2.16+) prevents
tenants with root access inside VMs from exceeding their project
quota allocation. This is a hard requirement for the driver.

Notifications impact
--------------------

None. The driver uses existing Manila share lifecycle notifications.

Other end user impact
---------------------

Users specify ``--share-protocol LUSTRE`` when creating shares via
python-manilaclient. No client-side code changes are needed.

Compute nodes that need to mount Lustre shares must have the Lustre
client packages installed and the appropriate LNet network configured.
This is a deployment prerequisite analogous to the CephFS native
driver requiring Ceph client packages.

Performance Impact
------------------

The driver executes a small number of subprocess calls per share
operation, comparable to other subprocess-based drivers in Manila.

Other deployer impact
---------------------

Deployers must:

* Install Lustre client packages on the manila-share host.
* Mount the target Lustre filesystem at the configured mount point.
* Add ``LUSTRE`` to the ``enabled_share_protocols`` configuration.
* Configure a backend section with the Lustre-specific options.
* Create a share type with ``share_protocol=LUSTRE`` and
  ``driver_handles_share_servers=False``.

Network connectivity between tenant VMs and the Lustre filesystem
(including any LNet routers needed to bridge tenant networks to the
storage fabric) is a deployment prerequisite managed by the operator,
not Manila. This is the same position as the CephFS DHSS=False driver
with regard to Ceph network connectivity.

This change takes effect only when explicitly configured. It is purely
additive and has no impact on existing deployments.

Developer impact
----------------

No new Python library dependencies are introduced. The driver uses
only subprocess calls to standard Lustre CLI tools.

A ``devstack-plugin-lustre`` project will be created to automate
provisioning of a Lustre cluster for CI and local development. A CI
job built on this plugin will be committed alongside the driver.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  gouthamr

Other contributors:
  None

Work Items
----------

* Add ``LUSTRE`` to ``SUPPORTED_SHARE_PROTOCOLS`` in Manila.
* Implement the Lustre share driver.
* Add unit tests for the driver.
* Add driver configuration reference documentation.
* Add admin guide section for deploying a Lustre backend.
* Create ``devstack-plugin-lustre`` for CI and local development.
* Add CI job for Lustre driver testing.
* Add release note for the new driver and protocol.

Future work (out of scope for initial implementation):

* Snapshot support. Lustre does not have native per-share snapshots
  today. Potential approaches include server-level ZFS snapshots or
  a future Lustre-native mechanism.
* DHSS=True mode where Manila provisions Lustre services (MGS/MDS/OSS)
  on demand, for example using containerized or virtualized Lustre
  servers.
* Manage/unmanage support for adopting existing subdirectories.
* Dynamic nodemap support (Lustre 2.17+) for ephemeral VM
  environments.
* Kerberos authenticated mounts for subtree exports (user-based
  access control).
* Striping configuration via share type extra specs (stripe_count,
  stripe_size).


Dependencies
============

* Lustre client packages (lustre-client) on the manila-share host.
  This is an external system dependency.
* Lustre filesystem version 2.16 or later.


Testing
=======

Unit tests will cover all driver operations by mocking subprocess
calls. These run in the standard upstream gate.

Tempest tests for share creation, deletion, extend, shrink, and access
control with ``share_protocol=LUSTRE`` will run in a CI job that uses
``devstack-plugin-lustre`` to provision a Lustre cluster.


Documentation Impact
====================

* Configuration reference: add Lustre driver options.
* Admin guide: deployment prerequisites and backend configuration.
* User guide: ``LUSTRE`` protocol, export location format, and client
  mount instructions.
* Feature support matrix: Lustre driver capabilities.
* Release notes.


References
==========

.. _`[1]`: https://top500.org/lists/top500/2025/06/
.. _`[2]`: https://hpc-news.sanger.ac.uk/wp-content/uploads/2021/05/Lustre-for-openstack-whitepapers-version-2.0.pdf
.. _`[3]`: https://www.openstack.org/summit/barcelona-2016/summit-schedule/events/16548/lustre-integration-for-hpc-on-openstack-at-cambridge-and-monash
.. _`[4]`: https://wiki.lustre.org/images/6/6e/LUG2018-Massive_Monash_University-Tan.pdf
.. _`[5]`: https://superuser.openinfra.org/articles/managed-fishos-openstack-for-climate-research-dkrzs-proven-cloud-success/
.. _`[6]`: https://www.eofs.eu/wp-content/uploads/2024/02/10_diego_moreno_ethz_simplified_mt_for_personalized_health.pdf
.. _`[7]`: https://fenix-ri.eu/news/lustre-integration-openstack-completed

* Lustre project: https://www.lustre.org/
* Lustre multi-tenancy (LUG 2025, S. Buisson):
  https://srcc.stanford.edu/sites/g/files/sbiybj25536/files/media/file/lug25-lustre_multitenancy-buisson_v1.2.pdf
* StackHPC lustre-tools for nodemap automation:
  https://github.com/stackhpc/lustre-tools
* DDN Cinder driver blueprint (abandoned):
  https://blueprints.launchpad.net/cinder/+spec/add-lustre-driver
