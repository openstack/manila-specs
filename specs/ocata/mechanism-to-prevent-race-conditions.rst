..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
Mechanism to Prevent Race Conditions
====================================

https://blueprints.launchpad.net/manila/+spec/eliminate-race-conditions

This proposal is to develop a general solution for preventing race
conditions which will work across services and in deployments where
there are multiple copies of the same services (commonly known as
Active/Active HA deployments).

The focus is on keeping all state in the database, and protecting changes to
database state using briefly-held locks. Also concurrent operations which
are mutually exclusive should fail as early as possible with a helpful error
code to simplify the retry logic of upper layers.

Problem description
===================

Certain operations in Manila should not be allowed to proceed in parallel,
because the result of one operation would prevent the other operation from
completing successfully.

For example, taking a snapshot of a share cannot happen simultaneously while
deleting that share. Either the snapshot must occur first, which prevents the
delete -- or the delete must occur first, which prevents the snapshot.
Unfortunately, not enough state is stored in the database to prevent these
operations from racing with each other, so in practice two API calls can both
proceed through the API service to the share manager, where eventually an
error will occur and one or both operations will fail mysteriously.

There are multiple scenarios like the above where undefined behavior results.
This specification does not attempt to enumerate all of them because the goal
is to describe a mechanism for fixing these kinds of issues rather than
explicitly fixing all such issues. Generally speaking, race conditions should
be treated as bugs, but up until now Manila has lacked the tools to fix these
bugs reliably.

Use cases
=========

Specific cases:

* Two snapshot operations should not be able to occur at the same time. One
  must complete before the second can begin. This ensures that snapshots occur
  in a known order, and prevents the useless situation of having 2 identical
  snapshots.

* Taking a snapshot of a share should prevent a delete of that share.

* Valid changes to access rules should always be accepted regardless of the
  state of the existing access rules. Although rules are applied to the backend
  asynchronously, it's valid to add multiple rules faster than the system can
  apply them and expect Manila to catch up.

General cases:

* These guarantees must be enforceable using a database running in a clustered
  configuration such as Galera. This prevents obvious solutions such as relying
  on DB row-level locking.

* These guarantees must be enforceable while running multiple copies of Manila
  services, including multiple API, scheduler, and share manager services.
  This prevents obvious solutions like in-process locks.

* These guarantees must be enforceable while running in a distributed
  configuration where cooperating services are on different nodes (physical,
  VMs, or containers). This prevents our existing approach of using file
  locks. Even though network-based file locking solutions exist, they represent
  a single point of failure and are unacceptable in properly distributed
  environments.

* The Manila services should be able to automatically and gracefully recover
  from crashes and other unplanned downtime. This means that implicit state
  should be avoided and because long-held locks are implicit state they should
  be avoided in favor of short-held locks with explicit state.

Proposed change
===============

More transitional states will be added so that operations which can conflict
with other operations can be explicitly detected by looking at the share
state. This spec only proposes one specific new state to address the races
involving snapshots, but more generally provides a framework for resolving
similar races as they are discovered.

Transitions between states will always be done while holding a distributed
lock -- a lock implemented by a distributed lock manager (DLM). Use of
distributed locks ensures that all services see the same locking state even if
services run on different nodes, and even across transient failures such as
node failures and network partitions. The lock will be held only for the
duration of the database test-and-set operation to minimize lock contention.

No locks will be held during calls from the share manager to the share driver.
Mutual exclusion between driver calls will be achieved with state checks.

No locks will be held during RPC calls or casts.

Alternatives
------------

The approach used by Cinder which relies on elaborate SQL calls to
compare-and-swap fields was considered but rejected for the following reasons:

* The code in Cinder can't be shared with Manila because it relies on OVO (Oslo
  Versioned Objects)

* Not enough people understand how it works so it's likely to be hard to
  maintain.

* Cinder's compare-and-swap approach limits the kind of state changes you can
  make because updating multiple tables atomically is impossible. Locks
  don't suffer from this restriction.

Data model impact
-----------------

New states will be added:

* Snapshotting

* States for access rules covered in `Access rules spec`_


.. graphviz::

    digraph share_states {
        label="Share States"
        // Transitional States
        creating[shape=hexagon];
        manage_starting[shape=hexagon];
        deleting[shape=hexagon];
        snapshotting[shape=hexagon,color=gold4, fontcolor=gold4];
        migrating[shape=hexagon];
        shrinking[shape=hexagon];
        extending[shape=hexagon];
        unmanage_starting[shape=hexagon];
        replication_change[shape=hexagon];
        // Error states
        error[color=red4, fontcolor=red4];
        shrinking_error[color=red4, fontcolor=red4];
        shrinking_possible_data_loss_error[color=red4, fontcolor=red4];
        extending_error[color=red4, fontcolor=red4];
        unmanage_error[color=red4, fontcolor=red4];
        manage_error[color=red4, fontcolor=red4];
        error_deleting[color=red4, fontcolor=red4];
        // Other states
        new[color=blue, fontcolor=blue];
        available[color=darkgreen, fontcolor=darkgreen];
        deleted[shape=box, color=navy, fontcolor=navy];
        unmanaged[shape=box, color=navy, fontcolor=navy];
        // User requested transitions
        new -> creating[label="create"];
        new -> manage_starting[label="manage"];
        available -> deleting[label="delete"];
        available -> snapshotting[label="create snapshot", color=gold4, fontcolor=gold4];
        available -> migrating[label="migrate"];
        available -> shrinking[label="shrink"];
        available -> extending[label="extend"];
        available -> unmanage_starting[label="unmanage"];
        available -> replication_change[label="add replica"];
        // Automatic transitions
        creating -> available[label="success", color=darkgreen, fontcolor=darkgreen];
        deleting -> deleted[label="success", color=darkgreen, fontcolor=darkgreen];
        snapshotting -> available[label="success", color=darkgreen, fontcolor=darkgreen];
        manage_starting -> available[label="success", color=darkgreen, fontcolor=darkgreen];
        unmanage_starting -> unmanaged[label="success", color=darkgreen, fontcolor=darkgreen];
        extending -> available[label="success", color=darkgreen, fontcolor=darkgreen];
        shrinking -> available[label="success", color=darkgreen, fontcolor=darkgreen];
        replication_change -> available[label="success", color=darkgreen, fontcolor=darkgreen];
        // Reset transitions
        error -> available[label="reset"];
        shrinking_error -> available[label="reset"];
        extending_error -> available[label="reset"];
        unmanage_error -> available[label="reset"];
        manage_error -> available[label="reset"];
        error_deleting -> available[label="reset"];
        // Error transitions
        creating -> error[label="fail", color=red4, fontcolor=red4];
        migrating -> error[label="fail", color=red4, fontcolor=red4];
        shrinking -> shrinking_error[label="fail", color=red4, fontcolor=red4];
        shrinking -> shrinking_possible_data_loss_error[label="fail", color=red4, fontcolor=red4];
        extending -> extending_error[label="fail", color=red4, fontcolor=red4];
        unmanage_starting -> unmanage_error[label="fail", color=red4, fontcolor=red4];
        manage_starting -> manage_error[label="fail", color=red4, fontcolor=red4];
        snapshotting -> error[label="fail", color=red4, fontcolor=red4];
        deleting -> error_deleting[label="fail", color=red4, fontcolor=red4];
    }

REST API impact
---------------

New states will be visible through any API that shows states. Also new
error conditions will become possible as we detect races earlier and
report them directly.

The behavioral changes related to locking will not be microversioned, as it
won't be possible or desirable to emulate the old behavior once the changes
are implemented. However in cases where new states are added, those changes
will be microversioned so that clients which depend on the new states can
detect that the server supports them.

Driver impact
-------------

None

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

Distributed locking is expected to moderately slow down state changes. Also
adding more state changes will slow down operations that require them.

Other deployer impact
---------------------

Requirement to deploy and configure suitable `Tooz`_ backend. Since Manila will
depend on tooz for correctness, tooz backends that fail to meet the API
contract won't be suitable.

Developer impact
----------------

This will be significant. Developers will need to follow the new model for
all features that involve state changes. Also care will be needed with locks
to avoid deadlock situations. Holding locks for very limited time will help
avoid deadlocks but in case 2 locks are ever held at the same time, they need
to be deadlock-proofed by establishing a lock order.

Implementation
==============

Assignee(s)
-----------
  bswartz

Work Items
----------

* Add snapshotting state

* Complete tooz integration

* Wrap state changes with tooz locks

Dependencies
============

Tooz

Testing
=======

Existing tests will help ensure no regressions, but to detect race
conditions we need rally tests or similarly high-concurrency tests.

Documentation Impact
====================

Admin guide - need to document tooz requirements.

Developer reference - need to document state machines and locking protocol

References
==========

_`Access rules spec`: https://review.openstack.org/#/c/399049/

_`Tooz` integration: https://review.openstack.org/#/c/318336/
