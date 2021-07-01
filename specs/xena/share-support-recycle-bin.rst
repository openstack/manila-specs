..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================
Support Share Recycle Bin
=========================

https://blueprints.launchpad.net/manila/+spec/manila-share-support-Recycle-Bin

Manila doesn't support share Recycle Bin. This BP is to add support for share
Recycle Bin. The end user can soft delete share to Recycle Bin, and can restore
it.

Problem Description
===================

If users want to remove a share from Manila, it is possible to delete a given
share. Or if they want the share only not being managed by Manila anymore,
they can unmanage it and the share would still exist in the share backend. If
users later wants to restore the share, they must record
``export_location_path``, ``host``, ``share_proto`` and so on, and end users
do not have permission to manage a share. If the user never uses the share
again, this share data will remain permanently on the back-end storage,
becoming garbage data. So users need a way to temporarily delete and easily
restore.

Use Cases
=========

* The user wants to soft delete shares to Recycle Bin.
* The user wants to list shares in Recycle Bin.
* The user wants to restore shares from Recycle Bin.
* The user wants to completely delete share from Recycle Bin.

Proposed Change
===============

The following are the changes to be made:

* A new column ``is_soft_deleted`` will be added in ``shares`` table to
  identify shares placed in the Recycle Bin.

* A new column ``scheduled_to_be_deleted_at`` will be added in ``shares``
  table, the shares in Recycle Bin will be automatically and completely
  deleted once the expire time reached. Shares not in Recycle Bin
  ``scheduled_to_be_deleted_at`` will be None.

* A new property ``is_soft_deleted`` will be added in ``ShareInstance`` model.
  This property will inherit the parent share's value.

* Add new configuration item ``soft_deleted_share_hold_time``, which means
  the maximum time cloud administrators want to keep a share in the recycle
  bin. The default value is 604800 seconds (7 days).

* A new share action will be added for soft delete share. Once the share has
  been deleted to the Recycle Bin, will set ``is_soft_deleted`` of share to be
  True. and calculate the ``scheduled_to_be_deleted_at`` of the share.
  ``scheduled_to_be_deleted_at`` = now_time + ``soft_deleted_share_hold_time``.

* A new request parameter ``is_soft_deleted`` will be added to the original
  list shares API. ``is_soft_deleted`` default is False, if set True, it will
  only list shares that were soft deleted.

* the project quota will remain allocated after the share is soft deleted,
  and it will not be released until the shares are deleted from the Recycle
  Bin.

* A new periodic task will be added to the share manager layer to check if the
  shares in the Recycle Bin have reached their ``scheduled_to_be_deleted_at``.
  If the ``scheduled_to_be_deleted_at`` gets reached, the periodic task will
  delete the share automatically.

* Add new API to restore share from Recycle Bin.

* List share instances will filter the share's ``is_soft_deleted`` is False.

* The above API changes will bump a new microversion.

Alternatives
------------

None

Data model impact
-----------------

* Two new columns ``is_soft_deleted`` and ``scheduled_to_be_deleted_at`` will
  be added to model of ``shares``.

* A new property ``is_soft_deleted`` will be added in ``ShareInstance`` model.
  This property will inherit the parent share's value.

REST API impact
---------------

* Soft delete a share to Recycle Bin

POST /v2/shares/{share_id}/action

{"soft_delete": null}

Preconditions
    * (1)Share status must be available, error or inactive
    * (2)You cannot soft delete share already in Recycle Bin.
    * (3)You cannot already have a snapshot of the share.
    * (4)You cannot already have a group snapshot of the share.
    * (5)You cannot already have a replica of the share.
    * (6)You cannot soft delete a share that doesn't belong to your project.

If the provided `share_id` doesn't exist, the API will respond with
``404 Not Found``.
If the operation can't be performed due to not meet the (1)(3)(4) constraints,
the API will respond with ``400 Bad Request``.
If the operation can't be performed due to not meet the (2)(5) constraints, the
API will respond with ``409 Conflict Request``.
If the operation can't be performed due to not meet the (6) constraints, the
API will respond with ``403 forbidden``.

* List shares in Recycle Bin

GET /v2/shares?is_soft_deleted=true

* List shares in Recycle Bin with details

GET /v2/shares/detail?is_soft_deleted=true

* Delete share completely from Recycle Bin
  same as delete share not in Recycle Bin

DELETE /v2/shares/{share_id}

If the provided `share_id` doesn't exist, the API will respond with
``404 Not Found``.

* Restore share from Recycle Bin

POST /v2/shares/{share_id}/restore

{'restore': null}

If the provided `share_id` doesn't exist, the API will respond with
``404 Not Found``.
If the share not exist in Recycle Bin, the API will return success directly.

Security impact
---------------

None

Notifications impact
--------------------

None.

Other end user impact
---------------------

The Manila client, CLI will be extended to support share Recycle Bin.

* The command of soft delete share will be like::

    manila soft-delete <share_id>

* The command of list shares in Recycle Bin, the supported parameters are the
  same as the Manila list, it will be like::

    manila list --soft-deleted

* The command of restore share from Recycle Bin will be like::

    manila restore <share_id>

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
  haixin<haixin@inspur.com>


Work Items
----------

* Update API.
* Update Manager.
* Update Manila CLI commands.
* Update unit and tempest test.
* Update related documents.
* Update Manila UI.

Dependencies
============

None


Testing
=======

* Add the unit tests
* Add the tempest tests

Documentation Impact
====================

The following OpenStack documentations will be updated to reflect this change:

* Openstack Admin Guide
* OpenStack User Guide
* OpenStack API Reference

References
==========

None
