..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================================
Show resource's total count info in share list APIs
====================================================

https://blueprints.launchpad.net/manila/+spec/add-amount-info-in-list-api


This blueprint proposes adding total count info in Manila's /shares
and /shares/detail APIs.

Problem description
===================

Show how many resources a user or tenant has is usually required in the web
portal's summary section, but we cannot get this info now. In order to show
this kind of total number to the end user, many clouds have to collect all of
the resources.

Use Cases
=========

The administrators or users to know how many resources they have in total
without retrieving and accumulating them all when it return just a slice
of paginated list.

Proposed change
===============

According to the openstack API-WG `[1]`_ proposal. It provide guidance on
returning the total size of a resource collection in a project's public REST
API.

So this bp proposes to add new attribute ``count`` in our /shares and
/shares/detail APIs to having the total count information in /shares and
/shares/detail APIs response. If we add the ``count`` attribute
into our response body in default, it could has a performance impact if
we have a mount of resources, considering not every request requires
this kind of info, the additional query parameter is required to turn
this on when listing resource.

Alternatives
------------

There are few alternative solutions for this requirement, let's list and
compare them all here.

1. Add amount information in response header::

    X-resource-count: 200

The disadvantage of this is it will add some burden to the API customers,
we don't have any history for adding useful content in response
headers. Also it makes more difficult for documentation.

2. Add explicit API for each resources::

    POST: /v2/{tenant_id}/{resources}/action {os-count}

This change involves a lot of modifications and creates several new APIs.
Adding this amount of APIs for such a simple API change is overdesign.

3. Create a new API for different resources::

    POST: /V3/{tenant_id}/action {os-count}
    BODY:
        {
            "resource": "share",
            "user": "user_1",
            "other_filter": "other_value"
        }

For this change, one more API request is required if the end user wants to
know how many resources in total when listing resources.


Data model impact
-----------------

None

REST API impact
---------------

Microversion bump is required for this change.

Add the 'with_count' parameter in query string in share list API, it
is used to indicate if the total count of resources should or should
not be returned from a GET REST API request. Any value that equates to
True indicates that the count should be returned. Conversely, any value
that equates to False indicates that the count should not be returned.

* List shares

  * URL: /v2/{tenant_id}/shares?with_count=true
  * Method: POST
  * JSON body:

    .. code:: json

        {
            'shares': [{
                'name': 'test',
                ...
            }]
            'count': <int_count>,
        }

* List shares with details

  * URL: /v2/{tenant_id}/shares/detail?with_count=true
  * Method: POST
  * JSON body:

    .. code:: json

        {
            'shares': [{
                'name': 'test',
                ...
            }]
            'count': <int_count>,
        }

Client impact
-------------

The share list command will be upgraded to support this.

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

Since we will add additional ``COUNT()`` statement if the list
APIs are requested with the ``with_count`` option, there would be a
performance impact on those APIs, especially when there are a lot
of data in database.

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
  zhongjun(jun.zhongjun2@gmail.com)

Work Items
----------

* Add ``with_count`` option support in share list APIs.
* Add related unit testcases.
* Update manila-client.
* Update manila-ui.

Dependencies
============

None

Testing
=======

* Add unit tests and tempest test to cover this change.

Documentation Impact
====================

Update API documentation.

References
==========

_`[1]`: http://specs.openstack.org/openstack/api-wg/guidelines/counting.html




