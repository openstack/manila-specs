..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
Add mountable snapshots
=======================

https://blueprints.launchpad.net/manila/+spec/manila-mountable-snapshots

Currently, manila only allows the user to create new shares from snapshots.
This limitation raised some discussions on Tokyo and Austin summits to add new
snapshot semantics. One of the new features agreed to be implemented is the
capability to mount a snapshot in read-only mode.

Problem description
===================

Some drivers can expose snapshots to be mountable, but the current
implementation demands that the only action that the user can do with a
snapshot is to create a new share from it. While this is useful for several
situations, there are use cases in which creating a whole new share from a
snapshot is overkill, or in some cases, cannot be done by existing drivers.

Use Cases
=========

There are situations in which the user only needs a single file or a set of
files to be restored. In these cases it would be simpler to just mount the
snapshot and retrieve the desired files, instead of needing to create a whole
new share to copy the files and then delete it. It could also be used by an
administrator or an external system, like manila-data.

This feature could also be used to check the contents of the snapshot before
reverting the share to its state, using the revert-to-snapshot feature.

Proposed change
===============

The proposal is to add export locations to the snapshots, so that the user can
allow and deny access to the snapshots to mount them in read-only mode. Some
users or systems that do not have access to the parent share might need access
to the snapshot, so the access rules for the snapshot must be separated from
its parent share. This separation also provides more security, since it avoids
unwanted access to the share. It is important to note that in order to support
this feature, a driver must be able to support this separation of access rules.

The snapshot export locations will be provided by the driver when creating a
snapshot. The APIs for this proposal will follow the current implementation of
share allow and deny access, including the new protocol-specific access rule
restrictions agreed in Barcelona summit. In order to support this feature,
a new extra spec called ``mount_snapshot_support`` will be added.

Alternatives
------------

The main alternative is to create a new share from the snapshot, so the files
could be retrieved from this new share. While this is already implemented and
usable, this alternative consumes the user quotas and in most cases the new
share will just be used to retrieve a single file, which makes this overly
complicated. Another alternative is to implement a read-only share that could
be used in these situations. Being a read-only share, some of the operations
that the user can do with a share in manila might not make sense (e.g. create
snapshot, extend, shrink), so this new type of share might not comply with the
current concept of shares in manila. There is also the possibility that the
snapshot inherits the access rules from the share, but in read-only. While this
alternative is simpler, it is also more restrictive because it does not allow
systems or users that do not originally have access to the share to access the
snapshot.

Data model impact
-----------------

Three new tables will be needed:

1) ShareSnapshotInstanceExportLocations
2) ShareSnapshotAccessMapping
3) ShareSnapshotInstanceAccessMapping

A new ``export_locations`` property will be added to the existing
ShareSnapshotInstance table and a ``mount_snapshot_support``
property will be added to the existing Share table.

REST API impact
---------------

The following new API methods will be implemented:

1) (202, POST) snapshot-access-allow: allows access to the snapshot

URL: /snapshots/<id>/action
Body: {"access-allow": {"access_type": <value>, "access_to": <value>}}

2) (202, POST) snapshot-access-deny: denies access to the snapshot

URL: /snapshots/<id>/action
Body: {"access-deny": {"access_id": <value>}}

3) (200, GET) snapshot-access-list: list the access of the snapshot

URL: /snapshots/<id>/access-list

4) (200, GET) snapshot-export-location-list

URL: /snapshots/<id>/export-locations

5) (200, GET) snapshot-instance-export-location-list

URL: /snapshot-instances/<id>/export-locations

6) (200), GET) snapshot-export-location-show

URL: /snapshots/<snap-id>/export-locations/<el-id>

7) (200, GET) snapshot-instance-export-location-show

URL: /snapshot-instances/<si-id>/export-locations/<el-id>

Driver impact
-------------

Add driver interfaces::

    def snapshot_allow_access(self, context, snapshot, access,
                              share_server=None):
    """Allow access to the snapshot."""

    def snapshot_deny_access(self, context, snapshot, access,
                             share_server=None):
    """Deny access to the snapshot."""


To support this feature, the create_snapshot method must be changed to return
the snapshot export locations, and the drivers must report
``mount_snapshot_support`` as True.

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

This feature will be implemented in python-manilaclient and manila-ui. The
commands for allowing and denying access to snapshots will follow the current
implementation for allow and deny access to shares and getting the snapshot
export locations will be similar to the shares implementation.

Performance Impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

This will require a change on the driver interface. To support this feature,
drivers will need to implement the new methods.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  tiago.pasqualini

Work Items
----------

* Implement the core feature with functional tempest and scenario test
  coverage
* Implement snapshot-access-allow and snapshot-access-deny commands in
  python-manilaclient as well as python-manilaclient functional tests
* Implement mountable snapshots in one of the first-party drivers
* Implement manila-ui changes (snapshot allow/deny access and expose
  snapshot export locations)
* Create mountable snapshots documentation

Dependencies
============

This feature depends on
`Create share from snapshot extra spec <create-share-from-snapshot-extra-spec>`_,
since it will remove overload that currently exists on the 'snapshot_support'
extra spec.

Testing
=======

* Unit tests
* Functional tempest tests
* Scenario tests

Documentation Impact
====================

- Docstrings
- Devref
- API reference
- User guide

References
==========

Etherpad: https://etherpad.openstack.org/p/mitaka-manila-mountable-snapshots
