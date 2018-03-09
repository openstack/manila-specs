..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Support metadata for access rule
=================================

https://blueprints.launchpad.net/manila/+spec/metadata-for-access-rule

Add a new "metadata" property for access rule resource.

Problem description
===================

Access rule resource lacks the ability for getting/setting metadata property.


Use Cases
=========

The metadata here for access rule is the descriptive metadata. It's used for
discovery and identification. Users could add key-value pairs for the access
rules to describe them. Users could also filter access rules with specified
metadata while requesting a list of access rules.


Proposed change
===============

* The "metadata" property will be added to access rule object.
* A new DB table "share_access_rules_metadata" will be created.
* The access rule create API will be updated to support "metadata".
* A set of APIs will be created for access rule metadata's CRUD.

Alternatives
------------

We can only get the desired access rule value by access_to. There is no
way to tag what the access rule is by the user or add some special info
to access.

Data model impact
-----------------

The new "share_access_rules_metadata" table will be created in DB and it
will include access rule id. The primary key is "id". We could get
information of the access_rules table by the access rule id.

::

    +-------------------+--------------+-------------+-------------+-----------------+
    |     key           | value        | id          | deleted     |    access_id    |
    +-------------------+--------------+-------------+-------------+-----------------+
    |    varchar(255)   | varchar(255) | varchar(36) | varchar(36) |    varchar(36)  |
    +-------------------+--------------+-------------+-------------+-----------------+
    |    varchar(255)   | varchar(255) | varchar(36) | varchar(36) |    varchar(36)  |
    +-------------------+--------------+-------------+-------------+-----------------+`

REST API impact
---------------

* It will follow the microversion rules.
* The access rule create API's request body will be updated to support "metadata".

  ::

    POST /v2/{project_id}/shares/{share_id}/action

    the request body can contain "metadata".
    {
        "allow_access":{
            "metadata":{
                "key1": "value1",
                "key2": "value2"
            }
            ...
        }
    }

* A set of new APIs related to access rule metadata will be created.

  * Show an access rule's metadata

    ::

      GET    /v2/{project_id}/share-access-rules/{access_id}

      Response
      {
          "access": {
              "access_level": "rw",
              "state": "error",
              "id": "507bf114-36f2-4f56-8cf4-857985ca87c1",
              "access_type": "cert",
              "access_to": "example.com",
              "access_key": null,
              "metadata": {
                  "key1": "value1",
                  "key2": "value2"
              }
          }
      }

  * List access rules filtered by access rule metadata

    ::

      GET   /v2/{project_id}/share-access-rules?share_id={share-id}&key1=value1&key2=value2

      Response
      {
          "accesses": [
              {
                      "access_level": "rw",
                      "state": "active",
                      "id": "507bf114-36f2-4f56-8cf4-857985ca87c1",
                      "access_type": "cert",
                      "access_to": "example.com",
                      "access_key": null,
                      "metadata": {
                          "key1": "value1",
                          "key2": "value2"
                      }
              },
              {
                      "access_level": "rw",
                      "state": "error",
                      "id": "329bf795-2cd5-69s2-cs8d-857985ca3652",
                      "access_type": "ip",
                      "access_to": "10.0.0.2",
                      "access_key": null,
                      "metadata": {
                          "key1": "value1",
                          "key2": "value2"
                      }
              },
          ]
      }

    * The "share_id" is a mandatory query key, and the API will respond with
      HTTP 400 if the "share_id" is not provided.

    .. note::

       The current `access rules list API
       <https://developer.openstack.org/api-ref/shared-file-system/#list-access-rules>`_
       accepts HTTP POST requests. To ensure correct HTTP semantics around
       idempotent and safe information retrieval, we will introduce a new API that
       accepts GET requests. The old API will be capped with a maximum micro-version,
       i.e, it will not be available from the micro-version that this new API is
       introduced within.

  * Remove one specified metadata

    ::

      DELETE   /v2/{project_id}/share-access-rules/{access_id}/metadata/{key}

    * If we don't input the "key" value, manila won't remove any metadata and
      return HTTP 400.

  * Update a specified metadata

    ::

      PUT   /v2/{project_id}/share-access-rules/{access_id}/metadata

      Request
      {
          "metadata":{
              "key1": "value1",
              "key2": "value2"
          }
      }

    * If we don't input the "key" value, it won't update any metadata
      and return error.

Security impact
---------------

None

Notifications impact
--------------------

The new APIs will send new notifications as well.

Other end user impact
---------------------

The Manila client, CLI will be extended to support access metadata.

* The access-allow command with access metadata supported will be like::

    manila access-allow [--metadata <key=value> [<key=value> ...]]
                        [--access-level <access_level>]
                        <share> <access_type> <access_to>

* The new access-metadata command will be like::

    manila access-metadata <access_id> <action> <key=value> [<key=value> ...]

    Set or delete metadata on a access rule.

    Positional arguments:
    <access_id>     ID of the access rule to update metadata on.
    <action>     Actions: "set" or "unset" to set or delete metadata on a access rule.
    <key=value>  Metadata to set or unset, only key is necessary to unset.


* The new access-show command with access metadata supported will be like::

    manila access-show <access_id>

    Show information of given access rule.

    Positional arguments:
    <access_id>       ID of the access rule to update metadata on.


* The access-deny command will delete all access rule metadata. The command syntax
  won't be changed::

    manila access-deny <share> <id>

* The access-list command will add metadata filter. The command will be like::

    manila access-list [--columns <columns>] <share>
                       [--metadata [<key=value> [<key=value> ...]]]
    Show access list for share.

    Positional arguments:

    <share> Name or ID of the share.

    Optional arguments:

    --columns  Comma separated list of columns to be displayed
    example –columns “access_type,access_to”.

    --metadata Filters results by a metadata key and value.
    OPTIONAL: Default=None.

Performance Impact
------------------

A new "share_access_rules_metadata" table will be created. The DB join action
may cause the search performance to reduce on the existing access rules APIs.

Other deployer impact
---------------------

None

Developer impact
----------------

Drivers will not have access to the metadata, and the driver interfaces for
update_access will not be modified.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  zhongjun

Work Items
----------

* Add metadata property to access rule object and bump the APIs version.
* Create a new DB table "share_access_rules_metadata" and add db upgrade script.
* Update access rule create/list API.
* Add update/delete APIs for access rule metadata.
* Add new tests within openstack/manila-tempest-plugin for the new APIs.
* Allow adding/updating/deleting access rule metadata in Manila UI and python-manilaclient.

Dependencies
============

None


Testing
=======

* Unit tests
* Tempest tests


Documentation Impact
====================

* Api-ref needs update.
* update user documentation and CLI documentation


References
==========

None
