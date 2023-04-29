..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================
Allow locking shares against deletion
=====================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/manila/+spec/allow-locking-shares-against-deletion

The default RBAC permits a non-reader project user to create and delete
shares under the project's namespace. Deletion of shares can be dangerous,
and we expect that users exercise caution before initiating the action.
The Shared File Systems (Manila) API ensures that some pre-conditions are met
prior to proceeding with the deletion. A desirable pre-condition would be to
check if the share is actively being used by a client workload. Such a check
however is not straight forward to perform as it cannot reliably be
implemented in a consistent manner across all Network Attached Storage (NAS)
protocols or storage system back ends that Manila supports. In other words,
Manila does not know who and how many clients have mounted a share, or if data
is actively being read or written into the share. So there is a need for a
safety mechanism to prevent unintentional consequences.

This specification proposes a new pre-condition, one that allows any
non-reader project user to create a deletion lock against a share in the
project's namespace. The deletion lock can be removed by the same user, or
by a privileged user.

Problem description
===================

A shared file system is served by a network file server and it allows several
simultaneous clients to connect, read and write to it. In OpenStack, Manila
shares are collectively owned by the project users that the share belongs to.
A user in the project can delete a share that some other user is actively
using, and Manila API provides no way to indicate or coordinate
communication prior to this deletion.

Further, as part of the protocol, NAS clients are hardened to survive
minor network interruptions and server side failures within a degree of
tolerance. If the server goes unresponsive for a while, the client can wait,
filling up its write cache or retrying its reads until they succeed, or
until the tolerance expires. In the most common scenario, the client can be
instructed to wait indefinitely ("hard mounts"). So, an extended server
failure can be catastrophic to the client. If a file system that is mounted
is deleted on the server, the client typically exercises the same waiting
behavior and can go unresponsive in the process.

Use Cases
=========

The most recent use case is with the OpenStack Compute feature that allows
users to mount their shares to virtual machines via VirtIOFS `[1]`_. With
this feature, a compute host can plumb a mounted network filesystem to one or
more guests. If the share is deleted while it is mounted, the compute
host would be compromised. This would disable all virtual machines on the
host, not just the virtual machine that was using the share via VirtIOFS. So
while the Compute service orchestrates the mount, a user's action of
deleting the backing share can bring down the shared infrastructure.

Proposed change
===============

Users will have the ability to lock any share in the project. Multiple locks
can be placed on the share. A share cannot be deleted, unmanaged or
soft-deleted unless all locks have been removed. Only the user that placed
the lock, or the administrator user can remove a given lock. If a user
attempts to lock a share that is previously locked by them for a specific
resource action, the API will not re-create the lock. The lock record can be
updated with a new lock reason.

A service user (with the use of `X-Service-Token` and a `service` role) can
create a lock on a share on behalf of a regular user. A lock user context is
also recorded by manila in such a case. So, even when using a user's token,
if a service token is provided, the lock created cannot be removed by the
user. Only a service user, or a user with "admin" role can remove or update
the lock.

The implementation of this feature will include generalizations for future
extensibility. The lock API will accept a resource ID, resource type, and
a resource action that must be locked. In the 2023.2 Bobcat release cycle,
we will only be implementing deletion locks for shares.

Alternatives
------------

Shares could have an "in-use" state that could prevent adverse manipulation.
The presence of access rules can allow a share to transition to this
"in-use" status. The problem with this approach is that users could drain
access rules prior to deleting the share. This provides a two-step deletion
ensuring that the action is deliberate. However, in the use case above, it
wouldn't protect OpenStack Compute service resources from losing the share
gracelessly.

Data model impact
-----------------

A new table will be introduced::

    +-------------------+----------------------------------+----------+----------+
    |       Field       |               Type               | Nullable | Default  |
    +-------------------+----------------------------------+----------+----------+
    | ID                | varchar(36)                      | NO       | NULL     |
    | USER_ID           | varchar(36)                      | YES      | NULL     |
    | PROJECT_ID        | varchar(36)                      | YES      | NULL     |
    | RESOURCE_ACTION   | Enum('delete', ..)               | YES      | 'delete' |
    | RESOURCE_TYPE     | Enum('share', ..)                | YES      | NULL     |
    | RESOURCE_ID       | varchar(36)                      | NO       | NULL     |
    | LOCK_USER_CONTEXT | Enum('user', 'service', 'admin') | YES      | NULL     |
    | LOCK_REASON       | varchar(1023)                    | YES      | NULL     |
    | DELETED           | varchar(36)                      | YES      | 'false'  |
    | CREATED_AT        | datetime(6)                      | YES      | NULL     |
    | DELETED_AT        | datetime(6)                      | YES      | NULL     |
    | UPDATED_AT        | datetime(6)                      | YES      | NULL     |
    +-------------------+----------------------------------+----------+----------+


The table will assist storing lock records and will be manipulated as locks
are created, updated and removed. A database migration will create this table
with no initial data. The `LOCK_USER_CONTEXT` field will permit "user",
"service" and "admin" via enum. `RESOURCE_TYPE` and `RESOURCE_ACTION` fields
will also be limited by enum constants to supported resource types and
resource actions.

REST API impact
---------------

The APIs using resource lock endpoints and methods will only be available in a
new API micro version. However, if resource locks exist, they cannot be
circumvented by using an older API micro version to perform the action that
they are preventing.

**Create a resource lock on a particular action**::

    POST /v2/resource-locks

Normal http response code(s):

- 200 - Lock created successfully

Expected http error code(s):

- 401 - Unauthorized; user has not authenticated
- 400 - Bad Request
- 400 - Unrecognized action on resource
- 400 - Unrecognized resource (no such resource in project namespace)
- 403 - Forbidden; user is forbidden by policy
- 404 - API does not exist in micro version


Request example::

    {
        'resource_lock': {
            'resource_action': 'delete',
            'resource_type': 'share',
            'resource_id': 'a448e0d2-7501-4b99-a447-1b89e3961e39',
            'lock_reason': 'share is used by audit team'
        }
    }


Response example::

    {
        'resource_lock': {
            'id': 'be0871e8-742e-4c19-8567-7016fa0e2235',
            'user_id': 'cec1dd3e297b45348228f4fc3f5dba38',
            'project_id': '2e47ac4e2cf04a5b8b8509de8177d65d',
            'resource_action': 'delete',
            'resource_type': 'share',
            'resource_id': 'a448e0d2-7501-4b99-a447-1b89e3961e39',
            'lock_reason': 'share is used by audit team',
            'created_at': '2023-04-28T09:49:58-05:00',
            'updated_at': None
        }
    }


**Update a resource lock**::

    PUT /v2/resource-locks/{id}

Updatable fields include "resource_action" and "lock_reason".
"lock_reason" can be nullified on update. Only the user that created the
lock or a user with "admin" role will be allowed to update a lock per default
RBAC policy.

Normal http response code(s):

- 200 - Lock updated successfully

Expected http error code(s):

- 401 - Unauthorized; user has not authenticated
- 400 - Bad Request
- 400 - Unrecognized action on resource
- 403 - Forbidden; user is forbidden by policy
- 404 - API does not exist in micro version
- 404 - lock does not exist in project namespace

Request example::

    {
        'resource_lock': {
            'lock_reason': 'share will be used by audit team until 2024'
        }
    }


Response example::

    {
        'resource_lock': {
            'id': 'be0871e8-742e-4c19-8567-7016fa0e2235',
            'user_id': 'cec1dd3e297b45348228f4fc3f5dba38',
            'project_id': '2e47ac4e2cf04a5b8b8509de8177d65d',
            'resource_action': 'delete',
            'resource_type': 'share',
            'resource_id': 'a448e0d2-7501-4b99-a447-1b89e3961e39',
            'lock_reason': 'share will be used by audit team until 2024',
            'created_at': '2023-04-28T09:49:58.231919',
            'updated_at': '2023-04-28T20:01:13.12106'
        }
    }


**Delete a resource lock**::

    DELETE /v2/resource-locks/{id}

Only the user that created the lock or a user with "admin" role will be
allowed to delete a lock per default RBAC policy.

Normal http response code(s):

- 204 - Lock deleted successfully

Expected http error code(s):

- 401 - Unauthorized; user has not authenticated
- 400 - Bad Request
- 403 - Forbidden; user is forbidden by policy
- 404 - API does not exist in microversion
- 404 - lock does not exist in project namespace

Request and response do not contain any data

**List resource locks**::

    GET /v2/resource-locks?{queries}

Queries will allow filtering with exact and inexact ("created_since",
"created_before") attributes. Querying with "project_id" or "all_projects"
will only be allowed for a user with "admin" role per default RBAC policy.

Normal http response code(s):

- 200 - List of locks in project namespace

Expected http error code(s):

- 401 - Unauthorized; user has not authenticated
- 403 - Forbidden; user is forbidden by policy
- 404 - API does not exist in microversion

Response example::

    {
        'resource_locks': [
            {
                'id': 'be0871e8-742e-4c19-8567-7016fa0e2235',
                'user_id': 'cec1dd3e297b45348228f4fc3f5dba38',
                'project_id': '2e47ac4e2cf04a5b8b8509de8177d65d',
                'resource_action': 'delete',
                'resource_type': 'share',
                'resource_id': 'a448e0d2-7501-4b99-a447-1b89e3961e39',
                'lock_reason': 'share will be used by audit team until 2024'
            },
            {
                'id': '4945b04e-cdda-4308-9cfd-1483e7f9dd8c',
                'user_id': '80b789450540431db23575b333059ca8',
                'project_id': '2e47ac4e2cf04a5b8b8509de8177d65d',
                'resource_action': 'shrink',
                'resource_type': 'share',
                'resource_id': '4227fbd2-7f55-4ff4-9239-2cfc700d9fdf',
                'lock_reason': 'space is reserved for in place snapshots'
            }
        ]
    }

**Show lock**::

    GET /v2/resource-locks/{id}

Normal http response code(s):

- 200 - Details of a lock in the project namespace

Expected http error code(s):

- 401 - Unauthorized; user has not authenticated
- 403 - Forbidden; user is forbidden by policy
- 404 - API does not exist in micro version
- 404 - lock does not exist in project namespace

Response example::

    {
        'resource_lock': {
            'id': 'be0871e8-742e-4c19-8567-7016fa0e2235',
            'user_id': 'cec1dd3e297b45348228f4fc3f5dba38',
            'project_id': '2e47ac4e2cf04a5b8b8509de8177d65d',
            'resource_action': 'delete',
            'resource_type': 'share',
            'resource_id': 'a448e0d2-7501-4b99-a447-1b89e3961e39',
            'lock_reason': 'share will be used by audit team until 2024',
            'created_at': '2023-04-28T09:49:58.231919',
            'updated_at': '2023-04-28T20:01:13.12106'
        }
    }


**Deleting a share that has locks**::

    DELETE /v2/shares/{id}

Normal http response code(s):

- 202 - No locks exist and all other pre-conditions allow, accepted

Expected http error code(s):

- 401 - Unauthorized; user has not authenticated
- 403 - Forbidden; user is forbidden by policy
- 404 - API does not exist in microversion
- 404 - share does not exist in project namespace
- 409 - share deletion precondition failed, perhaps there's a lock

A "delete" lock will also prevent soft deletion or un-manage operations on
the share in a similar fashion.


**New RBAC policies will be introduced:**

.. code-block:: python

    """Policy defaults that are used in specific policies below:"""

    RULE_ADMIN = "role:admin"
    RULE_SERVICE = "role:service"
    RULE_ADMIN_OR_SERVICE = f'({RULE_ADMIN}) or ({RULE_SERVICE})'
    PROJECT_MEMBER = "rule:project-member"
    PROJECT_READER = "rule:project-reader"
    PROJECT_OWNER_USER = "rule:project-owner-user"

    ADMIN_OR_SERVICE_OR_PROJECT_MEMBER = f'({RULE_ADMIN_OR_SERVICE}) or ({PROJECT_MEMBER})'
    ADMIN_OR_SERVICE_OR_PROJECT_MEMBER = f'({RULE_ADMIN_OR_SERVICE}) or ({PROJECT_READER})'
    ADMIN_OR_SERVICE_OR_PROJECT_OWNER_USER = f'({RULE_ADMIN_OR_SERVICE}) or ({PROJECT_OWNER_USER})'

    rules = [
        policy.RuleDefault(
            name='project-member',
            check_str='role:member and '
                      'project_id:%(project_id)s',
            description='Project scoped Member',
            scope_types=['project']),
        policy.RuleDefault(
            name='project-reader',
            check_str='role:reader and '
                      'project_id:%(project_id)s',
            description='Project scoped Reader',
            scope_types=['project']),
        policy.RuleDefault(
            name='project-owner-user',
            check_str='role:member and '
                      'project_id:%(project_id)s and '
                      'user_id:%(user_id)s',
            description='Project scoped Member who owns a resource',
            scope_types=['project']),
    ]


* Create a lock

.. code-block:: python

   policy.DocumentedRuleDefault(
        name= 'resource_locks:create',
        check_str=ADMIN_OR_SERVICE_OR_PROJECT_MEMBER,
        scope_types=['project'],
        description="Create a resource lock.",
        operations=[
            {
                'method': 'POST',
                'path': '/resource-locks',
            },
        ],
    )

* Update a lock

.. code-block:: python

   policy.DocumentedRuleDefault(
        name= 'resource_locks:update',
        check_str=ADMIN_OR_SERVICE_OR_PROJECT_OWNER_USER,
        scope_types=['project'],
        description="Update a resource lock.",
        operations=[
            {
                'method': 'PUT',
                'path': '/resource-locks/{id}',
            },
        ],
    )


* Delete a lock

.. code-block:: python

   policy.DocumentedRuleDefault(
        name= 'resource_locks:delete',
        check_str=ADMIN_OR_SERVICE_OR_PROJECT_OWNER_USER,
        scope_types=['project'],
        description="Delete a resource lock.",
        operations=[
            {
                'method': 'DELETE',
                'path': '/resource-locks/{id}',
            },
        ],
    )


* List locks

.. code-block:: python

   policy.DocumentedRuleDefault(
        name= 'resource_locks:index',
        check_str=ADMIN_OR_SERVICE_OR_PROJECT_READER,
        scope_types=['project'],
        description="List all resource locks.",
        operations=[
            {
                'method': 'GET',
                'path': '/resource-locks?{queries}',
            },
        ],
    )

* List locks with project queries

.. code-block:: python

   policy.DocumentedRuleDefault(
        name= 'resource_locks:get_all_projects',
        check_str=RULE_ADMIN,
        scope_types=['project'],
        description="Get resource locks across projects.",
        operations=[
            {
                'method': 'GET',
                'path': '/resource-locks?all_projects=1&project_id={project_id}',
            },
        ],
    )

* Get lock

.. code-block:: python

   policy.DocumentedRuleDefault(
        name= 'resource_locks:get',
        check_str=ADMIN_OR_SERVICE_OR_PROJECT_READER,
        scope_types=['project'],
        description="Get details about a resource lock.",
        operations=[
            {
                'method': 'GET',
                'path': '/resource-locks/{id}',
            },
        ],
    )


Driver impact
-------------

None. This is an API only feature.

Security impact
---------------

Default RBAC policies will allow users with "admin" role to create, view or
delete user locks. The "admin" role is presumed to be given to the operator
user. If a lock must be created on behalf of the user by a service or an
application, it is advised that the service or application is configured
with a user that has the "service" role and not "admin".

No further security impact, positive or negative is noted.

Notifications impact
--------------------

"lock.create" and "lock.delete" notification events will be emitted for the
respective actions.

Other end user impact
---------------------

User Interface improvements will be introduced in OpenStackClient
(``python-manilaclient`` plugin) and the OpenStack Dashboard (``manila-ui``
plugin). The OpenStackClient addition will be accompanied by ``manilaclient``
and ``openstacksdk`` interfaces:

* Create a resource lock:

.. code-block:: bash

  openstack share lock create <resource_id> \
    [--resource-action <resource_action>] \
    [--resource-type <resource_type>] \
    [--reason <lock_reason>}]

The "resource-action" defaults to "delete".

* Update a resource lock:

.. code-block:: bash

  openstack share lock update <id> \
    [--resource-action <resource_action>] \
    [--reason <lock_reason>}]

* Delete a resource lock:

.. code-block:: bash

  openstack share lock delete <id>

* List resource locks:

.. code-block:: bash

  openstack share lock list

* Show a resource lock:

.. code-block:: bash

  openstack share lock show <id>


Performance Impact
------------------

As we're introducing a new pre-condition on share deletion (or
unmanage/soft-deletion), the share delete API will suffer performance
degradation due to the additional lookup. It's not possible to avoid this
lookup even when locks are not used in the environment. We'll optimize the
query by using appropriate indices. In the future, as more resources and
resource actions use this approach, we will be impacting the existing
performance of these APIs. It's a trade-off for the feature functionality.

Other deployer impact
---------------------

None.

Developer impact
----------------

Consider allowing locks via this interface when defining or manipulating
actions.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  gouthamr

Work Items
----------

- Manila API changes
- support in manilaclient, openstackclient, manila UI
- support in openstacksdk
- e2e tests with manila-tempest-plugin
- API Reference, user and administrator documentation

Dependencies
============

* This feature doesn't depend on work elsewhere, but, the VirtIOFS
  integration effort in Nova requires this feature.


Testing
=======

New tests will be added to create locks, list locks, show locks, delete
locks. Test cases will cover use of multiple locks and involve validation of
request and response schema and codes. RBAC policies will also be tested via
tempest.


Documentation Impact
====================

API Reference will be updated alongside the API changes. User and
administrator documentation will follow alongside the UX changes in
respective repositories.


References
==========

_`[1]` `VirtIOFS Specification <https://specs.openstack
.org/openstack/nova-specs/specs/2023.2/approved/libvirt-virtiofs-attach-manila-shares.html>`_

[2] `2023.2 Bobcat PTG Discussion <https://etherpad.opendev
.org/p/nova-bobcat-ptg#72>`_
