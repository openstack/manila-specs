..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Manila - Share backup out of place restore
==========================================

Blueprint: https://blueprints.launchpad.net/manila/+spec/out-of-place-restore

The previously introduced share backup spec and API allows for the creation and
restoration of shares through manila, however at the moment, this API is limited
to only performing restores to the source share of the given backup. This
spec proposes to enhance the share backup API to allow restores to non source
share targets.

Problem description
===================
At the moment, it is impossible to restore a given share backup to a location
other then the source share used to create the backup. Allowing users to
restore to a different share specified by share-id or name would extend the
usability of Manila share backups.

Use Cases
=========

As far as business continuity and disaster recovery (BCDR)  is concerned,
there are a few reasons that make such a feature beneficial:

If a user is limited to performing restores to the source share, they must
ensure that the sum of the current shares size and the restores content does
not exceed the size / quota of the given share. This might stop users from
being able to perform restores. Additionally users may be encouraged to
"browse by restore" destructively removing existing share content to see if
a backup has valid content they are interested in.

Likewise, if the storage backend that backs the source share type suddenly
becomes unavailable, users will not be able to utilise their backup.

Allowing restores to a separate share gives an opportunity for the resumption
of a clients service by restoring to a different share backend to that of
the source.

This would bring Manila share backup in-line with what is offered by Cinder,
and would deliver on the Future Work item detailed "Add support to create share
from backup" in the original specification, in an alternative manner.

Proposed change
===============

* Extend python-manilaclient 'share backup restore' command

    The share backup restore command will be extended to allow a optional
    argument, --target-share to be added by a end user, in the form of a valid
    share_id or share_name. This will restore a given backup to the targeted
    share.

* Extend python-manilaclient 'share create' command

   The share create command will be extended to allow a optional argument,
   --backup-id, to be added by a end user in the form of a valid backup id.
   this will create a new share and restore the content of the given backup
   to it.

* update API call for targeted share restore

    The share backup API call for restores will be extended to allow the
    appending of a target share to the body of a restore call as part of
    its information.

    - The component side wsgi action for restores will be modifed to consider
      this new attachment to the body, to check that the supplied share exists
      in the scope of the clients context, and to pass it to the RPC call
      component.

    - Finally, where relevant, backup drivers that wish to make use of targeted
      restores should be updated with additional sanity checks such that a
      difference between the share_protocol of a given backup and target share
      are handled appropriately. In the case of support, the backup should
      proceed as expected, whereas if a given matchup of share_protocols isn't
      supported, a failure should occur explicitly and be communicated.

* backup_driver abstract class update

    To ensure compatibility, between different backup drivers, a new flag
    boolean will be introduced to the backup_driver abstract class:
    self.support_restore_to_target, in this way a backup driver can signal
    to the data manager if it is capable of supporting out of place restores
    to its supported storage backend.

* Manila Data-manager update

    The Manila data-manager will be modified to allow it to receive a
    target_share value as part of a restore call. This doesn't take the form
    of a new argument, rather the data manager will check to see if a share
    can be considered a targeted restore by evaluating the backup source shares
    id agains't the id of the supplied share.

    If these two elements match, a normal restore can be considered to be
    underway, if they do not, a further check should be performed against
    the current backup drivers boolean flag to ensure the operation is
    supported.

    If it is the target shares status will be updated to BACKUP_RESTORING,
    and passed in to the backup driver in place of the original share for
    a restore.

    finally, given these checks or the restore operation succeeds or fails
    the data_manager will appropriately update the state of the share in
    contention.

* NFSBackupDriver update

    The default NFSBackupDriver contained within the data manager will have to
    be extended to support out of place restores, or otherwise have the
    support boolean for targeted restores set False so as to disallow such
    requests.

Alternatives
------------

We could accept that out of place restores are not a valid use case for Manila
and cease pursuit of the feature.

Data model impact
-----------------

None

CLI API impact
--------------

expand the restore and create commands in the python manila client OSC plugin:

openstack share backup restore [<backup> --target-share <target-share>]

* backup: ID of backup to restore.
* target_share: *optional* Share to target for restore, where value can be A
                share name or ID. Default to None, e.g. restore to source)

openstack share create [<share arguments> --backup-id <backup-id>]

* share arguments: mandatory parameters expected when creating a share, e.g.
                   share_protocol, size, etc.
*  backup-id: *optional* Optional backup ID to create the share from.


REST API impact
---------------

Normal API Restores will still be available as before using::

    POST /v2/share-backups/{backup_id}/action

    {
        "restore": null
    }

while if a targeted restore is desired, the ID can be appended as info::

    POST /v2/share-backups/{backup_id}/action

    {
        "restore": <target_share_id>
    }


In the case the target_share or backup is not known to manila, the API will
respond with ``404 Not Found`` as with a normal share.

In the case the target_share or backup is known but the tenant user has no
permissions, the API will respond with ``403 Unauthorised``

The API will respond ``202`` if request is accepted.

Driver impact
-------------

The backup driver will have to be modified to include the new support
boolean, eg::

    class BackupDriver(object):

        def __init__(self):
            super(BackupDriver, self).__init__()

            # This flag indicates if backup driver implement backup, restore,
            # delete, and get progress operations by its own or uses the data
            # manager.
            self.use_data_manager = True

            # This flag indicates if the backup driver supports out of place
            # restores to a share other then the source of a given backup.
            self.support_restore_to_target = False

Security impact
---------------

As with the original share backup specification the data node would have
access to read the source and target share data. If the deny access phase
fails, the node will continue forever with access to the user's data.

Notifications impact
--------------------

None

Other end user impact
---------------------

If the End user targets an existing share other then the source, it will
become unavailable in regards to other share operations during the restore.
e.g. no extend/shrink share, replication share, share-group operation,
migration share.

Performance Impact
------------------

None

Other deployer impact
---------------------

The deployer will be able to restore a share to a non source target.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
    * zgoggin(zachary.goggin@cern.ch)

Work Items
----------

* Update restore command in python-manilaclient.
* Update restore API WSGI handling
* Update data manager and backup_driver to support targeted restores
* Update tempest support.
* Update manila-ui support.
* Update manila backup in devstack plugin to support.

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
