..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================================
Support retrieving shares filtered by export-location
=====================================================

https://blueprints.launchpad.net/manila/+spec/support-filter-share-by-export-location

Currently we can only filter the shares by share name, status, share type,
extra specs, share server id, metadata, snapshot, host, share network, project
id, and consistency group, but this is too limited for user's various cases.
Manila doesn't support filtering the shares by export location. This spec
intends to add the ability for Manila.


Problem description
===================

It's a common use case to list NAS shares with filters, and Manila has
the API to cover this, but it does not satisfy the situation when export
location filter is desired. The main problem is that having export location
we don't have possibility to get to know which manila share owns it.

Use Cases
=========

When user has a share that it has been mounted in VM, and user only knows the
share export location path. User often wants to quickly filter a share by
share export location in Manila. This change will help him achieve this.

Proposed change
===============

The modified version of shares and share-instances list will be identical at
the most part, except for the parts mentioned below:

* A query parameter 'export_location' will be introduced to shares and
  share-instances list API (GET shares and GET shares/detail, GET
  share-instances and GET share-instances/detail).
* In the API service, the export_location will be added in share and share
  instance search options. In Manila DB's method, will add a new export
  location filter in SQLAlchemy API, and add a new logic which would filter
  all the shares or share instances with the export location.
* The Manila client will also add one argument '--export_location' to support
  this.

Alternatives
------------

We could get share instance ID from share export location path, and use
"share-instance-show <instance>" command to find out associated share.

Data model impact
-----------------

None

REST API impact
---------------

The API microversion will have to be bumped up for the new APIs below.

Share and share-instances list API will accept new query string parameter
'export_location'. Administrators can pass path and id of export_location
to retrieve shares filtered.

* ``GET /v2/{tenant_id}/shares?export_location=test_path``
* ``GET /v2/{tenant_id}/share-instances?export_location=test_path``

Detailed share and share-instance list API will also accept new query string
parameter 'export_location'. Administrator can pass path and id of the export
location to retrieve shares filtered.

* ``GET /v2/{tenant_id}/shares/detail?export_location=test_path``
* ``GET /v2/{tenant_id}/share-instances/detail?export_location=test_path``

The search will be among "allowed", for user, shares - all his shares and
public ones.

Manila client impact
--------------------

Manila client will add a new parameter '--export_location <export-location>'
to command 'list', the modified version of command will be like::

* list --export_location <export-location> <other existing command arguments>
* share-instance-list --export_location <export-location> <other existing command arguments>


Security impact
---------------

None

Notifications impact
--------------------

None.

Other end user impact
---------------------

None.

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
  zhongjun2(jun.zhongjun2@gmail.com)


Work Items
----------

* Add 'export_location' parameter to 'list' collection.
* Add related tests in Manila.
* Add documentation in Manila.
* Add new argument for command 'list' in Manila client.
* Add related tests in Manila client.

Dependencies
============

None


Testing
=======

1. Unit and tempest tests whether new API works correctly.
2. Manila client's unit tests and functional tests on new added argument.

Documentation Impact
====================

1. The Manila API documentation will need to be updated to reflect the REST
   API changes.

References
==========
