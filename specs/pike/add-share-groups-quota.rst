..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================
Add share_groups quota
======================

https://blueprints.launchpad.net/manila/+spec/add-share-groups-quota

Manila is able to create several different kinds of resources, which use
real back-end resources. Since real resources are always finite, it is
practical to have different kinds of limits/quotas for them.
And one of not covered resources by limits is amount of share groups that
user is able to create. Share Groups were added in Ocata timeframe and
were, originally, consistency-groups in older than Ocata releases.

Problem description
===================

The problem is in possibility to schedule creation of unlimited amount of
share groups, that can lead to 2 bad things:

* garbaging of DB with any amount of DB table records;
* waste of real back-end resources.

So, implementation of quotas for share_group resource will solve above
mentioned problems.

Use Cases
=========

Control of used manila resources will now be more flexible, because admin
will be able to set additional quotas for previously untracked manila resource.

Proposed change
===============

Implement new quota resource called "share_groups" in existing quota mechanism.
This quota will track amount of created share groups resources and restrict
creation of them after limits are exceeded.
Make default value be equal to amount of shares.

Alternatives
------------

Do not implement quotas for share groups. In this case any user will be able
to create unlimited amount of share groups.

Data model impact
-----------------

Existing DB schema does not require any changes. Hence, no new DB migrations
are required. We will reuse existing logic, where we can track any amount of
resources by their names.

REST API impact
---------------

Only one (1) API will be changed, it is "quota-update". It will be able to
handle one more key provided in POST data, called "share_groups".
Also, "quota-show" command will return dict with quota resources [and usages]
for this key too. Both these changes require bump of microversion and will be
available only with this or later one microverisons.
If for moment of this spec implementation share-groups feature
stays 'experimental', then these API changes should be available only if
'X-OpenStack-Manila-API-Experimental' header provided and set to 'true'
boolean value.

* Update quotas

  * URL: /quota-sets
  * Method: PUT
  * JSON body:

    .. code:: json

        {
          'quotas': {
            'share_groups': 100,
          }
        }

Driver impact
-------------

None

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

Users will now be restricted in amount of share groups compared to previous
behavior.

Manila client will have additional following optional key to "quota-update"
command::

    $ manila quota-update <tenant_id> --share-groups %amount_as_integer_value%

In general, quota should be able to be set to any value and in case usage
exceeds current quota, request for creation of new resource will be refused.

Performance Impact
------------------

Resource usages will be checked each time we create/delete resources, but
it is relatively cheap operation.

Other deployer impact
---------------------

Deployers will be able to set quotas for share groups using either
configuration file or "quota-update" API. It is optional possibility,
not a mandatory action.
New option will be named in the same way as all
existing ones - "quota_share_groups". It will accept only integer values.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:

* vponomaryov ( vponomaryov@mirantis.com )

Work Items
----------

* Server side API changes, including unit and functional tests
* Manila Client side changes, including unit and functional tests
* Manila UI side changes, including unit tests

Dependencies
============

None

Testing
=======

* unit tests in 'manila', 'python-manilaclient' and 'manila-ui' projects
* functional tests in 'manila' and 'python-manilaclient' projects

Documentation Impact
====================

* admin guide: add new '--share-groups' parameter to 'update-quota' command.
* devref: reflect the proposed change

References
==========

* Share Groups feature spec:
  https://github.com/openstack/manila-specs/blob/master/specs/ocata/manila-share-groups.rst
