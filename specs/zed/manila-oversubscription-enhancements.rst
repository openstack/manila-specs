..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
Manila oversubscription enhancements
====================================

https://blueprints.launchpad.net/manila/+spec/manila-oversubscription-enhancements

About allocated_capacity_gb and provisioned_capacity_gb, allocated_capacity_gb
corresponds to shares_gb created on a storage pool via manila.
provisioned_capacity_gb is equal to the sum of allocated_capacity_gb add
shares_gb_created_manually (or another manila) on the same storage pool. It
implies that provisioned_capacity_gb greater than or equal to
allocated_capacity_gb. It is important for administrators to know these two
values in order to obtain information about back-end allocated capacity.

For a storage backend that supports thin provisioning,The following describes
the reported capacity information.

total_capacity_gb = Total physical disk capacity of the storage pool used by
Manila.

free_capacity_gb = total_capacity_gb - Used physical disk capacity of the
storage pool.

provisioned_capacity_gb = Total capacity allocated by all shares (include
shares created manually or other manila) in the storage pool.

allocated_capacity_gb = Total capacity allocated by all shares created by
current manila.



Problem Description
===================

Problem One:
Here's an example. Storage pool pool_A physical disk capacity is 10G. There are
four shares in this pool.
Share A created by openstack O_A(current manila), share size is 5G. There is a
124MB file in this share.
Share B created by openstack O_A(current manila), share size is 4G. There is a
0MB file in this share.
Share C created by openstack O_B, share size is 3G. There is a 400MB file in
this share.
Share D created manually, share size is 2G. There is a 500MB file in
this share.

The total physical capacity in use is 1G(124MB+0MB+400MB+500MB), the result is:
total_capacity_gb = 10G
free_capacity_gb = 9G
provisioned_capacity_gb = 14G (5G+4G+3G+2G)
allocated_capacity_gb = 9G (5G+4G, only share A and B is created by current manila)

In fact, the storage back end doesn't know how many Manila services are using
the storage pool or whether the administrator has manually created shares in
the pool. So the allocated_capacity_gb reported by backend storage driver is
not correct. Only provisioned_capacity_gb reported by backend storage driver is
correct.

Currently, some drivers (inspur, huawei, netapp, qnap, zadara) report
allocated_capacity_gb and some drivers (infinibox, hpe_3par, nexenta)
report provisioned_capacity_gb, which is chaotic.

Problem Two:
About scheduler service supports Active/Active HA, The consume_from_share
function is executed each time a share is created or extend, in this function
allocated_capacity_gb and free_capacity_gb must be updated. Let's suppose 100
1gb shares are created, 50 of which are executed on node 1 and 30 on node 2
and 20 on node 3. Since allocated_capacity_gb is maintained in the memory of
the scheduling service. As a result, the allocated_capacity_gb values of the
three scheduling nodes are inconsistent. This is problematic. We need to
calibrate allocated_capacity_gb periodically.

Use Cases
=========
* The administrator can view the capacity allocation of the back-end storage
  by allocated_capacity_gb and provisioned_capacity_gb.
* Manila scheduler use provisioned_capacity_gb to filter and weigh back-end.

Proposed Change
===============
This spec proposes to:

* allocated_capacity_gb reported by driver may be correct if the driver is
  capable of tracking it accurately, but allocated_capacity_gb calculated
  from database must be correct. So for uniformity, allocated_capacity_gb
  will use calculated by database rather than reported by driver.
* we strongly recommend that the driver report provisioned_capacity_gb, if
  it is not reported, Manila is considered exclusive storage pool, will use
  allocated_capacity_gb as the provisioned_capacity_gb value.

* for problem two,  Add a periodic task in Manila Share to periodically collect
  allocated_capacity_gb statistics from the database and record it in
  self._allocated_capacity_gb(a member variable of class ShareManager). Read
  self._allocated_capacity_gb and update allocated_capacity_gb in
  share_stats[pools] when executing _report_driver_status. Because Manila
  doesn't require very much real-time data for allocated_capacity_gb,
  Therefore, the interval of periodic tasks can be set to 5 minutes to reduce
  system and database pressure.

Alternatives
------------

The current model is an alternative, however, the calculation occuring in the
scheduler is problematic because it's happening in each scheduler process for
each backend and for each pool, which is inefficient and wrong

Data model impact
-----------------

None

REST API impact
---------------

None

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

we'd be removing a huge performance bottleneck from the scheduler service!

Other deployer impact
---------------------

None

Developer impact
----------------

Drivers should update to report provisioned_capacity_gb instead of
allocated_capacity_gb.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  haixin <haix09@chinatelecom.cn>


Work Items
----------

* Update share manager and scheduler's host manager.
* Update unittest.
* Update related documents.

Dependencies
============

None


Testing
=======

* Add the unit tests

Documentation Impact
====================

The following OpenStack documentations will be updated to reflect this change:

* OpenStack Contributor Guide

References
==========

None
