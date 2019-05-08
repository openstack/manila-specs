..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================
Add new priority attribute for access rules
============================================

https://blueprints.launchpad.net/manila/+spec/add-priority-for-access-rule

Currently allow access rule API only supports access-level, which has "ro"
and "rw" as valid values. This is not enough when users want to specify
priority to individual access rules.


Problem description
===================

The user is not able to reorder their access rules, and different drivers have
different access rule sorts. Looking at the current behavior of share drivers,
we can broadly classify them into three categories for our understanding:

* The "last-rule" drivers add the access rules one by one, the latest access
  rule always has higher priority.
* The "first-rule" drivers always let the prior access rules have higher
  priority.
* The "most specific rule" drivers make sure individual addresses always sort
  higher than the subnets they are part of, and smaller subnets always sort
  above larger subnets.
* Sometimes, after updating the access rules in driver, the access rules order
  won't always be the same as the last time we update the rules.

For the moment Manila API of share access allow does not allow specifying
extra attributes of access like "priority".

It only makes sense for access rules which could possibly overlap. Which would
include anything that uses IP-based access. Perhaps user-based access doesn't
need priorities. Although with groups, one could imagine overlap with
user-based access too. So, if the user rules don't need it, they don't need to
use the priority parameter when the user creates an access rule.

Use Cases
=========

The user wants to set which access have higher priority to write or read the
data of share, which access have lower priority to write or read the data of
share.
Such as: A 192.168.1.10 RW rule would be in effect against a 192.168.1.0/24 RO
rule if the 192.168.1.10 RW one has higher priority. Then if the user wants to
hide the 192.168.1.10 RW rule, they could simply set it to lower priority.

Proposed change
===============

The following are the changes to be made:

* A new parameter 'priority' will be added to access-allow API parameters.
* The 'priority' value supports an integer value in the range 1-200, where a
  lower number means a higher priority. This means the number '1' is the
  highest priority. The maximum value of 'priority' is set to 200.
  We never need more priorities than there are bits in a netmask for ip-based
  rules. Because 2 different rule with the same netmask are either for the
  same address range or nonconflicting, and the prefix length can be all the
  way long to 128 in manila access rule allow API, it could add 129
  different conflicting access rules. Since we don't treat 10.1.0.1/32 and
  10.1.0.1 as conflicts, so perhaps even a few more different conflicting
  access rules. For non-IP-based rules, such as user-based access, the number
  of possible conflicts has to do with the number of groups a user could be
  in, and it seems unlikely that a user would be a member of more than 200
  groups all of which had different levels of accesses to the same share. The
  range is not for different access rules, just for overlapping ones, If we
  leave it unbound people will wonder about negative numbers and numbers maybe
  higher than 4.2 billion, so the validation is important and the number 200
  will result in better usability.
* The access rules will be sorted through the 'priority' value in the share
  manager before they are sent down to the driver. The rules with highest
  priority will be stored in the front of the access rule list that will be
  sent down to the driver.

  * If two of more rules conflict and have the same priority, then the
    behavior is undefined. Where possible, the behavior in this case should
    remain unchanged from previous manila versions, where the behavior in
    this case is also undefined.

* Add the new API to support modifying the access rule 'priority'. Modifying
  rule priority results in the manager invoking update access method in the
  driver.
* The Manila API terminology treats 'deny' the access rules as rules removal
  and not in the classic ACL deny (where some IPs/users can access a resource
  and some are denied from it/just not allowed). So prioritizing 'deny' means
  prioritizing rule removal, thus doesn't mean anything. The deny access rule
  API and command won't be changed.

Alternatives
------------

* Instead of adding a separate parameter, we could overload the "access_to"
  parameter to allow specifying the priority by using '#' as a separator. We
  will parse the value of access_to in driver. such as: "manila access-allow
  test_share_id user#priority=1".

* Prioritization only matters to rules whose clients can overlap. The most
  important use case of this is with NFS rules to client IP addresses. So,
  instead of allowing user-defined priorities, we could adopt a behavior in
  manila where access rules are always sorted in the share manager in the
  following order: Rules for Individual IP addresses sort higher than those
  for subnets containing them, and smaller subnets sort higher than larger
  subnets containing these smaller subnets. For example: If the user allows
  access in the following way:

  1. allow 'ro' access to 192.168.17.16
  2. allow 'rw' access to 192.168.17.0/22
  3. allow 'ro' access to 192.168.17.0/24
  4. allow 'rw' access to 192.160.16.15

  The priority that manila ensures would be as follows::

      +----------+-----------------+--------------+
      | Priority |    Access To    | Access Level |
      +----------+-----------------+--------------+
      |        1 | 192.168.17.16   | ro           |
      |        2 | 192.160.16.15   | rw           |
      |        3 | 192.168.17.0/24 | rw           |
      |        4 | 192.168.17.0/22 | ro           |
      +----------+-----------------+--------------+

  * In this way, if we want to disabled the Individual IP addresses rules or
    smaller subnets access rules, we have to delete those access rules. For
    example, if I want to force all my shares to read only for a short period
    while I fix something, but I don't want to have to delete all my rules out
    of manila to achieve that.
  * This way will also make upgrades a bit harder for backwards compatibility.
    Because the access rules will have the different order after upgrades to
    the new version.

Data model impact
-----------------

The ``manila.db.sqlalchemy.models.ShareInstanceAccessMapping`` model will
have a new field called ``priority``.

The priority of all pre-existing rules will be set to `100` after upgrading
manila.

REST API impact
---------------
The new parameter will be added in access API. We will bump the micro-version
to expose the 'priority' parameter:

**Adding an access rule**

    POST /v2/{tenant-id}/shares/{share-id}/action
    BODY::

        {
            'allow_access': {
                    "access_level": "rw",
                    "access_type": "ip",
                    "access_to": "0.0.0.0/0",
                    "priority": "1"
            }
        }

    The "priority" is an optional parameter. If the user doesn't input it,
    it will be set to `1`.

**Updating access rules**

    PATCH /v2/{tenant-id}/share-access-rules/{access-id}
    BODY::

        {
            "priority": "1"
        }

**Listing access rules**

    GET   /v2/{project_id}/share-access-rules?share_id={share-id}&sort_dir=desc&sort_key=priority
    Response::

        {
            "accesses": [
                {
                        "access_level": "rw",
                        "state": "active",
                        "id": "507bf114-36f2-4f56-8cf4-857985ca87c1",
                        "access_type": "cert",
                        "access_to": "example.com",
                        "access_key": null,
                        "priority": "1",
                },
                {
                        "access_level": "rw",
                        "state": "error",
                        "id": "329bf795-2cd5-69s2-cs8d-857985ca3652",
                        "access_type": "ip",
                        "access_to": "10.0.0.2",
                        "access_key": null,
                        "priority": "3",
                },
            ]
        }

    Adding the "priority" field in a micro-version change to this new API.
    The "share_id" is a mandatory query key, and the API will respond with
    HTTP 400 if the "share_id" is not provided.
    Adding "sort_dir" and "sort_key" filter in list API. The "sort_dir" means
    sort direction, and the value of "sort_dir" should be 'desc' or 'asc'.


.. note::

    * The current `access rules list API
      <https://developer.openstack.org/api-ref/shared-file-system/#list-access-rules>`_
      accepts HTTP POST requests. To ensure correct HTTP semantics around
      idempotent and safe information retrieval, we will introduce a new API
      that accepts GET requests. The old API will be capped with a maximum
      micro-version, i.e, it will not be available from the micro-version that
      this new API is introduced within.


Security impact
---------------

None

Notifications impact
--------------------

Add the "priority" field in user error notifcations when we create an access
rule or update the access rule.

Other end user impact
---------------------

The Manila client, CLI will be extended to support access rule priority.

* The access-allow command with access priority supported will be like::

    manila access-allow [--priority <priority>]
                        [--access-level <access_level>]
                        <share> <access_type> <access_to>

    Optional arguments:

    --priority  The 'priority' value supports an integer value in the
                range 1-200, where a lower number means a higher priority.
                The default value is set to '100'.

* The new access-update command with priority supported will be like::

    manila access-update <access> [--priority <priority>]

    Optional arguments:

    --priority  The 'priority' value supports an integer value in the
                range 1-200, where a lower number means a higher priority.
                OPTIONAL: Default=None.

* The access-list command with access priority sorting supported will be like::


    manila access-list [--columns <columns>] [--sort-key <sort_key>]
                       [--sort-dir <sort_dir>]
                       <share>

    Optional arguments:
    --sort-dir <sort_dir>, --sort_dir <sort_dir>
                        Sort direction, available values are ('asc', 'desc').
                        OPTIONAL: Default=None.
    --sort-key <sort_key>, --sort_key <sort_key>
                        Key to be sorted, available keys are (priority).
                        OPTIONAL: Default=None.

* We will also perform client side validation for value of "priority" limit
  range from 1 to 200.


Performance impact
------------------

Sorting access rules have a negative service performance impact.

Other deployer impact
---------------------

None

Developer impact
----------------

None

Driver impact
-------------

The access rules will be ordered in the share manager before being sent down
to drivers in update_access function, so the drivers need not see the
priority field and need not change their current behavior.
If some drivers are already reordering the rules, we will audit drivers that
are reordering rules and report bugs right away. They have to be updated to
comply with the order determined by the share manager.
If your back end can't correctly implement a broad rule overriding a more
narrow rule when the broad rule is earlier in the list, then the driver must
drop the more narrow rule and not send it to the backend at all.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  | zhongjun


Work Items
----------

* Add priority property to access rule object and bump the API microversion.
* Add a new parameter in "access_rules" table and add db upgrade script.
* Add a new update-access API.
* Add new unit and tempest tests for access rule priority.
* Update python-manilaclient and the UI, Allow the end user to move rules
  around in a list and figure out the priorities, like other public clouds
  do `[1]`_.

Dependencies
============

None


Testing
=======

* Add the unit tests
* Add the tempest tests

Documentation Impact
====================

The following OpenStack documentation will be updated to reflect this change:

* OpenStack User Guide
* OpenStack API Reference
* Manila Developer Reference
* The Admin Guide

References
==========

* Support for access rule priority in Alibaba Cloud:

  * _`[1]` https://www.alibabacloud.com/help/doc-detail/27534.htm?spm=a3c0i.o27518en.b99.23.371c253bJ2y4HY

* Support for access rule priority in Tencent Cloud:

  https://intl.cloud.tencent.com/document/product/582/13778
