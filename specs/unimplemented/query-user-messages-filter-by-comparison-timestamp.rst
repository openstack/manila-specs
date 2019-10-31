..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================================
Support to query user messages filtering by timestamp comparison
================================================================

https://blueprints.launchpad.net/manila/+spec/query-user-message-by-timestamp

Add support for querying user messages by specifying a timestamp, which will
be compared to the created_at field, and Manila will return all the messages
that match to the given time condition.

Problem description
===================

Manila API only support filtering user messages by discrete values (such as
message_id or resource_id), even though user messages have timestamp fields
as created_at, users can not query by a given period. Users may be interested
in user messages in specific periods in order to analyze causes of
asynchronous errors. Currently they need to retrieve and filter the resource
messages by themselves. This change will improve the usability when searching
for user messages and the efficiency of timestamp based queries.

Use Cases
=========

In large scale environment, lots of asynchronous error messages may be created
. In order to quickly locate the cause of the error, user or system manager
only need to get user messages corresponding to the resource which was created
during a specified time period, instead of querying all asynchronous error
messages of the resource.


Proposed change
===============

The following changes are proposed:

* Introduce two additional parameters in the url that searches for user
  messages, being both timestamps.

  * **created_before** Return results older than the specified time.
  * **created_since** Return results since the specified time.


Alternatives
------------

None

Data model impact
-----------------

None

REST API impact
---------------

List user messages API will accept new query string parameters. User can add
``created_before`` to the url to query user messages older than the specified
time and ``created_since`` to query user messages newer than the specified
time, or use both of them to query user messages that were created during the
specified time interval. This changes also need to bump the microversion of
API to keep forward compatibility.

* GET /v2/{project_id}/messages?created_since=2019-11-01T01:00:00&created_before=2019-11-02T01:00:00

Security impact
---------------

None

Notifications impact
--------------------

None.

Other end user impact
---------------------

One command will be updated::

  manila message-list [--resource_id <resource_id>]
                      [--resource_type <type>] [--action_id <id>]
                      [--detail_id <id>] [--request_id <request_id>]
                      [--level <level>] [--limit <limit>]
                      [--offset <offset>] [--sort-key <sort_key>]
                      [--sort-dir <sort_dir>] [--columns <columns>]
                      [--create-before <b_time>] [--create-since <s_time>]


Python client may add help to inform users this new filter, and will update
the corresponding cli command.

Performance Impact
------------------

Performance can be improved for timestamp based queries while using the
database engine. Additionally, using indexes, the query performance will be
enhanced.

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

* Add API filter
* Add querying support in sql
* Add related unit and functional tests
* Add python-manilaclient support
* Add docs update


Dependencies
============

None


Testing
=======

1. Unit test to test if those filters can be correctly applied.
2. Tempest test if change filter work correctly from API perspective.

Documentation Impact
====================

1. The manila API documentation will need to be updated to reflect the REST
   API changes.

References
==========

None
