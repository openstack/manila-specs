..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================
The share size can be limited by share type
===========================================

https://blueprints.launchpad.net/manila/+spec/share-size-limited-by-share-type

Add support for limit the size of share through the share type, the share
created by the user shall not be greater than the maximum value set in the
share type and shall not be less than the minimum value set. Of course,
depending on the usage scenario, only a maximum or a minimum can be set.

Problem description
===================

In some usage scenarios, we do not want users to create too large share size,
then affect the number of shares that can be created, maybe limited by project
quota or backend storage, or in some usage scenarios, creating a share that
is too small makes no sense. So administrators desperately need a way to limit
the size of share created by users.

Use Cases
=========

In scenarios where storage backend capacity is small, or project quota is
limited and the number of users is high, In order to enable more users to
create their own share, Administrators need to limit the maximum size of each
share. In other scenarios, if the share is too small to ensure the normal
operation of the business, the administrator needs to set the minimum value
of each share. Administrator can also set the maximum and minimum values of
share according to actual needs.

Proposed change
===============

The following changes are proposed:

* Two new extra_specs keys of share type are added for support set the minimum
  share size and maximum share size.

  * **provisioning:min_share_size** Set the minimum size of share by the
    share type.
  * **provisioning:max_share_size** Set the maximum size of share by the
    share type.

* The value of these keys is a positive integer, the unit defaults to GB.
* These two extra specifications will be visible to end users. If they're not
  set, it is assumed that there are no share size limits enforced by the share
  type, but limits may be enforced by the service and the user's quotas.

Alternatives
------------

None

Data model impact
-----------------

None

REST API impact
---------------

The size restrictions will be checked at the API level as part of share
creation, extend, shrink or migration.

Security impact
---------------

None

Notifications impact
--------------------

None.

Other end user impact
---------------------

The size of the share created by the end user will be limited by the share
type if set specified keys in extra_specs in share type. Operations including
extend, shrink, and migration operations are also subject to this restriction.

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
  haixin<haixin@inspur.com>


Work Items
----------

* Add size check in API level
* Add related unit and functional tests
* Add docs update


Dependencies
============

None


Testing
=======

1. Unit test to test whether these keys restrictions are in effect.
2. Tempest test whether set keys work correctly from API perspective.

Documentation Impact
====================

1. The manila API documentation will need to be updated to reflect extra_specs
   of share type support two more keys.

References
==========

None
