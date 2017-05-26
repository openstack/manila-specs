..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================
Flexible API filters
=====================

https://blueprints.launchpad.net/manila/+spec/like-filter

This spec proposes changing API filter behavior to inexact matching of filter
values.

Problem description
===================

Currently, we could only filter manila resource by exact match. This is
not flexible enough, because customer needs to remember full share name,
and can not get shares quickly by key words. However for customer, if partly
alike filters are supported to retrieve the shares via user input, that
should be useful for user to filter shares by name flexibly, especially for
the users who have a lot of shares which are managed by manila.

By the way, it's already introduced in many other projects, we can take
advantage of those some existing mechanism, nova `[1]`_ and ironic `[2]`_.
In nova, it change for regex filter matching in sqlalchemy. In ironic, it
add a way to filter nodes in the API by their name (regex and wildcard).

Use Cases
=========

User has really big amount of shares. He named them using some different
templates based on his needs. And he may want to filter out shares based
on some part of share names. The same situation with "description".

Proposed change
===============

It's worth to mention here, that this feature is also been proposed in
cinder `[4]`_ and it's depended on the generalized resource filter
`[5]`_ , but making the API behavior configurable is still open to
question in manila, so we would only pick the part we'd like to have
while keep the API consistency with cinder.

Use '~' operator after the filter key to trigger the like filtering
operation, this is also how our API will look like::

   GET /v2/<tenant_id>/shares?name~=test

Support like filter only on specified resources (cinder let the
administrator to have that decision by configuration file):

   * list shares (name, description)
   * list snapshots (name, description)
   * list share-networks (name, description)
   * list share-groups (name, description)

The Manila client will add a new command argument '--name~' and
'--description~' to 'list' APIs to filter manila resources by inexact match.

We can provide resource filtering based on regex filter, but there
is a possibility that we could have ReDos `[3]`_ attack. Considering the
filter is flexible and safe enough for this case. We could introduce 'LIKE'
operator. And we can easily apply this filter at the existing sqlalchemy
filtering function, and use 'LIKE' operator to filtering resource.

Alternatives
------------

There is an option that we can deploy searchlight which mainly focus on
various cloud resource querying, and that's a more wider topic. But we
should also consider the cloud environment that don't contain searchlight.

Also, there is another option that user can gather all the raw data and
do the filtering on their own, but it's obvious that is what we try
to avoid, cause it costs a lot of unnecessary resource expense.

Data model impact
-----------------

None

REST API impact
---------------

Microversion bump is required for this change.

* Add ``name~`` and ``description~`` parameter to list API
  interfaces. It means we can use this new parameter to inexact filtering::

    List shares with share name

    GET /v2/<tenant_id>/shares?name~=test

    Accept: application/json

    JSON Response

    {
       "shares": [
             {
                "status": "available",
                "export_location": %ip%:/opt/stack/data/manila/mnt/share-%share_id%,
                "name": "test_1",
                ...
           },
           {
                "status": "available",
                "export_location": null,
                "name": "2_test",
                ...
           }
       ]
    }


    List snapshot with share snapshot name

    GET /v2/<tenant_id>/snapshots?name~=snap_test

    Accept: application/json

    JSON Response

    {
       "shares": [
           {
                "status": "available",
                "name": "snap_test_1",
                ...
           },
           {
                "status": "available",
                "name": "snap_test_xxxxx",
                ...
           }
       ]
    }

Client impact
-------------

The Manila client will add a new command argument '--name~'
and '--description~' to 'list' to filter manila resource by inexact match::

    manila list [--name~ <name~>] [--description~ <description~>]

We assuming we have two shares in the name of'test_1', '2_test', usually we
would get none of them if type this command::

    manila list --name test

But within this feature merged, we would have both of them with the identical
command if type this command::

    manila list --name~ test

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

None

Other deployer impact
---------------------

None

Developer impact
----------------

Developer should update the filtering function to support exact/inexact filter
(use 'LIKE' operator to filtering resource) in DB API whenever the list/show
API changed.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  tommylikehu(tommylikehu@gmail.com)
  jun.zhongjun(jun.zhongjun2@gmail.com)

Work Items
----------

* Functional tests
* Implement core feature
* Add related unit tests
* Update python-manilaclient

Dependencies
============

None

Testing
=======

* Add unit tests and functional tests to cover filter process change.

Documentation Impact
====================

Update API documentation.

References
==========

It is proposing essentially the same spec for cinder `[5]`_.

_`[1]`: https://review.openstack.org/#/c/45026/
_`[2]`: https://review.openstack.org/#/c/266688/
_`[3]`: https://en.wikipedia.org/wiki/ReDoS
_`[4]`: https://review.openstack.org/#/c/442982/
_`[5]`: https://review.openstack.org/#/c/441516/
