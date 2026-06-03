..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================
Service Startup Reconciliation
================================

https://blueprints.launchpad.net/manila/+spec/service-startup-reconciliation

When the manila-share service stops ungracefully (OOM kill, SIGKILL,
node reboot, container eviction), resources that were mid-operation
remain stuck in transient states such as ``creating``, ``deleting``,
or ``extending``. This affects share instances, snapshot instances,
share replicas, share servers, share groups, and share backups.
Today, the only recovery is for an operator to manually run
``reset-state`` on each affected resource. This spec adds automatic
reconciliation: after startup, the share manager inspects transient
resources owned by its host, queries the storage backend for ground
truth, and either completes the operation, retries it, or marks the
resource as errored.


Problem description
===================

The share manager's ``ensure_driver_resources()`` method runs at
service startup to re-export shares and verify backend consistency.
However, it only processes share instances in ``available`` or
``ensuring`` status. Instances in any transient status are explicitly
skipped. This means a share stuck in ``creating`` after a crash will
remain stuck indefinitely, even though the backend may have completed
the operation. The same gap applies to snapshot instances, share
replicas, share servers, share groups, and share backups.

Consider a share instance stuck in ``creating``: the backend may have
created the share, but the service crashed before recording export
locations and setting ``available``. The operator must determine the
backend state manually, then delete or reset. Similarly, a share
stuck in ``deleting`` cannot be deleted again through the API, and a
share stuck in ``extending`` shows the wrong size and cannot be
resized.

There is precedent for handling stuck transient states: the replica
promotion code already sets stuck ``creating`` and ``deleting``
snapshot instances to ``error``.


Use Cases
=========

* An operator performs a rolling upgrade. Several resources happen to
  be mid-operation when service instances are restarted. After the
  upgrade, the operator expects these resources to have either
  completed or failed cleanly, without needing to audit and
  ``reset-state`` each one manually.

* An end user creates a share and the service crashes moments later.
  The user expects the share to eventually reach ``available`` or
  ``error``, not remain in ``creating`` indefinitely.


Proposed change
===============

A reconciliation pass runs in a deferred thread after service startup.
It scans all resources owned by the restarting host that are in
transient states and takes per-status corrective action based on what
actually exists on the storage backend.

Resources are reconciled in dependency order:

1. **Share servers** (DHSS=True only), because shares depend on them.
2. **Share instances**, the primary resources.
3. **Share replicas**, which depend on shares.
4. **Snapshot instances**, which depend on shares.
5. **Share groups and share group snapshots**.
6. **Share backups** (delete only; create and restore have their own
   periodic ``_continue`` tasks).

This ordering also allows incremental implementation: share servers
and share instances can land first; other resource types follow.

Reconciliation actions
----------------------

The reconciler takes one of three actions depending on the transient
status: **query** the backend for ground truth, **retry** a delete
(expected to be idempotent), or **mark error** when the operation
cannot be safely resumed.

**Query backend and converge**

Where a driver interface exists to report backend state, the
reconciler queries the backend before deciding the outcome.

.. list-table::
   :widths: 20 20 60
   :header-rows: 1

   * - Resource
     - Status
     - Action
   * - Share instance
     - ``creating``,
       ``creating_from_snapshot``
     - Call ``get_share_status()``. If the share exists and is
       healthy, set ``available`` and record export locations.
       If not found or error, set ``error``.
   * - Share instance
     - ``extending``,
       ``shrinking``
     - Call ``get_share_status()`` to read the current size.
       Update the database to match reality, set ``available``.
       On error, set ``extending_error`` or ``shrinking_error``.
   * - Snapshot instance
     - ``creating``
     - Call ``get_snapshot_status()``. Same pattern as shares.
   * - Share replica
     - ``creating``
     - Call ``get_replica_status()``. Same pattern as shares.
   * - Share group
     - ``creating``
     - Call ``get_share_group_status()``. Same pattern as
       shares.
   * - Share group
       snapshot
     - ``creating``
     - Call ``get_share_group_snapshot_status()``. Same pattern
       as shares.

All ``get_*_status()`` methods are optional. Drivers that do not
implement them raise ``NotImplementedError``, and the reconciler
falls back to setting the resource to ``error``.

**Retry delete (all resource types)**

For any resource stuck in ``deleting`` (or ``deferred_deleting`` for
share instances), the reconciler retries the corresponding driver
delete call. Delete operations are expected to be idempotent: if the
resource was already removed, the driver should succeed or indicate
the resource is gone. On success, mark ``deleted``. On failure, set
``error_deleting`` (or ``error`` for share servers).

**Mark error (all resource types)**

The remaining transient statuses cannot be safely resumed without
the original operation context. The reconciler sets an appropriate
error status and logs a warning.

.. list-table::
   :widths: 20 40 40
   :header-rows: 1

   * - Resource
     - Statuses
     - Error status
   * - Share instance
     - ``manage_starting``, ``unmanage_starting``
     - ``manage_error``, ``unmanage_error``
   * - Share instance
     - ``restoring``, ``reverting``,
       ``replication_change``,
       ``backup_creating``, ``backup_restoring``
     - ``error``
   * - Snapshot instance
     - ``manage_starting``,
       ``unmanage_starting``, ``restoring``
     - ``manage_error``, ``unmanage_error``,
       or ``error``
   * - Share server
     - ``creating``, ``managing``, ``unmanaging``,
       ``server_network_change``
     - ``error``, ``manage_error``, or
       ``unmanage_error``

Excluded statuses
-----------------

The following statuses are already handled by existing periodic tasks
and are not subject to this reconciliation:

* Share instances with an active ``task_state`` (migration
  micro-states) are skipped by the existing ``is_busy`` check.

* Share instances and share servers in ``server_migrating`` or
  ``server_migrating_to`` are handled by ``migration_continue``.

* Share backups in ``creating`` or ``restoring`` are handled by
  ``create_backup_continue`` and ``restore_backup_continue``.

Relationship to ensure shares
------------------------------

This reconciliation operates on a disjoint set of resources from
``ensure_driver_resources()`` and the ensure shares API.
``ensure_driver_resources()`` processes shares in ``available`` or
``ensuring`` status; this reconciliation processes shares in
transient statuses. Neither will act on a resource the other is
handling. The reconciliation thread starts after
``ensure_driver_resources()`` completes, but the two can also run
independently (e.g., if the ensure shares API is invoked later)
without interference.

Active/Active HA coordination
-----------------------------

In Active/Active deployments where multiple share managers serve the
same host, two managers could restart simultaneously and attempt to
reconcile the same transient instances. To prevent this, the
reconciler acquires a non-blocking try-lock using tooz for each
resource before acting on it. If the lock is held by another
manager, the resource is skipped. Holding the lock during driver
calls is acceptable because the resource is in a transient state and
the API rejects all other operations on it.

Deferred execution
------------------

Reconciliation runs in a separate thread spawned after
``ensure_driver_resources()`` completes. Transient statuses prevent
any API operation on the affected resources, so there is no race
between the service accepting new work and the reconciliation thread
processing stale instances.

Alternatives
------------

* **Set all transient resources to error.** Simple but lossy.
  Resources that completed on the backend would be incorrectly marked
  as errored.

* **Retry every transient operation.** Risks duplicate resources on
  backends that are not idempotent.

* **Periodic reconciliation loop.** A larger architectural change,
  better pursued as future work once the startup-only approach is
  proven.

Data model impact
-----------------

None.

REST API impact
---------------

None.

Driver impact
-------------

The reconciliation relies on one existing and several new optional
driver interface methods:

* ``get_share_status(share, share_server=None)``: Already defined in
  the base ``ShareDriver`` class and implemented by several drivers
  (NetApp, CephFS). Reconciliation extends its use to transient
  states at startup.

* ``get_snapshot_status()``, ``get_replica_status()``,
  ``get_share_group_status()``,
  ``get_share_group_snapshot_status()``: New optional methods
  following the same pattern as ``get_share_status()``. Each returns
  a dictionary with at minimum a ``status`` key. Drivers that do not
  implement them raise ``NotImplementedError``, and the reconciler
  falls back to setting the resource to ``error``.

* The various ``delete_*`` and ``teardown_server`` methods are already
  expected to be idempotent per their existing contracts.

Drivers can implement the ``get_*_status()`` methods incrementally to
enable smarter recovery for each resource type.

Security impact
---------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

Resources that previously remained stuck in transient states after a
service restart will now resolve to a terminal status. No client-side
changes are needed.

Performance Impact
------------------

Runs once at startup in a deferred thread. Zero cost if no resources
are stuck. No steady-state performance impact.

Other deployer impact
---------------------

Two new configuration options:

* ``startup_reconciliation_enabled`` (BoolOpt, default ``True``):
  Controls whether reconciliation runs at startup.

* ``startup_reconciliation_wait_seconds`` (IntOpt, default ``10``):
  Seconds to wait before beginning reconciliation, giving the driver
  time to complete initialization.

Developer impact
----------------

Developers adding new transient states should add corresponding
reconciliation logic.


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

* Implement a reconciliation pass that scans transient resources on
  this host, with non-blocking try-lock per resource for Active/Active
  coordination.
* Implement per-status reconciliation handlers for each resource type.
* Add the new ``get_*_status()`` optional driver interface methods.
* Run reconciliation in a deferred thread after
  ``ensure_driver_resources()`` completes.
* Add configuration options.
* Add unit tests for each resource type and transient status path.
* Add an ansible-based functional test using the dummy driver (see
  Testing section).
* Add a release note.


Dependencies
============

None. This work uses existing driver interfaces, database functions,
and the tooz coordination framework (already a dependency of Manila).

Related specifications:

* Ensure Share `[1]`_
* Ensure Shares API `[2]`_
* Mechanism to Prevent Race Conditions `[3]`_
* Per-Process Healthchecks `[4]`_


Testing
=======

Unit tests will cover each resource type and transient status
reconciliation path, including the Active/Active try-lock contention
case.

Full integration testing of crash recovery is difficult to simulate
in tempest because tempest tests cannot restart services mid-test.
Instead, a dedicated ansible-based functional test will be written
using the dummy driver. The test playbook will:

1. Deploy manila with the dummy driver backend.
2. Create shares, snapshots, and (for DHSS=True) share servers.
3. Use the admin ``reset-state`` command to place resources into
   various transient states, simulating a mid-operation crash.
4. Restart the manila-share service.
5. Assert that each resource transitions to the expected terminal
   status after reconciliation.

This runs in the standard CI gate without requiring real storage or
the ability to crash the service at a precise moment.


Documentation Impact
====================

* Admin guide: document the new configuration options and the startup
  reconciliation behavior.
* Release note for the new feature.


References
==========

.. _`[1]`: https://specs.openstack.org/openstack/manila-specs/specs/queens/ensure-share.html
.. _`[2]`: https://specs.openstack.org/openstack/manila-specs/specs/dalmatian/add-ensure-shares-api.html
.. _`[3]`: https://specs.openstack.org/openstack/manila-specs/specs/release_independent/mechanism-to-prevent-race-conditions.html
.. _`[4]`: https://review.opendev.org/c/openstack/manila-specs/+/987622

Future work
-----------

* **Periodic reconciliation**: Extend reconciliation to run on a
  timer (not just at startup) to catch resources that become stuck
  for reasons other than a service restart.
* **Heartbeat-based coordination**: For long-running operations, a
  heartbeat mechanism (using tooz leases) could distinguish between
  operations that are genuinely stuck and operations that are merely
  slow.
