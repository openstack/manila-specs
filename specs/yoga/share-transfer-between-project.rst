..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============================
Transfer share between project
==============================

https://blueprints.launchpad.net/manila/+spec/transfer-share-between-project

Add support for share transfer that allows a user to transfer a share to
another user under other project, just like cinder transfer-create,
transfer-accept, transfer-delete.
There are two scenarios, the first is DHSS=False, the second is DHSS=True.
In the first scenario, will only support transferring shares that are not tied
to a share network. The object of the transfer is Share.
In the second scenario, we would support transferring share networks, which
would cause all of its shares to be migrated within. This will be completed in
another spec.


Problem Description
===================

In an actual OpenStack production environment, it is common for a user to
transfer a storage resource to another user for collaborative pipeline work.
Manila Share also needs a secure way to transfer shares so that other users
can receive them safely.

Use Cases
=========
* The user in project A pre-populate data for another user in project B.
* When a project is "ending"/being decommissioned, share data can be
  transferred to a project that's still needed.
* Transfer share to specific user to collaborate on the rest of the work.

Proposed Change
===============
This spec proposes to:

* Add a new table, which named ``transfers``. Includes the following fields:

  * ``id`` the primary key of table.
  * ``resource_type`` the type of resource which is being transferred, this can
    be share, share network and so on.
  * ``resource_id`` the uuid of resource which is being transferred.
  * ``name`` the display name of transfer.
  * ``salt`` a short random string used to calculate crypt_hash, after create
    transfer will return auth_key(also a short random string), salt and
    auth_key calculate crypt_hash by sha1.
  * ``crypt_hash`` the hash value for transfer.
  * ``expires_at`` the expire time of transfer, one hour(default) later after
    created.
  * ``source_project_id`` the source project id of share.
  * ``destination_project_id`` the destination project id of share.
  * ``accepted`` whether the share has been accepted.

* Add a new status ``awaiting_transfer`` for shares, which is being
  transferred.

* Check the following before create share transfer:

  * check the status of share and share_snapshot and share_replica before
    create transfer, share and all share_snapshot must be ``available``.
  * check whether the share in share group, share cannot be in a share group.
  * check whether share has share network, this spec just supports the first
    scenario (DHSS=False), share without share network will allowed to be
    transferred.

* Check the following before accept transfer:

  * check quota of destination project, include project quota, project user
    quota, project share type quota, per_share_gigabytes, make sure destination
    project has enough remaining quota.
  * if the share type of source share is not public, and the share type access
    of share type does not include the destination project, it will fail to
    accept transfer.

* For security reasons, a share transfer will be cancelled automatically if it
  is not accepted one hour(default) after creation. There will be a periodic task that
  checks for expire transfers every 5 minutes.

* Add a new configure item ``wait_transfer_timeout_seconds``, default is 3600
  seconds.

* "transfer-accept" need to call driver to complete project_id change, such as
  CephFS driver stores the project_id as metadata. The parent class driver only
  defines the function, but does not implement it.

* When accepting a transfer, users can choose whether need to clear the share
  access rule. By default, the share access rule will not be cleared.

* Add new API for transfer-create, transfer-accept, transfer-delete, transfer-
  list, transfer-show.

* The above API changes will bump a new microversion.

Alternatives
------------

* An alternative here is manage/unamanage, a share in Project A can be
  "unmanaged" by a privileged user and "managed" into another project. However
  two issues with that approach:

  * manage/unmanage is meant to be disruptive - pre-existing access rules will
    be flushed and the share would be re-exported.
  * you'd need to be a privileged user by virtue of the default RBAC because
    this approach gives away some of teh abstraction of the cloud.

Data model impact
-----------------

* A new table ``transfers`` will be added.

REST API impact
---------------

* Create a share transfer

POST /v2/share-transfers

Request Example:

.. code-block:: json

   {
       "transfer":
       {
           "share_id": "da8eb12e-123c-49ea-ae2b-5d42f02fa00e",
           "name": "share transfer"
       }
   }

Response Example:

.. code-block:: json

   {
       "transfer":
       {
           "auth_key": "qbgcabtdbad29r08",
           "created_at": "2021-12-27T11:29:46.743632",
           "name": "share transfer",
           "share_id": "da8eb12e-123c-49ea-ae2b-5d42f02fa00e",
       }
   }

auth_key is necessary in accept share transfer, please keep records properly,
since it cannot be retrieved later on.

* Accept a share transfer

POST /v2/share-transfers/{transfer_id}/accept

Request Example:

.. code-block:: json

   {
       "accept":
       {
           "auth_key": "6461646164641397",
           "clear_access_rules": "false",
       }
   }

* List share transfers for a project

GET /v2/share-transfers

* List share transfers and details

GET /v2/share-transfers/detail

* Show share transfer detail

GET /v2/share-transfers/{transfer_id}

* Delete share transfer

DELETE /v2/share-transfers/{transfer_id}

Security impact
---------------

* Each of these transfer APIs will be protected by RBAC.
* Users can only transfer shares belonging to their project.
* The transfer ID and transfer auth key  are expected to be conveyed out of
  band. The transfer auth key is only returned by the API during creation of a
  transfer. There will be no way to retrieve the transfer auth key after a
  transfer has been created.
* the choice of cryptographic algorithms to generate an auth_key and the fact
  that transfer IDs are non-guessable UUIDs will provide additional security
  to the process.
* Since transfers expire in an hour(default), there's a time bound protection
  to this critical change of namespaces.

Notifications impact
--------------------

We must notify to external listeners when a volume transfer has been initiated
and when it has been accepted.

Other end user impact
---------------------

The Manila client, CLI will be extended to support share transfer.

* The command of create share transfer will be like::

    openstack share transfer create <share>

* The command of accept share transfer will be like::

    openstack share transfer accept [--clear-rules] <transfer> <auth_key>

* The command of list share transfer will be like::

    openstack share transfer list

* The command of show share transfer detail will be like::

    openstack share transfer show <transfer>

* The command of delete share transfer will be like::

    openstack share transfer delete <transfer>

Performance Impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

Drivers that use project and user based metadata must implement the new driver
interface to complete "transfer_accept".


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  haixin <haix09@chinatelecom.cn>


Work Items
----------

* Update API.
* Update Database
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

* OpenStack User Guide
* OpenStack Contributor Guide
* OpenStack API Reference

References
==========

Based on cinder transfer strategy. Includes the following:

* https://blueprints.launchpad.net/cinder/+spec/improve-volume-transfer-records
* https://blueprints.launchpad.net/cinder/+spec/transfer-snps-with-vols
