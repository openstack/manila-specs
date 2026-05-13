..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================
Share Server Replica
====================

Blueprint: https://blueprints.launchpad.net/manila/+spec/share-server-replica

Operators need a Manila-native way to manage disaster recovery at the share
server layer. Today, share server replication feature is not available in
Manila, which makes failover orchestration, operational visibility, and
day-2 administration difficult. This specification introduces a first-class
share server replica resource in Manila so administrators can create, inspect,
update, promote, resync, and delete share server replicas through standard APIs
and client commands.


Problem description
===================

Share-level replication already exists in Manila, but share-server-level
replication is not available in Manila. That leaves operators without a
consistent API/CLI for managing failover-oriented workflows at the
share-server layer.

The current gap has a few consequences:

* Operators must rely on backend-specific procedures for replica lifecycle.
* Failover behavior is harder to automate and document.
* Share server replica status and share server replica state are not visible
  as a first-class Manila resource.
* Managing control-path movement during promotion is challenging when the
  relationship is established through the backend.


This specification addresses that gap by adding a Manila-managed share server
replica resource.


Use Cases
=========

1. An administrator wants to create a sync or async policy based replica of an
   active share server on a target backend host.

2. An administrator wants to delete a share server replica that is no longer
   needed, or force-delete one that is in an error state.

3. An administrator wants to promote an in-sync share server replica to
   active during a failure event or planned maintenance.

4. An administrator wants to resync a non-active share server replica after
   recovery so that it can be used again as a standby.

5. An administrator wants to list and inspect share server replica status, state,
   host, and replication policy for troubleshooting and monitoring.



Proposed change
===============

Below are the Proposed changes to support the share server replication.

* Create a share server replica with metadata for a share server on a target
  availability zone.
* List share server replicas, optionally filtered by source share server.
* Show a share server replica.
* Update share server replica metadata, replica state, and status.
* Delete a share server replica, with force support to delete the share server
  replica and purge it from Manila, even if the backend hasn't been cleaned
  up.
* Promote a share server replica to active.
* Resync a share server replica with its active source.
* Show if a share instance is part of share server replication or not.

This addition will consist of new database tables, share API
validation, manager orchestration, driver interfaces, policy rules,
api-ref documentation, and python-manilaclient support.

Implementation Details
----------------------

manila.conf
~~~~~~~~~~~~

Storage backends that are capable of replicating share servers between each
other must be in the same replication domain and it must be represented in
manila.conf through the ``replication_domain`` flag.

.. code-block:: ini

    # Backend A
    [backend-a]
    replication_domain = replication_domain-A

    # Backend B (can replicate with A)
    [backend-b]
    replication_domain = replication_domain-A


Configuration:

* Deployments enabling share server replication must configure
  ``replication_domain`` on each participating backend.
* Backends without this capability are not considered valid destinations
  for share server replica create operations.

Share server replica create
~~~~~~~~~~~~~~~~~~~~~~~~~~~

At DB, API and manager level, the expected sequence is:

Validation

* The share server must exist and have an ``active`` status.
* Fanout will not be supported, if a non-active replica exists,
  new replica creation requests will receive ``HTTP 400``.
* If an availability_zone is provided, make sure the AZ has a valid
  share network/subnet.


Destination Share Server Creation, Status, and State Updates

* Add a database record for the destination share server in the
  ``share_servers`` table with status set to ``creating``.
* Update the replica state to ``out_of_sync``.
* Update source replica state in the ``share_servers`` table to ``active``.


Make Driver Call

* The driver create call is issued with the replica, replica list, and all share
  instances under the source share server details.

On success:

* Update the destination share server status in the ``share_servers`` table to
  ``inactive``. Because the destination share server will be in an inactive
  state, it will be excluded from share instance creation.
* The share server replica status should either match the share server status or
  be derived and calculated from it.
* Update the ``source_share_server_id`` for destination share server in
  ``share_servers`` table
* Update the share server quota.

On failure:

* Set the replica share server status to ``error`` in the ``share_servers``
  table.


Delete replica
~~~~~~~~~~~~~~

At API and manager level, the expected sequence is:

* If share server replica is not found, raise not-found exception.
* Deleting an active share server replica should show proper error message.
  The operator must promote the non-active replica before deleting the
  primary active replica.
* The share server status of replica is marked ``deleting`` in DB.
* When deleting a share server replica with ``force=True``, manager will try
  to delete the share server replica via driver and irrespective of success/failure
  the manila database entry for replica share server will be deleted.
* Users should be able to delete the active replica with the force flag if there
  are no replicas associated with it. This call will not make a driver call, it
  will only delete the active replica records from the database.

Make Driver Call

* Driver delete call is issued with source and destination share server details.
* The driver is responsible for cleaning up all artifacts at the destination
  site, including deleting relationships and removing all objects created
  through those relationships.

On success:

* Replica share server is deleted.
* Share server quota is updated.

On failure:

* Share server replica status is set to ``error`` and ``replica_state`` set to
  ``out_of_sync``.
* Share server replica status is set to ``error_deleting`` and ``replica_state``
  set to ``error`` if backend call raises an exception.


Resync replica
~~~~~~~~~~~~~~

At API and manager level, the expected sequence is:

* Share server replica must exist and available status.
* Share server replica must not be active.
* Manager updates DB after driver returns.
* Manager will have a method to poll the replication status on a configurable
  time interval.

On success:

* The share server replica status should be set to ``available`` and the
  ``replica_state`` should be set to ``in_sync``.

On failure:

* Share server replica status is set to ``error`` and replica state is set to
  ``out_of_sync``.


Replica promote
~~~~~~~~~~~~~~~

At API and manager level, the expected sequence is:

* Share server replica must exist.
* Share server replica must not be active.
* Manager loads target share server replica and all replicas for the same
  share server.
* Manager calls driver promote.

On failure:

* Share server replica status is set to ``error`` and share server replica
  state is set to ``out_of_sync``.

On success:

* Promoted share server replica is set to ``active`` and ``available``.
* Source share server replica is set to ``out_of_sync`` and ``available``.
* Update the share server ID and host for all the share instances.
* Update the share server ID for the share group.
* The source share server status is set to ``inactive`` and the promoted
  replica's share server is set to ``active``.
* Update the ``source_share_server_id`` to the promoted share server.
* Update the ``source_share_server_id`` to null for the source share server.
* Update the share network and security services for both the promoted and
  source share servers.



Unplanned failover workflow
---------------------------

1. An ``unplanned_failover_check_interval`` method will be added to the manager,
   which will be invoked at a configurable periodic interval.

2. The manager makes a call to the driver and checks if any failover has
   occurred. The driver needs to add logic to detect if a failover happens
   in the backend and return the failover details to the manager.

3. If failover is reported, Manila flips control-plane state. Manager does all
   of the following automatically:

   * Demote previous active share server replica state to ``out_of_sync``.
   * Mark new active share server replica as ``active``.
   * Update the share server status for the new share server replica to
     ``inactive`` and update the share server active share server replica
     status to ``active``.

4. Update all share instances with correct share server ID and host.
5. Update the share network and security service mapping with both
   source and destination share server.


Important behavior note:

* Automatic change is not instantaneous; it is detected on the next periodic
  poll cycle.
* Effective switchover latency is approximately equal to the configured value
  of ``share_server_failover_check_interval``.
* ``share_server_failover_check_interval`` will be a configurable
  ``manila.conf`` option. If not explicitly set, the default interval is
  ``300`` seconds.
* If share_server_failover_check_interval is set to -1, then it will disable
  the periodic job for ``check_for_unplanned_share_server_replica_failover``
  method.


Replica states
~~~~~~~~~~~~~~

Each share server replica has a ``replica_state`` value. This specification
uses the same state model used by Manila share replication:

* ``active``: The replica currently serving as the active copy.
* ``in_sync``: A passive replica that is fully synchronized with the active
  replica and can be promoted.
* ``out_of_sync``: A passive replica that is not fully synchronized (including
  newly created replicas before initial sync completes).
* ``error``: The replica has encountered an unrecoverable condition and needs
  administrator intervention.

These states are independent of the operational ``status`` field
(``creating``, ``available``, ``error``, ``deleting``, ``error_deleting``,
etc.) and should be interpreted together during troubleshooting and failover
operations.

Quota and API guardrails
------------------------

* Share server replica create operations must enforce project quota.
  A new quota key, ``share_server_replicas``, will be checked at create time
  with a default limit of ``10`` replicas per project.
* Quota accounting should be released when share server replica
  delete completes.
* API layer should validate conflicting conditions before proceeding, including
  share-server operations that are incompatible with existing replica
  relationships.


Alternatives
------------

1. Leave share server replica management entirely inside backend
   drivers. That keeps the feature fragmented and inconsistent
   across deployments.

2. Reuse the share replica resource. That resource models share-level
   behavior, not share-server-level control-path movement and backend DR.

3. Expose the feature only through backend-specific extensions. That would
   reduce portability and make the feature much harder to document and test.

The proposed approach keeps the user-visible behavior in Manila while still
letting drivers implement backend-specific replication details.


Data model impact
-----------------

``share_servers`` table (existing table, new column added)::

    +------------------------+-------------+----------+-----------------------------------+
    | replica_state          | string(32)  | No       | active/in_sync/out_of_sync/error  |
    +------------------------+-------------+----------+-----------------------------------+

share_server_metadata::

    +-------------------------+-------------+----------+-----------------------+
    | Field                   | Type        | Nullable | Notes                 |
    +=========================+=============+==========+=======================+
    | id                      | string(36)  | No       | Primary key           |
    +-------------------------+-------------+----------+-----------------------+
    | share_server_id         | string(36)  | No       | FK to share servers   |
    +-------------------------+-------------+----------+-----------------------+
    | key                     | string(255) | No       | metadata key          |
    +-------------------------+-------------+----------+-----------------------+
    | value                   | string(1023)| No       | metadata value        |
    +-------------------------+-------------+----------+-----------------------+



An Alembic migration will create new table, add the ``replica_state``
column to ``share_servers``, and create the associated indexes.


CLI API impact
--------------

Add new OpenStackClient commands for share server replica management:

.. code-block:: bash

    openstack share server replica create \
        [--property <key=value>] \
        [--wait] [--availability-zone <availability-zone>] \
        <share-server>

    openstack share server replica delete [--force] <replica> [<replica> ...]

    openstack share server replica list [--share-server <share-server>]

    openstack share server replica show <replica>

    openstack share server replica set <replica> \
        [--replica-state <state>][--property <key=value>]

    openstack share server replica unset <replica> [--property <key=value>]

    openstack share server replica promote [--wait] <replica>

    openstack share server replica resync <replica>


REST API impact
---------------

The resource will be exposed under ``/v2/{project_id}/share-server-replicas``.

**Create share server replica**::

    POST /v2/{project_id}/share-server-replicas

Request::

    {
        "share_server_replica": {
            "share_server": "<share_server_uuid>",
            "availability_zone": "<availability_zone>",
            "property": {
                "replication_policy": "gold"
            }
        }
    }

Response(202)::

    {
      "share_server_replica": {
        "id": "server-uuid",
        "source_share_server_id": "source_share_server_id",
        "availability_zone": "<availability_zone>",
        "status": "creating",
        "replica_state": "out_of_sync",
        "created_at": "2026-06-09T12:00:00.000000",
        "property": {
          "replication_policy": "gold"
        }
      }
    }

The request should fail with ``400 Bad Request`` if required parameters are
missing.


**List share server replicas**::

    GET /v2/{project_id}/share-server-replicas

Optional query parameters include ``share_server_id``, ``sort_key``,
``sort_dir``, ``limit``, and ``offset``.


Response(200)::

    {
        "share_server_replicas": [
            {
                "id": "<share_server_uuid>",
                "source_share_server_id": "source_share_server_id",
                "host": "host@backend",
                "availability_zone": "<availability_zone>",
                "status": "available",
                "replica_state": "in_sync",
                "created_at": "2026-05-10T10:00:00Z"
            }
        ]
    }


**Show share server replica**::

    GET /v2/{project_id}/share-server-replicas/{share_server_replica_id}

Response(200)::

    {
        "share_server_replica": {
            "id": "<share_server_uuid>",
            "source_share_server_id": "source_share_server_id",
            "host": "host@backend",
            "availability_zone": "<availability_zone>",
            "status": "available",
            "replica_state": "in_sync",
            "created_at": "2026-05-10T10:00:00Z",
            "updated_at": "2026-05-10T11:00:00Z",
            "property": {
                "replication_policy": "gold"
            }
        }
    }


**Reset state of the share server replica**::

    POST /v2/{project_id}/share-server-replicas/{share_server_replica_id}/action

Request::

    {
        "reset_replica_state": {
            "replica_state": "out_of_sync"
        }
    }

Response(202)::

    None


**Delete share server replica**::

    DELETE /v2/{project_id}/share-server-replicas/{share_server_replica_id}

Replica with force delete::

    POST /v2/{project_id}/share-server-replicas/{share_server_replica_id}/action

Request::

    {
    "force_delete": null
    }

Deleting an active replica without force should fail if replica has a
non-active replica.


Response(202)::

    None


**Promote share server replica**::

    POST /v2/{project_id}/share-server-replicas/{share_server_replica_id}/action

Request::

    {
        "promote": {}
    }

Response(202)::

    None


**Resync share server replica**::

    POST /v2/{project_id}/share-server-replicas/{share_server_replica_id}/action

Request::

    {
        "resync": {}
    }

Response(202)::

    None


**Metadata API support for share server replica**::
    Shows, sets, updates, and unsets share server replica metadata.

Show all share server replica metadata::

    GET /v2/{project_id}/share-server-replicas/{share_server_replica_id}/metadata

Show share server replica metadata item::

     GET /v2/{project_id}/share-server-replicas/{share_server_replica_id}/metadata{key}

Set share server replica metadata::

    POST /v2/{project_id}/share-server-replicas/{share_server_replica_id}/metadata

Update share server replica metadata::

    PUT /v2/{project_id}/share-server-replicas/{share_server_replica_id}/metadata

Delete share server replica metadata item::

    DELETE /v2/{project_id}/share-server-replicas/{share_server_replica_id}/metadata{key}


Policy rules will be added for create, delete, show, list, update, promote,
and resync. All actions should remain admin-only by default.


Driver impact
-------------

There is no impact for drivers that do not want to support this feature.

Drivers that want to support share server replication must implement the
following driver methods in the share driver interface:

.. code-block:: python

    def create_share_server_replica(self, context, new_share_server_replica,
                                    share_server_replica_list):
        """Create a replica for an entire share server.

        Establish the backend replication relationship between the source
        (derived from ``share_server_replica_list``) and the destination
        (``new_share_server_replica``).
        Return None or a dict containing optional model updates for the new
        share server replica record. Backend-specific replication details
        should be returned under ``backend_details``. The returned
        ``state`` should be either ``'out_of_sync'`` or ``'in_sync'``;
        drivers should not report an error state through this field.
        """
        raise NotImplementedError()

    def delete_share_server_replica(self, context, share_server_replica,
                                    share_server_replica_list):
        """Delete a share server replica on the backend.

        Remove the replication relationship and clean up all destination
        objects.
        Return None or any exception raised will set the share server
        replica's ``status`` to ``'error_deleting'`` and its
        ``replica_state`` to ``'error'``.
        """
        raise NotImplementedError()

    def promote_share_server_replica(self, context, share_server_replica,
                                     share_server_replica_list,
                                     share_server_resources=None):
        """Promote a share server replica to active state.

        Perform the backend role reversal. Return ``None`` or a dictionary
        with ``replica_list`` (list of replicas with their ``replica_id``,
        ``status``, and ``replica_state`` updates) and
        ``instance_host_mappings`` (dictionary mapping share instance IDs to
        their new host locations after promotion).
        """
        raise NotImplementedError()

    def update_share_server_replica_state(self, context,
                                          share_server_replica,
                                          share_server_replica_list):
        """Return the current replication state for a share server replica.

        Called periodically by the manager and on manual refresh. Return
        ``replica_state``: a string value denoting the replica state.
        Valid values are ``'in_sync'`` and ``'out_of_sync'`` or
        ``None`` (to leave the current replica_state unchanged).
        Any exception raised will set the share server replica's ``status``
        to ``'error'`` and its ``replica_state`` to ``'error'``.
        """
        raise NotImplementedError()

    def  check_for_unplanned_share_server_replica_failover(self, context,
                                          share_server_replica_list,
                                          share_server_resources=None):
        """Check whether an unplanned failover happened in the backend.

        The manager calls this periodically to detect backend-driven
        failover events and update the control path accordingly.
        Return ``None`` or a dict with ``promote_required`` (boolean flag
        indicating if promotion is needed), (list of replicas with
        ``replica_id``, ``status``, and ``replica_state`` updates), and
        ``instance_host_mappings`` (dictionary mapping share instance
        IDs to their new host locations).
        """
        raise NotImplementedError()

Each method receives the target replica dict (containing an embedded
``share_server`` object), the full list of existing replicas for the same
share server, and the relevant share instances. All methods may return
``None`` or a dict with ``status`` and ``replica_state`` keys to influence
the final database state.

Drivers must also ensure that deleting a share server with active replica
relationships is blocked until those replicas are removed, and that share
deletion may require an explicit unprotect step when share server replication
is active.


Security impact
---------------

This feature does not introduce new secrets or credentials, but it does expose
new administrator-facing API actions that can change backend failover state.
Validation and RBAC enforcement are therefore important, especially for
promote, resync, and delete operations.

RBAC requirements:

* Cloud administrators must be able to execute all share server replica
  CLI commands and API endpoints.
* All other users must not have permission to execute share server
  replication-related CLI commands or APIs.


Notifications impact
--------------------
"share_server_replica.create", "share_server_replica.delete",
"share_server_replica.promote" and "share_server_replica.resync" notification
events will be emitted for the respective actions.


Other end user impact
---------------------

python-manilaclient will expose a share server replica manager and OSC command
set so that operators can use the feature from the standard OpenStack client.
This is the primary user-facing entry point for the new API.


Performance Impact
------------------

The feature adds database lookups for share server replica list
and show operations, and it adds backend calls for create, promote,
and resync. The performance impact should be limited to explicit
operator actions rather than periodic polling.


Other deployer impact
---------------------

Deployers will need to run the database migration before enabling the feature.
No additional configuration option is required for the baseline behavior.

Deployer documentation must include quota configuration for share server
replication. By default, Manila will enforce ``share_server_replicas = 10``
per project, and deployers can override this through quota configuration based
on capacity planning.


Developer impact
----------------

Driver authors will need to implement share server replica-specific
behavior if their backend is expected to support the new resource.
Client and API code must stay aligned with new microversion.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  gireesh(gawasthi2010@gmail.com)

Other contributors:
  manideep(manideep.openstack@gmail.com)


Work Items
----------

* Add the database migration and SQLAlchemy models for share server replicas,
  share_instance and share_servers.
* Add API controller, route registration, policy rules, and view builders.
* Add share API validation and manager orchestration.
* Add share server replica metadata support and quota enforcement.
* Add backend driver hooks for create, delete, promote, and resync.
* Add python-manilaclient v2 manager support and OSC commands.
* Add unit tests and tempest coverage for the new API.


Dependencies
============

* Manila API microversion 2.xx (``MIN_SUPPORTED_API_VERSION`` for all
  share server replica endpoints).
* Corresponding python-manilaclient support.
* Backend drivers that can implement server-level replication workflows.


Testing
=======

* Unit tests for API validation, policy enforcement, and manager behavior.
* Tempest tests for the create, list, show, update, delete, promote, and
  resync flows.
* Backend driver tests for share server replica lifecycle operations.


Documentation Impact
====================

API reference documentation will need new sections for the share server
replica resource and its action endpoint. User-facing docs should also explain
the microversion requirement and the expected administrator workflow.


References
==========

* https://etherpad.opendev.org/p/netapp-manila-active-sync-support
* https://wiki.openstack.org/wiki/Manila/design/manila-mitaka-data-replication
