..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============
Remove v1 API
=============

https://blueprints.launchpad.net/manila/+spec/remove-v1

Remove the long-deprecated v1 API.

Problem description
===================

The v1 API has been deprecated pretty much the entirety of Manila's time as an
independent project. We do not wish to focus on it when adding JSON Schema
schemas (as part of the OpenAPI effort) and its continued presence makes that
effort harder than is necessary. The removal of this API is long-overdue.

Use Cases
=========

* As a developer, I don't want to have to worry about multiple APIs.

Proposed change
===============

Remove the API, merging any shared code into the v2 API.

Alternatives
------------

None.

Data model impact
-----------------

None.

REST API impact
---------------

Requests to the ``/v1`` API will be rejected.

Driver impact
-------------

None.

Security impact
---------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

None. Most client tooling already used v2 API.

Performance Impact
------------------

None.

Other deployer impact
---------------------

Deployers will need to delete any endpoint for the v1 API upon upgrade.

Developer impact
----------------

None. The API code will be simplified.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  stephen.finucane

Work Items
----------

* Move or copy any common code from ``manila.api.v1`` to ``manila.api.v2``
* Remove tests for v1 API
* Delete the v1 API
* Update documentation


Dependencies
============

None.


Testing
=======

We will remove tests for the v1 API. v2 API tests will remain unchanged.


Documentation Impact
====================

References to the v1 API will be removed. A release note will be added.


References
==========

None.
