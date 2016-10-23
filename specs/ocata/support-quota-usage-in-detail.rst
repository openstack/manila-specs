..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Add detail API to quota-set API collection
==========================================

https://blueprints.launchpad.net/manila/+spec/admin-check-tenant-quota-usage

Currently we can only get the quota-set information with 'limit' attribute
returned both for user and tenant in Manila, but this is too limited for
user's various cases. In other OpenStack components such as Nova, it both
has the ability to check the 'quota-show'(only 'limit') and 'quota-show
--detail'('in_use', 'limit' and 'reserved'), Cinder also has the same ability
with command 'quota-usage'. This change intends to add the same ability for
Manila just like Nova.

Problem description
===================

It's a common use case to check the quota-set information, and Manila has
the API 'quota-sets' to cover this, but it does not satisfy the situation
when overall 'quota-sets' information is desired. In effect, Manila already
supported this inside the quota-set's code [1] at present, but this hasn't
been exposed to user. This change intends to support this.

Use Cases
=========

As described above, user often wants to check the quota-set's overall
information (both tenant and user of this tenant), this change will help
them achieve this.

Proposed change
===============

The following are the changes to be made:

* A new API '.../quota-sets/{tenant_id}/detail' will be added to quota-sets
  API collection.
* The new API will share same policy with the existing GET 'quota-sets' API.
* The subsequent logic will be identical with existed API 'GET quota-sets'
  except that the parameter 'usage' in 'QuotaSetsController._get_quotas' is
  set True.
* The '_get_quotas' will return a dictionary with 'in_use', 'limit',
  'reserved' attributes contained.
* The Manila client will add a new command argument '--detail' to 'quota-show'
  to indicate whether to return the quota-sets information in detail.

Alternatives
------------

Currently Manila does not support this.

Data model impact
-----------------

None

REST API impact
---------------

The API microversion will have to be bumped up for the new APIs below.

1. (GET 200 403) quota-sets detail: show quota-sets in detail

URL: /v2/{tenant_id}/quota-sets/{tenant_id}/detail?user_id={u_id_query}

Response Body::

    {
        'quota_set':{
            'id': '1e1cc7d3-c524-4b05-85b4-5d548916b11b',
            'shares': {'in_use': 0, 'limit': 23, 'reserved': 0},
            'gigabytes': {'in_use': 0, 'limit': 45, 'reserved': 0},
            'snapshots': {'in_use': 0, 'limit': 34, 'reserved': 0},
            'snapshot_gigabytes': {'in_use': 0,'limit': 56,'reserved': 0},
            'share_networks': {'in_use': 0,'limit': 67,'reserved': 0}
        }
    }

Manila client impact
--------------------

Manila client will add a new command argument '--detail' to command
'quota-show', the modified version of command will be like::

* quota-show --detail <other existing command arguments>

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

* Add 'detail' API to 'quota-sets' collection.
* Add related tests in Manila.
* Add documentation in Manila.
* Add new argument for command 'quota-show' in Manila client.
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

[1]  https://github.com/openstack/manila/blob/master/manila/quota.py
