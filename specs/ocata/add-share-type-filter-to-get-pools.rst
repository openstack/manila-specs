..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================
Support retrieve pools filtered by share-type
=============================================

https://blueprints.launchpad.net/manila/+spec/pool-list-by-share-type


Add feature that allows administrators can get back-end storage pools filtered
by share-type, with that Manila will return the pools filtered by share-type's
extra-specs.

Problem description
===================

Currently Manila's pools list API(detail API included) only supports filter
pools by 'host_name', 'pool_name' and so on. This is not enough when
administrators want to know the pool's status which also can match the
specified share type's requirements. This information is useful when
administrators want to track the usage of each share type.
Administrators also can get all pools and filter them on their own, but
it's more complicated and inefficient. This change intends to cover this
situation and bring more convenience to administrators.

Use Cases
=========

In production environments, administrators often need to have an overall
pools availability statistics grouped by share type, this will help them to
make an adjustment before resources run out.

Proposed change
===============

The modified version of pools list will be identical at the most part, except
for the parts mentioned below:

* A query parameter 'share_type' will be introduced to pools list API(GET
  pools and GET pools/detail).
* In the API service, the share_type's 'extra_specs' property will be
  retrieved and wrapped in 'capabilities' as a filter parameter for method
  'get_pools' in scheduler [1].
* In Manila scheduler's host_manager which currently retrieves the pools dict,
  will add a new logic which would filter all the pools with the
  'capabilities' parameter.
* The logic mentioned above is already existing in Manila [2]
  (_satisfies_extra_specs) which focus on the extra_spec's comparison, so it
  would be better to add an util function in order to reduce code redundancy.
* The Manila client will also add two arguments '--share_type' and '--detail'
  to support this(the '--detail' argument intends to cover the existing
  feature in Manila).

Alternatives
------------

Administrators also can retrieve and filter the pools on their own, but it's
more complicated and inefficient. This change can reduce the request amount
and filter unnecessary data transmitted from server to client.

Data model impact
-----------------

None

REST API impact
---------------

The API microversion will have to be bumped up for the new APIs below.

Pools list API will accept new query string parameter 'share_type'.
Administrators can pass name or id of share type to retrieve pools filtered.

* ``GET /v2/{tenant_id}/scheduler-stats/pools?share_type=share_type1``

Detailed pools list API will also accept new query string parameter
'share_type'. Administrator can pass name or id of the share type to retrieve
pools filtered.

* ``GET /v2/{tenant_id}/scheduler-stats/pools/detail?share_type=share_type1``

Manila client impact
--------------------

As the Manila client already has the pool_list command without 'detail'
feature supported, this spec will add two new command options, which are
'--detail' and '--share_type':

* '--detail' supports show pools information in detail.
* '--share_type' supports filter pools by share_type's extra_specs.

The modified version of command will be like::

    pool_list --share_type s_type1 --detail <other existing command arguments>

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
  TommyLike(tommylikehu@gmail.com)


Work Items
----------

* Add share type filter to pools list and detailed pools list API.
* Add unit tests and tempest tests in Manila.
* Add documentation in Manila.
* Add new command arguments in Manila client.
* Add unit tests and functional tests in Manila client.

Dependencies
============

None

Testing
=======

1. Unit test to test whether share_type filter can be correctly applied.
2. Tempest test whether share_type filter works correctly from API
   perspective.
3. Manila client's unit test to test whether new arguments work correctly.
4. Manila client's functional tests.

Documentation Impact
====================

1. The Manila API documentation will need to be updated to reflect the REST
   API changes.

References
==========

[1] https://github.com/openstack/manila/blob/master/manila/scheduler/rpcapi.py#L72
[2] https://github.com/openstack/manila/blob/master/manila/scheduler/filters/capabilities.py
