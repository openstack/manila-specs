..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============
User Messages
=============

https://blueprints.launchpad.net/manila/+spec/user-messages

For quite some time, OpenStack services have been wanting to be able to send
messages to API end users (by user I do not mean the Operator, but the user
that is interacting with the client).

At the Mitaka cinder mid-cycle `[6]`_, we decided to begin solving this problem
by adding a /messages resource to the API in order for end-users to query for
more information about errors. We should follow suit in manila.


Problem description
===================

If manila operations fail, e.g. create share or create share group, user gets
no detailed information whatsoever. Only the operation status is updated, e.g.
error or failed, without any update to user.


Use Cases
=========

Provide more information to user about failed async operations in following
situations. Below is a list of reasons/messages we could create when
a resource (e.g. a share) goes to error state:

* Shares:
    * Creation fails because no valid host is found - if chain of filters
      returned no hosts, the last filter name is in the message.
    * Creation fails because driver does not expect share-network to be
      provided with current configuration.
    * Creation fails because failed to get share server.
    * Creation fails for an unknown reason.
    * Deletion of access rules failed.
    * Extending error (represented by status too).
    * Shrinking fails because quota update fails.
    * Shrinking fails because of possible data loss.

* Share group:
    * Creation fails because can not choose compatible share-server.
    * Creation fails because of share instance failed to get share
      server.
    * Creation fails for an unknown reason.
    * Creation fails because driver does not expect share-network to be
      provided with current configuration.

* Share group snapshot:
    * Creation fails for an unknown reason.
    * Deletion fails for an unknown reason.

* Share replica:
    * Creation fails because an 'ACTIVE' replica must exist in 'AVAILABLE'
      state to create a new replica for share.
    * Creation fails because driver does not expect share-network to be
      provided with current configuration.
    * Creation fails because failed to get share server for share replica
      creation.
    * Creation fails for an unknown reason.
    * Deletion of access rules failed.
    * Update of share replica fails because of a driver error.

* Snapshot:
    * Creation fails for an unknown reason.
    * Creation of replicated snapshot fails for an unknown reason.
    * Deletion of access rules failed.
    * Revert to snapshot fails for an unknown reason.

* Snapshot replica:
    * The driver was unable to delete some or all of the share replica
      snapshots on the backend/s.
    * Update fails because replica snapshot was not found on the backend.
    * Update fails because of a driver error.

The above list reflects a subset of situations where error state is set for a
resource in the share service `[3]`_. `[4]`_ shows creation of scheduling
error messages.

Proposed change
===============

When an error occurs during an async operation, following details about the
error are stored in database:
* resource type and ID for the resource on which the error occurred
* action during which the error occurred
* error detail (user-friendly error explanation)
* error level
* expiration time
* request ID
* project ID

Because exceptions raised by drivers or manila services may contain sensitive
information (e.g. about backend infrastructure, host names...), this exception
is not exposed to user, instead only sanitized error detail is exposed.

User can get messages either directly through the new /messages API or through
manila CLI which allows showing a message, listing all messages and deleting
existing messages. This spec doesn't address integration with Horizon.

The same approach is already implemented in cinder.

Alternatives
------------

* User facing notifications - use the existing notification framework in
  combination with an AMQP consumer to pull messages off and provide an
  endpoint for the user. Faults with this approach are that we do not want to
  display the current information in notifications to the user and it will
  require many more services as dependencies.

* Retrieving user messages via a separate service (such as zaqar) This approach
  suggests storing user messages in another service that the user could query
  or the service could utilize webhooks to notify the user. One
  major drawback to this approach is the complexity in writing bindings for the
  separate service(s) and the need for a separate service as a dependency.

What this specification does not solve
--------------------------------------

* State change notifications. This solution does not intend to solve the
  use-case of alerting users when a share or any other resource changes state.
  For example, when a share changes its status from CREATING to AVAILABLE.

Data model impact
-----------------

New messages table in database to store all messages. This table may prove to
grow large in a cloud with lots of errors. The admin will be able to utilize
the expires_at column to delete old messages by a manila-manage CLI command:

* id = Column(String(36), primary_key=True, nullable=False)
* project_id = Column(String(36), nullable=False)
* message_level = Column(String(32), nullable=False)
* request_id = Column(String(255), nullable=True)
* resource_type = Column(String(255))
* resource_uuid = Column(String(36), nullable=True)
* action = Column(String(255), nullable=False)
* expires_at = Column(DateTime, nullable=True)
* detail = Column(String(255), nullable=True)
* deleted = Column(String(36), default='False')

Only a constant will be stored in 'detail' column and there will be a mapping
 (in a file) from these constants to user friendly messages:

message_map = {
    MessageIds.UNEXPECTED_NETWORK: _("Current back end configuration does not "
                                     "support creating shares within  project "
                                     "defined share-networks."),
    MessageIds.QUOTA_UPDATE: _("Failed to update quota.")
}

REST API impact
---------------

Message APIs:

* List messages

  * URL: /v2/<tenant>/messages
  * Method: GET (200, 400)
  * URL args:

    * offset - integer, set offset to define start point of message listing,
      0 or bigger
    * limit - integer, maximum number of messages to return, 1 or bigger
    * sort_key - string, key to be sorted (i.e. 'created_at' or
      'resource_type')
    * sort_dir - string, sort direction, should be either 'asc' or 'desc'.

  * JSON body:

    .. code-block:: json

      {
        "messages": [
          {
           "id": "5429fffa-5c76-4d68-a671-37a8e24f37cf",
           "action": "ALLOCATE_HOST",
           "user_message": "No storage could be allocated for this share
                           request. Trying again with a different size or
                           share type may succeed.",
           "message_level": "ERROR",
           "resource_type": "SHARE",
           "resource_uuid": "f292cc0c-54a7-4b3b-8174-d2ff82d87008",
           "created_at": 2015-08-27T09:49:58-05:00,
           "expires_at": 2015-09-27T09:49:58-05:00,
           "request_id": "req-936666d2-4c8f-4e41-9ac9-237b43f8b848",
          }
        ]
      }

* Show message

  * URL: /v2/<tenant>/messages/<message_id>
  * Method: GET (200, 403 Forbidden, 404 Not Found)
  * JSON body:

    .. code-block:: json

      {
        "message": {
          "id": "5429fffa-5c76-4d68-a671-37a8e24f37cf",
          "action": "ALLOCATE_HOST",
          "user_message": "No storage could be allocated for this share
                          request. Trying again with a different size or
                          share type may succeed.",
          "message_level": "ERROR",
          "resource_type": "SHARE",
          "resource_uuid": "f292cc0c-54a7-4b3b-8174-d2ff82d87008",
          "created_at": 2015-08-27T09:49:58-05:00,
          "expires_at": 2015-09-27T09:49:58-05:00,
          "request_id": "req-936666d2-4c8f-4e41-9ac9-237b43f8b848",
        }
      }

* Delete message

  * URL: /v2/<tenant>/messages/<message_id>
  * Method: DELETE (204, 403 Forbidden, 404 Not Found)

Driver impact
-------------

None

Security impact
---------------

Only message type (instead of internal error message) is kept in
database and exposed to user to assure that no sensitive information is
exposed. For example back-end driver might expose sensitive information
or infrastructure hostnames.

Notifications impact
--------------------

None

Other end user impact
---------------------

Users will be able to list, show and delete user messages through the new
API or CLI commands.

Performance Impact
------------------

None

Other deployer impact
---------------------

* New configuration option message_ttl that will dictate the number of seconds
  after the messages creation time to set the ‘expires_at’ attribute on
  generated messages.

* The messages table will be potentially large and may be
  reaped based on the ‘expires_at’ column. Where all messages with a
  expires_at date earlier than the current time can be safely deleted.

Developer impact
----------------

Developers should be aware of use-cases where the user needs information about
an error. In these situations, an appropriate user message should be written
and creation of the message added in the specific code path(s).


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Jan Provaznik
  Alex Meade

Alex Meade is author of the messages approach, this spec is mostly based on
Alex's existing patch (see references).

Work Items
----------

* Add new database table and migration for storing messages.
* Add /messages API `[1]`_.
* Extend CLI with message commands `[2]`_.
* Extend Horizon UI with an interface for listing, showing and deleting
  messages.
* Add "message create" calls on proper places in scheduler and share
  services `[3]`_ and `[4]`_.
* Add a new "manila-manage message delete-expired" command for deleting expired
  messages from database.


Dependencies
============

None


Testing
=======

Tempest tests should be written and run in the gate. It may prove difficult to
implement complete functional testing of the feature as messages will not be
created unless there is an error, which may be difficult to trigger. However,
some operations are easy to trigger failure with unlimited quotas. One example
is creating a share too big to be stored on the backend.


Documentation Impact
====================

* REST API documentation.
* New config option, message_ttl (time to live).
* New API policies for messages.


References
==========

* _`[1]` Alex Meade's existing patch (not up-to-date, it doesn't reflect this
  spec precisely): https://review.openstack.org/#/c/313549/
* _`[2]` CLI message commands: https://review.openstack.org/#/c/429614/
* _`[3]` messages added into share service:
  https://review.openstack.org/#/c/443101/
* _`[4]` messages added into scheduler service:
  https://review.openstack.org/443102
* _`[5]` A related cinder spec (which is more generic though):
  https://specs.openstack.org/openstack/cinder-specs/specs/newton/summarymessage.html
* _`[6]` cinder Mitaka midcycle notes:
  https://etherpad.openstack.org/p/mitaka-cinder-midcycle-user-notifications
* _`[7]` cinder user messages improvement spec:
  https://review.openstack.org/#/c/451761/
