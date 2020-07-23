..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================
Share Server Migration
======================

https://blueprints.launchpad.net/manila/+spec/share-server-migration

Manila supports the deployment model where share drivers are able to handle the
creation and the management of share servers as well as shares and their
capabilities[1]. By managing different share servers per tenant level, Manila
leverages its capability of configuring storage entities and provides more
manageability for administrators. As presented in Liberty release, and later
improved on Mitaka, Newton and Ocata releases, share migration operation allows
administrators to move a share across backends, in a non-disruptive manner, by
implementing a 2-phase migration approach. This spec now proposes to extend
this migration concept to the share server entity, relying on share drivers
that can do this operation in an atomic and efficient way.

Problem description
===================

Administrators might need to handle situations like back end evacuation or
rebalancing, and face the problem of migrating lots of shares, one by one, to
a specific, and probably common, destination. Even with additional tools or
scripts this task can be hard to manage and mainly, to recover from failure
states. The lack of a feature that helps administrators to rebalance/evacuate
large storage systems is the reason for proposing the following solution.

Use Cases
=========

There are several scenarios where share server migration comes handy and
provides benefits to cloud administrators:

* **Rebalance**: move shares to a back end that has more free capacity, freeing
  up space for other shares to grow over the time;
* **Optimization**: move shares and spare a back end in order to conserve
  power. Move data closer to the hosts for a better network performance;
* **Evacuation**: evacuate a back end that is too old or that is experiencing
  failures;
* **Maintenance**: move shares to a newer hardware version/model;
* **Others**: change shares' configuration like: share network,
  security services, etc.


Proposed change
===============

As designed for share migration on Newton release[2], the 2-phase migration
logic will be also implemented for share servers. By invoking
`share-server-migration-start`, the share server migration can start to copy
all data, from source to destination, including all shares, snapshots and
shares' access, if supported by the driver that implements it.

After finishing the 1st phase, administrators can plan and start the 2nd phase,
by invoking 'share-server-migration-complete' to finish the operation, that
usually causes the disruption of share's access, since share's export locations
might be updated.

It is important to note that when migrating a share server, many share
attributes won't be modified during the process, while share server attributes
might change depending on the provided parameters. Administrators will be able
to provide a new 'Share Network' to associate to the new share server, but
won't be able to change its shares' attributes like 'Share Type' since this
is a share level entity and different 'Share Types' can live in the same share
server.

Share API and Manager Changes
-----------------------------

The share API will hold all validations needed before proceeding with driver's
calls and database updates. The API will check if any of the shares within the
share server being migrated are in an invalid state or have any dependent
resource that cannot be migrated together with the share. The migration can
fail earlier if one of those validations cannot be satisfied.

Before starting the migration, the share server and all its shares will have
their status updated to reflect the operation that is being executed and to
block any other operation that could be triggered after this one started.
The source share server and all its shares will have their status updated to
``server_migrating`` while the destination share server will be updated to
``server_migrating_to``. By changing all shares' status, users will be
able to identify that a group of shares is blocked for receiving any other
operation.

After running through all validations with success, the share server's new
attribute called `task_state` will be updated to ``server_migration_starting``
and the scheduler will be invoked to validate if the host matches with the
provided share types.

By reaching share manager's migration start method, a driver's call will be
triggered to analyze if the destination back end can handle such operation
before starting the migration. If one of the required options can't be
satisfied, the migration will fail.

The share manager will update the share server's `task_state` to
``server_migrating`` and all its instances' status to ``server_migrating``. A
new share server might be requested in the destination back end to hold all the
data from source. It is expected that drivers will be able to identify that a
new server is being requested for migration purposes. After that, the driver
will be called to start the share server migration and to return immediately.

A share manager periodic task will continuously check share servers that have
the `task_state` set to ``server_migrating`` to invoke the driver's call
`share_server_migration_continue` to track the progress of share servers that
are in the 1st phase of the migration. After successfully finishing the 1st
phase, the share server `task_state` will be updated to
``server_migrating_phase1_done``.

Finally, share manager's `share_server_migration_complete` method can be
invoked for share servers that already completed the 1st phase, to finish the
migration. In this phase, the driver is called to finish the share server
migration and perform the last steps in the back end and return the list of
export locations for all its shares. The `task_state` of the share server is
set to ``server_migration_completed`` and all its shares have their export
paths updated before they become ``available`` again.

Before moving to the 2nd phase, during the data copy or at the 1st phase
completed, administrators can cancel the operation by invoking the
`share_server_migration_cancel` API. If supported by the driver, the cancel
operation will delete everything new that was created during the process, and
the share server and all its shares will go back to the initial state.

Scheduler Changes
-----------------

The scheduler filters can be used to validate if the destination host can
hold all shares associated to the share server being migrated. Share API will
need to provide the share server's total size along with all associated share
types' capabilities in order to validate if the destination host is suitable
for the new share server. However, the Scheduler won't be able to validate
share servers that spans across multiple pools, and for this type of scenario,
share server migration will need to rely on driver's checks to validate the
feasibility of such operation.

Alternatives
------------

The alternative is to use scripts or any other automation tool to move all
shares to a new destination, one by one, using share migration feature.

Data model impact
-----------------

A new field will be added to `Share Server` table to help tracking the states
of a share server migration. The new field `task_state` will work
like the same field that already exists on `Share` table. Administrator will be
able to reset the `task_state` by issuing the API
`share-server-reset-task-state`, as shown in the next section.

REST API impact
---------------

For admin-only, new API methods will be implemented:

1) `share-server-migration-start`

Migrates a share server::

  POST /share-servers/{share_server_id}/action

Body::

  {
    "migration_start": {
      "writable": true,
      "nondisruptive": true,
      "preserve_snapshots": true,
      "host": "host@dummy1#pool2",
      "new_share_network_id": "new_share_network_id"
    }
  }

The `host` contains the string host where the share server will be migrated to.
The capabilities `preserve_metadata`, `writable`, `nondisruptive` and
`preserve_snapshots`, if enabled, must be supported by the drivers that
implement such feature. If one of the capabilities isn't supported, the
migration will fail later in the driver's compatibility check.

By setting `writable` to ``true`` it's expected that all shares remain writable
during the first phase of the migration, where the data copy usually occurs.
However it doesn't guarantee that will remain ``writable`` during the second
phase, where the cutover usually happens for drivers that don't support a
`nondisruptive` migration.

By specifying `nondisruptive` equal to ``true``, the migration will be
performed without disrupting clients during the entire process, which usually
means that export locations won't be modified, and hence new network
allocations won't be made for the new share server.

If `preserve_snapshots` is set, it's expected that all snapshots from all
shares will be migrated together with the share server. If not supported by the
driver, users will need to consider unmanaging or deleting all snapshots
before proceeding with the migration.

The only optional parameters is 'new_share_network_id', which may need to be
provided to fit destination network requirements.

If the provided `share_server_id` doesn't exist, the API will respond with
``404 Not Found``. If one of the optional parameters is invalid or doesn't
exist, the API will respond with ``400 Bad Request``. If during the initial
validations in the Share API, one of the resources is busy or has an invalid
status, the API will respond with ``409 Conflict``.

Upon a failure, the share server and all its share will have their status
updated to ``available`` and their `task_state` set to
``server_migration_error``.

2) `share-server-migration-complete`

Start the 2nd phase of migration::

  POST /share-servers/{share_server_id}/action

Body::

  {"migration_complete": {}}

Triggers the start of the 2nd phase of migration on a share server that already
finished the 1st phase.

If the provided `share_server_id` doesn't exist, the API will respond with
``404 Not Found``.
If the operation can't be performed due to unsupported migration state, the API
will respond with ``400 Bad Request``.

Upon a failure in the second phase of the migration, the share server and all
its shares will have their status updated to ``error`` and their `task_state`
set to ``server_migration_error``. At this point, it won't be possible to
determine the status of the share server and its shares, and it will be up to
the administrator to manually fix this problem.

3) `share-server-migration-cancel`

Attempts to cancel migration::

  POST /share-servers/{share_server_id}/action

Body::

  {"migration_cancel": {}}

To cancel a migration in progress, the operation must not be in the 2nd phase
and the driver must support such operation.

If the provided `share_server_id` doesn't exist, the API will respond with
with ``404 Not Found``.
If the operation can't be performed due to unsupported migration state or
unsupported operation within the driver, the API will respond with
``400 Bad Request``.

After a successful migration cancellation operation, the share server and all
its shares will have their status updated to ``available`` and their
`task_state` set to ``server_migration_cancelled``.

4) `share-server-migration-get-progress`

Attempts to obtain migration progress::

  POST /share-servers/{share_server_id}/action

Body::

  {"migration_get_progress": {}}

Response::

  {"total_progress": 30}

Gives the current migration progress in a percentage value. Drivers might also
provide additional information together with `total_progress` info.

If the provided `share_server_id` doesn't exist, the API will respond with
``404 Not Found``.
If the provided `share_server_id` isn't performing a migration, the API will
respond with ``400 Bad Request``.

5) `share-server-reset-task-state`

Reset task state field value::

  POST /share-servers/{share_server_id}/action

Body::

  {
    "reset_task_state": {
      "task_state": "migration_error"
     }
  }

If the provided `share_server_id` doesn't exist, the API will respond with
``404 Not Found``.

6) `share-server-migration-check`

Check if a share server can be migrated to a destination host::

  POST /share-servers/{share_server_id}/action

Body::

  {
    "migration_check": {
      "writable": true,
      "nondisruptive": true,
      "preserve_snapshots": true,
      "host": "host@dummy1#pool2",
      "new_share_network_id": "new_share_network_id"
    }
  }

Response::

  {
    "compatible": true,
    "requested_capabilities": {
      "writable": true,
      "nondisruptive": true,
      "preserve_snapshots": true,
      "host": "host@dummy1#pool2",
      "new_share_network_id": "new_share_network_id"
    }
    "supported_capabilities": {
      "writable": true,
      "nondisruptive": false,
      "preserve_snapshots": true,
      "new_share_network_id": "new_share_network_id"
      "migration_cancel": true,
      "migration_get_progress" false,
    }
  }

Checks the feasibility of migrating a share server to a destination host.
Drivers will be able to check if the provided destination host can hold the
share server and which migration options will be available for this operation.

By answering `compatible` equal to ``true`` or ``false``, the admin will know
if the provided host is a feasible destination for the share server.

The migration options `writable`, `nondisruptive` and `preserve_snapshots`
show if the driver supports such options while migrating the share server.
If supported, the current share network or, if provided, the
`new_share_network_id` will also appear in the `supported_capabilities` field.

The migration operations `migration_cancel` and `migration_get_progress` may
also be available depending on the driver implementation.

Driver impact
-------------

Vendors that want to support share server migration must implement the
following interfaces:

* **choose_share_server_compatible_for_migration**: interface needed to tell
  the share manager which compatible share server can be used as destination
  in a migration operation;

* **share_server_migration_check_compatibility**: it will be always called
  before starting the migration to check if the driver supports migrating the
  share server to the required destination, and answer which kind of
  capabilities will be supported on such operation;

* **share_server_migration_start**: called to start the first phase of
  migration. The procedure should be started in the back end and return
  immediately.

* **share_server_migration_continue**: will be called to monitor the progress
  of a share server migration. Drivers will answer if the 1st phase was already
  finished or raise an exception in case of failure.

* **share_server_migration_complete**: starts the 2nd phase of the migration,
  to complete the operation by cutting over the access from the source and
  providing access through the destination.

* **share_server_migration_cancel**: drivers will implement this call if they
  support the cancellation of a migration operation that is already in
  progress. The migration cancellation won't be available for share servers
  that already started the 2nd phase;

* **share_server_migration_get_progress**: drivers will implement this call to
  provide the total progress of the migration.

As implemented in share migration approach, drivers will be invoked to check
the compatibility with the destination back end before starting the migration.
During this validation, drivers will be able to return the capabilities
supported for migrating a share server to the provided destination, such as
remaining writable, preserving snapshots and others.

After that, `share_server_migration_start` will take place and ask drivers to
start the 1st phase of the migration, that should be answered asynchronously.
Manila will reuse the same periodic task from share migration to continuously
check if the 1st phase is already completed by calling the driver interface
`share_server_migration_continue`.

Finally, the driver will need to perform the last steps to complete the share
server migration when the `share_server_migration_complete` is invoked. At this
moment, the access to the source share server shares may be interrupted,
depending on driver's capabilities, and moved to the new destination.

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

During the migration process users won't be able to perform any management
operation in all shares that belong to the share server being migrated.
Depending on driver's capabilities, users may also lose write access to those
shares.

Performance Impact
------------------

No performance impact is expected on implementing this feature. However,
depending on how many shares are placed within a share server, other
operations can be impacted due to the number of database operations triggered
by a share server migration, during sanity checks and status updates on all
affected resources (shares, snapshots, access, etc).

Other deployer impact
---------------------

Drivers that implement share server migration might need to retrieve the
configuration from other back ends in order to access it and provide a way of
copying all the data. Administrators will need to keep these files up to date
in all its share service instances.

Developer impact
----------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  dviroel

Work Items
----------

* Implement main patch that contains:
    * New API methods for share server migration;
    * New Scheduler call for share server migration start;
    * Share Manager implementation for share server migration;
    * Database updates for Share Server model;
    * New driver interfaces for migration of share servers.
* Update python-manilaclient with new share server's CLI commands.
* For testing:
    * Improve and implement both container and dummy drivers to support share
      server migration across different back ends.
    * New functional tests in manila-tempest-plugin.
* Documentation updates.

Dependencies
============

None.

Testing
=======

The container driver will need to be improved to support share server migration
across different back ends.

New functional tests will be added to perform share server migration on the
same back end and across different back ends. Vendors that implement support
for this feature will be encouraged to run these tests in their CI.

Documentation Impact
====================

The following documentation will be updated:

* API reference: Will update the Share Server API by adding the new actions for
  share server migration procedure.

* Admin reference: Will add information on how the functionality works and
  which drivers supports it.

* Developer reference: Will add information on how the new functionality works,
  and which interfaces need to be implemented.


References
==========

[1] https://docs.openstack.org/manila/ussuri/admin/shared-file-systems-share-server-management.html

[2] https://opendev.org/openstack/manila-specs/src/branch/master/specs/newton/newton-migration-improvements.rst

[3] https://etherpad.opendev.org/p/share-server-migration
