..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
Manila backup & restore share
=============================

Blueprint: https://blueprints.launchpad.net/manila/+spec/share-backup

Backup and restore share is a valuable feature for most of storage
users, especially for NAS user, but currently, manila itself doesn't
support backup and restore share features, this spec proposes a backup
and restore solution based on the existing one from cinder.

Problem description
===================

Today we can't use manila commands to backup shares. These shares reside on
the storage backend itself. Providing a way to backup shares directly will
allow the user to protect the shares on a backup device, separately from the
same storage backend.

Use Cases
=========

There are users who have many shares and would like to protect these shares.
This proposal of share backup provides a manila level of data protection.

There are other projects in OpenStack focusing on data protection such as
Freezer, Karbor, Raksha, etc. They are all in different stages of design,
development, or adoption. The backup API in manila is not a replacement of
those projects. Instead, manila APIs can be consumed by those higher level
projects for data protection and can also be used directly by users who do
not need those higher level projects.

When user backing up files from a share in a naive way, we lose metadata.
By having backup API in Manila, we can retain metadata (when backend is
identical)

Also when talking about share backup, most of the users would like to
enforce backup policies on them. Such as incremental backup every two days,
create backup local or remote, only keep latest 5 copies, etc. The policies
are not part of this spec and will be considered for future.

Proposed change
===============

* New API collection for backups

  In order to support backup, We will introduce the basic operations create
  /update/delete/list/show/restore for backups. We also allow to reset the
  backup state in some scenarios.

* New database resource backup

  When creating a backup, user can specify share_id and availability_zone.
  Information related to backup i.e. backup_id, share_id, status, AZ and
  host are stored in database.

* New backup driver

  A new type of driver called backup driver will be introduced to take
  responsibility for share related operations. The backup driver will
  configured at the manila data service node, and provide the basic abilities

  1. Create a backup.
  2. Delete a backup.
  3. Restore a backup in specified share.

  Correspondingly, we will implement these with a new and simple backup driver
  called 'NFSBackupDriver'. This driver is copied from cinder `[1]`_ and aims
  to provide the basic backup ability. The vendor that supports nfs, must
  provide space for nfs to interconnect with nfs backup drivers. The
  implementation of nfs backup driver will be generic though. The backup
  process for this driver would be:

  1. Make sure share is in available state and not busy.
  2. Allow read access to share and write access to backup share.
  3. Mount the share and backend driver's share(i.e. backup share) to the
     data service node.
  4. Copy data from share to backup share.
  5. Unmount the share and backup share.
  6. Deny access to share and backup share.

  Also, several new configuration options will be added to support backup
  driver, at present, only one backup backend is allowed for
  simplicity (we don't have to report the backup driver's capabilities
  and filtering the backends). By default no backup driver will
  be enabled::

      # OPTION1: enabled backup backend
      enabled_backup_backend = backup1
      [backup1]
      # OPTION2: which backup driver will be used for this backend
      backup_driver = manila.data.drivers.test.TestBackupDriver

* New status for backup and share:

  1. backup(
       creating, available,
       deleting, deleted, error_deleting
       backup_restoring, error)
  2. share(
       backing_creating, backup_restoring,
       backup_restoring_error)

  During backup, share will be marked as busy and other operations on share
  such as delete, soft_delete, migration, extend, shrink, ummanage,
  revert_to_snapshot, crate_snapshot, create_replica etc can not be performed
  unless share become available. Finally, whether or not the share is
  successfully backed up, the state of the share is rolled back to the
  available state. In case backup fails, share task_state will contain the
  failure information. Also, share message will be recorded.

* New clean up actions

  The backup and restore actions could break when service is down, so new
  clean up action will be added to reset the status and clean temporary
  files (if involved).

* New quotas for backup

  1. quota_backups, indicate the share backups allowed per project.
  2. quota_backup_gigabytes, indicate the total amount of storage, in
     gigabytes, allowed for backups per project.

  Correspondingly, we will add new configuration options.


Alternatives
------------

We could use the third-party projects to backup the file shares.

Data model impact
-----------------

* Add table for backup

  +-----------------------+--------------+------+-----+---------+-------+
  | Field                 | Type         | Null | Key | Default | Extra |
  +=======================+==============+======+=====+=========+=======+
  | created_at            | datetime     | YES  |     | NULL    |       |
  +-----------------------+--------------+------+-----+---------+-------+
  | updated_at            | datetime     | YES  |     | NULL    |       |
  +-----------------------+--------------+------+-----+---------+-------+
  | deleted_at            | datetime     | YES  |     | NULL    |       |
  +-----------------------+--------------+------+-----+---------+-------+
  | deleted               | tinyint(1)   | YES  |     | NULL    |       |
  +-----------------------+--------------+------+-----+---------+-------+
  | id                    | varchar(36)  | NO   | PRI | NULL    |       |
  +-----------------------+--------------+------+-----+---------+-------+
  | share_id              | varchar(36)  | NO   |     | NULL    |       |
  +-----------------------+--------------+------+-----+---------+-------+
  | user_id               | varchar(255) | YES  |     | NULL    |       |
  +-----------------------+--------------+------+-----+---------+-------+
  | project_id            | varchar(255) | YES  |     | NULL    |       |
  +-----------------------+--------------+------+-----+---------+-------+
  | host                  | varchar(255) | YES  |     | NULL    |       |
  +-----------------------+--------------+------+-----+---------+-------+
  | availability_zone     | varchar(255) | YES  |     | NULL    |       |
  +-----------------------+--------------+------+-----+---------+-------+
  | display_name          | varchar(255) | YES  |     | NULL    |       |
  +-----------------------+--------------+------+-----+---------+-------+
  | display_description   | varchar(255) | YES  |     | NULL    |       |
  +-----------------------+--------------+------+-----+---------+-------+
  | status                | varchar(255) | YES  |     | NULL    |       |
  +-----------------------+--------------+------+-----+---------+-------+
  | size                  | int(11)      | YES  |     | NULL    |       |
  +-----------------------+--------------+------+-----+---------+-------+


* New field in shares table

  +-----------------------+--------------+------+-----+---------+-------+
  | Field                 | Type         | Null | Key | Default | Extra |
  +=======================+==============+======+=====+=========+=======+
  | source_backup_id      | varchar(255) | YES  |     | NULL    |       |
  +-----------------------+--------------+------+-----+---------+-------+


CLI API impact
--------------

Add new commands to openstackclient(OSC):

openstack share backup create [--name <name>]
                              [--description <description>]
                              [--availabity-zone <availability-zone>]
                              <share>

* name: Name of backup. Default=None
* description: Backup description. Default=None.
* availability-zone: Availability-zone of backup.
* share: Name or ID of share to backup.


openstack share backup restore <backup>

* backup: ID of backup to restore.


openstack share backup list [--share <share>]

* share: Filter backups by a share name or ID.


openstack share backup show <backup>

* backup: ID or name of backup


openstack share backup set <backup>
                           [--name <name>]
                           [--description <description>]

* backup: ID or name of backup
* name: Name of backup. Default=None
* description: Backup description. Default=None.


openstack share backup delete <backup>
                              [--force <force>]

* backup: ID or name of a backup.
* force: Allows deleting backup of a share when its status is other than
  "available" or "error". Default=False.


REST API impact
---------------
APIs will be experimental, until some cycles of testing, and the eventual
graduation of them.

**Creating a share backup**::

    POST /v2/share-backups

Request::

    {
        "share_backup": {
            "share": "77eb3421-4549-4789-ac39-0d5185d68c28",
            "name":  "backup_share",
            "description": "This is my backup",
        }
    }

Backup details ``name`` and ``description``  are optional.

If the share is not known to manila, the API will respond with
``404 Not Found``.
If the share is not in available state or share has snapshots,
the API will respond with ``400 Invalid Share``.
If the project's share backup quota has exceeded, the API will
respond with ``413 QuotaError``.

Response(202 Accepted)::

    {
        "share_backup": {
            "share_id": "77eb3421-4549-4789-ac39-0d5185d68c28",
            "created_at": "2016-06-01T21:12:12.617687",
            "updated_at": "2016-06-01T21:12:12.617687",
            "id": "77eb3421-4549-4789-ac39-0d5185d68c29",
            "project_id": "e10a683c20da41248cfd5e1ab3d88c62",
            "display_name": "backup_share",
            "display_description": "This is my backup",
            "status": "creating"
        }
    }

**Delete a backup**::

    DELETE /v2/share-backups/{backup_id}

If the share backup is not known to manila, the API will respond with
``404 Not Found``.
If the share backup state not in ``available`` or ``error``, the API
will respond with ``400 Invalid State``.
API will respond ``202`` if request is accepted.

**Detailed listing of share backups**::

    GET /v2/share-backups/detail

Response(200 OK)::

    {
        "share_backups": [
            {
                "availability_zone": "az1",
                "created_at": "2016-04-02T10:35:27.000000",
                "id": "2ef47aee-8844-490c-804d-2a8efe561c65",
                "display_name": "my_backup",
                "display_description": "this is description",
                "size": 1,
                "status": "available",
                "share_id": "e5185058-943a-4cb4-96d9-72c184c337d6",
            },
            {
                "availability_zone": "az2",
                "created_at": "2016-05-02T10:35:27.000000",
                "id": "2ef47aee-8844-423c-804d-2a8efe561623",
                "display_name": "my_backup_1",
                "display_description": "this is description",
                "size": 2,
                "status": "available",
                "share_id": "e5185058-943a-4cb4-96d9-72c184c33dsd",
            }
        ]
    }

**Restore a share backup**::

    POST /v2/share-backups/{backup_id}/action

Request::

    {
        "restore": null
    }

The backup will be restored in source share (i.e. share from which backup
was created) will be used for restore.
In case, source share or share backup is not known to manila, API will
repsond with ``404 Not Found``.
In case, source share size is different than size of backup, API will
respond with ``400 Invalid Share``.
However, API will respond ``202`` if request is accepted.


**Update a share backup information**::

    PUT /v2/share-backups/{backup_id}

Request::

    {
        "share_backup": {
            "display_name": "test share backup",
        }
    }

If the share backup is not know to manila, the API will respond with
``404 Not Found``.
API will respond ``200`` if request is accepted.


Driver impact
-------------
The backup driver needs to implement these functions::

    def backup(self, backup, share):
        """Create a backup of a specified share.

        The driver should return the backup model with new created
        backup content if creation is successful, manila
        will update the model after this. For example::
        ```
        {"backup":
            {
                "name": "backup_one",
                "id": "e5185058-943a-4cb4-96d9-72c184c33d12",
                "created_at": "2016-04-02T10:35:27.000000",
            }
        }
        ```

        :param backup: backup object.
        :param share: share object.
        :returns: backup: backup object with backup object updated.
        """
        return


    def restore(self, backup, share):
        """Restore a shared backup.

        Driver will restore the specified backup.

        :param backup: backup object.
        :param share: share object.
        """
        return

    def delete(self, backup):
        """Delete a saved backup.

        Driver will delete the share backup.

        :param backup: backup object.
        """
        return


Security impact
---------------

During backup process the data node would have access to read the share
data. If the deny access phase fails, the node will continue forever with
access to the user's data.

Notifications impact
--------------------

None

Other end user impact
---------------------

End user will be unavailable or restricted to other share operations
while it is backing up. Such as: extend/shrink share, replication share,
share-group operation, migration share.

Performance Impact
------------------

None

Other deployer impact
---------------------

The deployer will be able to backup a share.

Developer impact
----------------



Implementation
==============

Assignee(s)
-----------

Primary assignee:
    * kpdev(kinpaa@gmail.com)

Work Items
----------

* Implement core feature.
* Implement backup in NFSBackupDriver.
* Implement backup command in python-manilaclient.
* Implement tempest support.
* Implement manila-ui support.
* Support manila backup in devstack plugin.

Future Work Items
-----------------

* New table 'backup_metadata' where the data service or the user can store some
  useful metadata such as the location of the backup. The metadata can possibly
  used to restore the backup, even in the event of a catastrophic database
  failure.
* Add support to create share from backup, where 'openstack share create' API
  will accept --backup <backup_id> option and create share of the size as that
  of backup. In addition, backup data will be copied to share.
* Add support to handle backup failover in case manila service is interrupted,
  i.e. when service is back online, monitor the backups, check on their
  completion and recover if posssible.
* Restore operation can be enhanced to consider restore to share of different
  size than backup. If restore share is smaller in size, it will be expanded
  before restore and if its larger in size, it will be shrinked before restore.

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

_`[1]`: https://specs.openstack.org/openstack/cinder-specs/specs/kilo/nfs-backup.html

