..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
Update share-type's metadata
============================

https://blueprints.launchpad.net/manila/+spec/update-share-type-name-or-description

This blueprint proposed to update the ``name``, ``description`` and/or
``share_type_access:is_public`` attribute of ``share-type`` after create
a share type, that support update the ``name``, ``description`` and/or
``share_type_access:is_public`` attribute of a share-type.

Problem description
===================
Currently, only the ``name``, ``description`` and/or
``share_type_access:is_public`` of share-type is set when the share-type is
created, and not allowed to be edited after the share-type is created. We can
only set extra spec for share-type. But not ``name``, ``description`` and/or
``share_type_access:is_public`` for share-type.

Use Cases
---------
As a user, I would like to update the attribute of share-type, and to update
its ``name``, ``description`` and/or ``share_type_access:is_public``
of share-type after created. I would prefer this when I need a new share-type
and do not have to create a new one.

Proposed change
===============
Add a new microversion to the share type API.

Add a new function API to the **Share types**.

* Add a update share-type API, available in a new micro version of
  the manila API:

  - Add ``update`` method to ``ShareTypesController`` class.

  - Add ``name``, ``description`` and/or ``share_type_access:is_public``
    parameters to ``share_type`` of the request body.

  - Make the ``name`` and ``description`` parameters optional. And their
    length not more than 255. ``share_type_access:is_public`` parameters
    is also optional, but it's a **boolean** type.

    If ``name`` is **NULL**, the name of share type will not be updated.

    If ``description`` is **NULL**, the description of share type will
    not be updated.

    If ``share_type_access:is_public`` is **NULL**, the public access of
    share type will not be updated.

    If ``name`` is "", the name of share type is not valid.

    If ``description`` is "", the description of share type will be empty.

Alternatives
------------
None

Data model impact
-----------------
None

REST API impact
---------------
* URL:
  * /v2/{project_id}/types/{share_type_id}

* Request method:
  * PUT

* Normal http response code(s):
  * 200(OK)

* Expected error http response code(s):
  * 400(Bad Request), 401(Unauthorized), 403(Forbidden),
  404(Not Found), 409(Conflict)

Update share-type request will accept the following payload ::

    {
        "share_type": {
            "name": "testing",
            "description": "share type description",
            "share_type_access:is_public": true
        }
    }

Security impact
---------------
None

Notifications impact
--------------------
None

Other end user impact
---------------------
User will be able to update share-type attribute after created. Manila client
may add help to inform users about this new function.

Performance Impact
------------------
None

Other deployer impact
---------------------
None

Developer impact
----------------
None

Upgrade impact
--------------
None

Implementation
==============
Assignee(s)
-----------
Primary assignee:
  haixin

Other contributors:
  brinzhang

Work Items
----------
* Add update share-type API to the **Share types**.
* Add update share-type API to manilaclient, and it will be supported
  by 'manila type-update'.
* Add update share-type API to **manila-ui** project.
* Add functional tests.
* Add units tests.
* Add tempest tests.

Dependencies
============
None

Testing
=======
* Add related unittest
* Add related functional test
* Add tempest tests

Documentation Impact
====================
Add update share-type API information to docs.

References
==========
None

History
=======
.. list-table:: Revisions
      :header-rows: 1

   * - Release Name
     - Description
   * - Train
     - Introduced
