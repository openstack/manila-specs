..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================================
Manila share and share snapshot deferred deletion
=================================================

Blueprint: https://blueprints.launchpad.net/manila/+spec/deferred-deletion

Manila does not support the deferred deletion of share and share snapshots.
This blueprint is to add support for share and share snapshot deferred
deletion. The end user will delete the share and share snapshot using API
and then Manila will perform the deferred deletion of the share and
share snapshot respectively thereby freeing quota immediately.

Problem description
===================

If users want to remove a share or share snapshot from Manila, it is possible
to delete them using API. The manila-API part of deletion procedure finishes
quickly, on the other hand the backend driver takes time. The quotas allocated
to the share or share snapshot are not freed until share or share snapshot are
deleted completely. This increases waiting time for creation of new share or
share snapshot. Providing a way to release the quota immediately and perform
deferred deletion will solve this issue. Also automated workflows face problem
in creation of new shares and/or share snapshots where they have to wait until
quota is available after share or share snapshot is deleted by backend driver.

Use Cases
=========

* In automated workflows where deletion of share/share-snapshot is an
  intermediate step, have to wait until share/share-snapshot is deleted. Defer
  deletion allows automation to proceed without waiting for driver.

* The user want to delete share/share-snapshot and check the available
  share/share-snapshot and their corresponding gigabytes quota.


Proposed change
===============

The following are the changes to be made:

* A new status values 'DELETING_IN_DRIVER' and 'DELETING_IN_DRIVER_ERROR' will
  be introduced to identify share instances which will be processed in
  deferred deletion.

* Add new configuration option 'deferred_deletion', which means if TRUE the
  shares are deleted in deferred deletion manner, and if FALSE the share will
  be deleted the way it is happening today.

* As soon as share or share-snapshot is deleted by API request, the quota will
  be reclaimed and share/share-snapshot instance will be moved to
  'DELETING_IN_DRIVER' state. Any delete request on share/share-snapshot while
  it is in 'DELETING_IN_DRIVER' state, will return error.
  A periodic task running every 300 seconds will be added in share manager,
  which gets all share instances in 'DELETING_IN_DRIVER' state. The share
  manager, will then call backend driver to delete the share/share-snapshot
  instance and move it to 'DELETED' state on successful deletion. Any failure
  during deletion, will move share/share-snapshot instance in the
  'DELETING_IN_DRIVER_ERROR' state. Only admin can list those
  shares/share-snapshots and then delete/cleaup them on storage, where they
  will be marked as 'DELETED'. The periodic task also checks for
  share/share-snapshot in the 'DELETING_IN_DRIVER_ERROR' state and tries to
  delete them again after 30 minutes of previous attempt.

* No change in backend driver.

Alternatives
------------

Instead of adding new status value, we can mark share/share-snapshot as
'DELETED' immediately and move those shares/share-snapshots to another project.
This project will contain only deleted shares/share-snapshots by manila API
service. The share manager will run a periodic task to delete share and
share-snapshot from this project. In this case as well since quota is
reclaimed earlier, share manager will make sure share instance must be deleted
by backend driver.

An issue with this approach is we will change project id of original share
during deletion which can be unwanted data modification.


Data model impact
-----------------

* No change in share instance table except introduction of two more values
  for 'status' field.


CLI API impact
--------------

None

REST API impact
---------------

Shares and share snapshots in 'deleting_in_driver_error' state are not visible
to regular user. Only admins can list/show those shares and share snapshots
because they can react to such error like cleaning up them.


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

If configured, instead of 'deleting' and 'deleting_error' states, end user
will now see 'deleting_in_driver' and 'deleting_in_driver_error' states
respectively. Also, end user will observe that share in 'deleting_in_driver'
state has freed its quota immediately.

The cloud admins can list shares in the new state(s) via API.

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
    * kpdev(kinpaa@gmail.com)

Work Items
----------

* Implement periodic share manager function to process deferred deletion.
* Implement tempest support.

Future Work Items
-----------------

None

Dependencies
============

None

Testing
=======

* Unit tests
* Tempest tests

Documentation Impact
====================

- Docstrings
- Devref
- User guide
- Admin guide
- Release notes

References
==========

None
