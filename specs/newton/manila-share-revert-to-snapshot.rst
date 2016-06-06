..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================================
Add share revert-to-snapshot feature to manila
==============================================

https://blueprints.launchpad.net/manila/+spec/manila-share-revert-to-snapshot

In manila, the only thing users can do with snapshots is to create new
shares from them. Alternate snapshot semantics were discussed in Tokyo
and Austin. At the Austin Summit, the community agreed one of the new
snapshot semantics would be to revert a share (in place) by restoring
the most recent snapshot of that share taken by manila.

Problem description
===================

There are some drivers which can take snapshots but are unable to create
new shares from those snapshots (or doing so is prohibitively expensive).
And even if creating a share from a snapshot is possible, doing so
consumes quota and in some cases real space on a backend storage system.

Use cases
=========

Consider the case where a file system becomes corrupted or even unusable,
perhaps due to a virus or accidental installation of libraries or other
dependencies, or even a malicious "rm -rf" attack.  In such cases, it
would be valuable to be able to turn back the clock to a point before
the corruption occurred, without having to do so using a new share or
restoring files from a mounted snapshot.


Proposed change
===============

As agreed by the manila community in Austin, manila can address this need
by adding the ability to revert a share (in place) from a snapshot.

Theoretically, if multiple snapshots exist, a storage system could
restore a share to any one of those snapshots.  But reverting a share
to a snapshot which is not the most recent one typically deletes the
later snapshots, which is a form of data loss and may not be what the
user intends.  So this feature will limit manila to restoring the most
recent share snapshot known to manila.

For this use case, reverting backwards in time starting with the most recent
snapshot makes sense. Only the user can determine whether a share is usable or
must be reverted, so getting back to a good state is a manual iterative
process::

  Observe corruption in active file system
  Revert to most recent snapshot
  While corruption is present in active file system:
    Delete most recent snapshot
    Revert to most recent snapshot

To keep the REST interface explicit, and to avoid a race condition where
a snapshot-create and share-revert-to-snapshot operation were run in quick
succession (leading to the user being surprised about which snapshot was
restored), the manila community decided the interface will accept the ID of
the snapshot to be restored. Manila will verify that the snapshot is the most
recent one via the created_at field on the snapshot object, returning an error
if not. At no time will manila delete any snapshots during the revert
operation, and it will not operate on any snapshot other than the one
specified in the REST call.

The share size might have changed after the snapshot to be restored was taken.
The size of any share at the time it is snapshotted is stored in the DB in the
snapshot record.  This should be used to ensure a quota will not be exceeded
by restoring the snapshot, and it may be used to update the share record as
well as the user's share size quota with the original size after the
restoration has succeeded.

The CLI would look like::

  usage: manila revert-to-snapshot <snapshot>

  Revert a share to the specified snapshot.

  Positional arguments:
    <snapshot>  Name or ID of the snapshot to restore.


Note that this feature is proposed in the short term for individual shares,
but it could apply equally well to share groups.  Since groups aren't yet
fully baked, revert-to-group-snapshot should come later.

Alternatives
------------

Manila could simply rely on creating new shares from snapshots.  But not
all snapshot-capable drivers can do this, and it involves either copying
data from the new share to the old one, or updating an application to
point to the new share.

Manila could also add a backup feature to address use cases such as this one
(complete restoration to an earlier point in time to resolve share-wide
corruption, etc.).  But backups typically involve an off-site data copy
(either full or differential) and so are inherently slower than a snapshot
restore operation performed in place by a storage controller.

Data model impact
-----------------

No changes to the database schema are required.

During a snapshot restoration, there are two objects that are affected whose
states must reflect the operation.  The share is being "reverted", while
the snapshot is being "restored".  Other operations must not be allowed
to either object during the restoration, so each will be updated with
a transitional status:

* Share —> 'reverting'
* Snapshot —> 'restoring'

After a successful restoration, both objects' states will again be set
to 'available'.  If the restoration fails, the share will be set to
'reverting_error', and the snapshot will be set to 'available'.

For replicated shares, all "active" replicas must be reverted immediately,
and other replicas will be "out-of-sync" until they are again consistent with
the "active" replicas. For the purpose of determining the most recent
snapshot, the created_at field on the snapshot object will be used; it would
be incorrect to look at the snapshot_instance objects because they may be
created much later during replica creation.

REST API impact
---------------

A single API method is required for revert-to-snapshot.  The REST API is
modeled after other share actions, such as deny_access::

  Method = POST
  URL = /shares/{share_id}/action
  Body = {'revert': {'snapshot_id': <snapshot_id>}}

Calling this method reverts a share to the specified snapshot.  It
is intended for both tenants and admins to use, and the policy.json
file will be updated to reflect allowed use by all.  If the snapshot is
not the most recent one known to manila, or the state of either share or
snapshot is not 'available', the API will return the HTTP error code 409
(Conflict).  If the share isn't found, 404.  If the snapshot doesn't exist,
400 (because it isn't explicitly referenced in the URL).

Driver impact
-------------

There will be one new driver entry point to revert a share to a snapshot, and
another to revert a replicated share to a snapshot.  Drivers may explicitly
advertise support for the revert feature using the 'revert_to_snapshot' pool
attribute, but the share manager will be able to discern that automatically
as it already does for 'snapshot_support' by looking for the presence of the
entry point(s).

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

This feature will be available in python-manilaclient, and it should
be straightforward to implement it in manila-ui as well.  In the latter
case, the GUI should only present the action on the snapshot known to
be the most recent one.

Performance impact
------------------

To correctly identify the latest snapshot known to manila, the existing
snapshot-create workflow must be protected with locks to ensure no races
occur that can cause the snapshot record timestamps to be out of order
relative to the order in which the snapshots were taken on the storage
controller.  This could cause delays in cases where multiple snapshots
of a share are taken in rapid succession, such as in automated tests.

The community decided to place a share in a 'snapshotting' state while
taking a snapshot in order to prevent multiple simultaneous snapshot
operations. The revert-to-snapshot feature depends on this work being
completed first.

Also, determining which snapshot is the latest requires a database
query that sorts by a timestamp (the created_at field on the snapshot object).
This would be slightly slower than a query that does not care about result
ordering.

Other deployer impact
---------------------

None

Developer impact
----------------

As with other snapshot semantics, including the ability to take a
snapshot, a driver must advertise its ability to restore a snapshot.
This will be done using the driver method discovery code that exists
today to report the share revert capability to the scheduler.
The new field shall be 'revert_to_snapshot'. It will also be reported
as a public extra spec on the share type to enable user-facing tools
to selectively offer the feature on a per-snapshot basis, and the
value will be copied from the share type to the share at the time of
share creation.  Shares created before this feature is released will
not have the attribute set, so the revert-to-snapshot action will not be
available on those even if the backend support is present.  The default value
of 'revert_to_snapshot' will be False.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

* clintonk (manila & python-manilaclient)

Other contributors:

* vponomaryov (manila-ui)
* TBD (1st-party drivers)
* TBD (Functional & scenario tests)

Work items
----------

* Implement snapshot locks to remove race conditions that cause snapshot
  timestamps to be unreliable for the purpose of finding the most recent
  one. These can be file locks as provided by Oslo concurrency, but they
  should become distributed locks once the Tooz adoption code is available.
* Implement revert-to-snapshot command in python-manilaclient
* Implement core feature
* Implement revert-to-snapshot in at least one first-party driver
* Implement tempest support
* Implement manila-ui support


Dependencies
============

* For the purpose of determining the most recent snapshot, the created_at
  field on the snapshot object will be used. The snapshot-create API must
  place a share in the 'snapshotting' state to ensure only one snapshot
  operation occurs on a given share at the same time. Otherwise, manila's view
  of the latest snapshot on a share with multiple snapshots could differ from
  the actual latest snapshot on the storage controller.

Testing
=======

Tempest coverage may be added that checks for the existence of the
'revert_to_snapshot' capability and exercises the corresponding API. Tests
should include cases where the share size is changed after the original
snapshot is taken to ensure the share size and quotas are correct after the
revert operation. Negative tests should include attempts to restore snapshots
other than the most recent one.

To guarantee that a snapshot restore actually took place, a new scenario
test will be needed that writes data to a share, creates a snapshot,
modifies the share, restores the snapshot, and ensures the original data
is present in the share.

Documentation impact
====================

As a user-facing feature, this should be covered in the user guide.
The manila devref must be updated to define the new revert_to_snapshot flag.

References
==========

https://etherpad.openstack.org/p/newton-manila-snapshot-semantics
