..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================
Create share from snapshot extra spec
=====================================

https://blueprints.launchpad.net/manila/+spec/add-create-share-from-snapshot-extra-spec

This spec describes why and how we must separate the overloaded meaning of
'snapshot_support' into two independent standard extra specs. Furthermore, to
prevent the proliferation of required extra specs as well as to make the
creation of new share types as simple and flexible as possible, we can make
snapshot_support and each of the new snapshot-related extra specs optional.

Problem description
===================

For a long time, manila has had the 'snapshot_support' standard extra spec,
which has been overloaded to mean *two* things:

* a driver can take a snapshot of a share, and
* a driver can create new shares from a snapshot

With additional snapshot semantics being proposed, including share revert and
mountable snapshots, we expect that some drivers may support the new
semantics without being able to create new shares from snapshots. It is
therefore necessary to break the overload of 'snapshot_support'.

We can add a new standard extra spec, create_share_from_snapshot_support, to
indicate whether a backend can create new shares from share snapshots.

The snapshot_support extra spec is currently required, which means an admin
cannot create a type where snapshot_support is a "don't care", such that
shares of that type will not offer snapshot features even if they exist on
storage controllers than can take snapshots. By making snapshot_support and
the other new snapshot-related extra specs optional, we can support "don't
care" semantics on share types.

Use cases
=========

A driver that can take snapshots of shares, but not create new shares from
snapshots, will benefit from this change. For example, such a driver could
report snapshot_support=True, create_share_from_snapshot_support=False, and
revert_to_snapshot_support=True.

Proposed change
===============

To accomplish this change, we will do the following:

* Define the new extra spec, create_share_from_snapshot_support, at the next
  available microversion.
* Add a database migration to add create_share_from_snapshot_support to each
  share type that contains snapshot_support. Because snapshot_support was
  overloaded prior to this change, we can simply copy the value of
  snapshot_support for each type to the new create_share_from_snapshot_support
  extra spec.
* Add a database migration to add the new create_share_from_snapshot_support
  capability to the share record, so that manila has a record of each share's
  relevant snapshot capability at the time the share was created.
* Copy the value of create_share_from_snapshot_support to each share upon
  share creation. If create_share_from_snapshot_support is not present on the
  share type, set the corresponding value on the share to False.
* Continue copying the value of snapshot_support to each share upon share
  creation. Because snapshot_support is now optional, if snapshot_support is
  not present on the share type, set the corresponding value on the share to
  False.
* Ensure create_share_from_snapshot_support is True when creating shares from
  snapshots.
* Ensure that previous microversions cannot delete the snapshot_support extra
  spec from share types.
* Ensure that previous microversions continue setting a default value of True
  for snapshot_support as well as create_share_from_snapshot_support.

Alternatives
------------

We could leave the overload in place, but this would lead to confusing and
incorrect behavior for drivers that can create snapshots but not create new
shares from them.

We could continue requiring snapshot_support and all new snapshot-related
extra specs to be present on each share type, but that would prevent admins
from creating share types with "don't care" semantics.

Data model impact
-----------------

* Add create_share_from_snapshot_support field to share table, and copy value
  of snapshot_support to the new create_share_from_snapshot_support field for
  each share.
* Add create_share_from_snapshot_support rows to the share_type_extra_specs
  table for each type that has snapshot_support, with the value copied from
  snapshot_support.

REST API impact
---------------

At the microversion for this change:

* The snapshot_support extra spec becomes deletable, as are the new
  snapshot-related extra specs.
* Default values for snapshot_support are no longer provided. If not specified
  on a share type, snapshot_support and create_share_from_snapshot_support are
  not returned from the share types API for admin contexts.  However, users see
  both values with a nominal value of False so that they know how shares of
  that type will behave.
* The value of create_share_from_snapshot_support is checked on a share prior
  to creating a share from a snapshot (else HTTPBadRequest is returned).

Driver impact
-------------

The value of create_share_from_snapshot_support is inferred for each driver
based on the methods it contains, much as is already done for snapshot_support.

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

Prior to this change, python-manilaclient provided default values for the
required extra spec snapshot_support. As of the new microversion, it will not
provide a default for snapshot_support because the extra spec is no longer
required.

Performance Impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  * clintonk (manila & python-manilaclient)

Other contributors:
  * gouthamr (tempest & db-related tests)

Work Items
----------

Code is all in one patch and has already been made available.


Dependencies
============

None


Testing
=======

New unit and tempest test coverage will be added for all of the added or
changed functionality.


Documentation Impact
====================

* API Ref: Add content about the API.
* User Guide: Add content about the share types.
* Admin Guide: Add content regarding the common capabilities and the don't care
  behavior.
* Developer Ref: Add create_share_from_snapshot_support to common capabilities
  and the infamous driver matrix.


References
==========

None
