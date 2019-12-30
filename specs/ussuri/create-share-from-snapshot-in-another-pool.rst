..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================================
Create share from snapshots in another pool or back end
=======================================================

https://blueprints.launchpad.net/manila/+spec/create-share-from-snapshot-in-another-pool-or-backend

One of the current limitations present in most Manila drivers is the inability
to create new shares from snapshots in pools other than the source share's.

Given that some storage back ends have the ability to fast clone a share from
a snapshot to another pool or back end, having this feature working in Manila
will greatly improve the user experience.

This spec proposes changes to the current design of scheduling new shares
created from snapshots. We will introduce a new scheduler capability and
improve the behavior of an existing API parameter. By allowing the user to
choose the desired availability zone and the admin to optimize its placement,
we can improve our scheduler to be smarter for better load balancing and
space efficiency results, while also preventing erroneous behavior.

One of the result of the improvements proposed is that drivers will be able to
rely on a proper scheduling mechanism that makes sure new shares created from
snapshots can land on compatible destinations.


Problem description
===================

Manila has the scheduler's configuration option
``use_scheduler_creating_share_from_snapshot`` that controls whether the
scheduler should consider all available pools or just the source one as
candidates to schedule a new share created from a snapshot. This option is
disabled by default and is crucial to be set this way for drivers that
cannot create shares anywhere other than the source pool. When enabled, this
option allows different results:

* The new share from snapshot may land on the same pool. This works for all
  drivers.
* The new share from snapshot may land on another pool in the same back end.
  This works for the generic driver and may work on others (we do not know
  which). In case it does not work, it will result in an error in the driver.
* The new share from snapshot may land on another compatible back end. This can
  work for the generic driver and is unlikely to work on others (we do not know
  which). In case it does not work, it will result in an error in the
  destination pool's driver.
* The new share from snapshot may land on another incompatible back end.
  This will certainly not work for any of the drivers in manila.

The main benefit from enabling this option is that the scheduler can select
a different compatible back end to place the new share, even when the source
share's back end has space enough to handle it. In order to have it working
properly, we need to address the following:

* Guarantee that only compatible back ends will receive requests;
* Guarantee that, when more than one compatible back end is available, the
  scheduler will choose the best fit;
* No regressions are inserted to the current behavior when the option is
  disabled.

Use Cases
=========

Given that there are storage back ends that support (or may want to support)
new shares from snapshots to be efficiently created in pools or back ends
different than the source share's, a cloud administrator may want to enable
this functionality, in order to balance the space usage across the several
configured different compatible back ends in the environment.

Furthermore, if a new share from a snapshot may be placed on a back end other
than the source share's, it may as well be placed in a different availability
zone. Since AZs are user-facing and are already included as a parameter in the
``Create share`` API, users could greatly benefit from this to create
independent copies of their shares in AZs that are closer to their application.

From the user's perspective there are two different ways of requesting new
shares from snapshot:

1. **User doesn't provide the destination AZ.** The user doesn't specify an AZ
   to place the new share from snapshot. The scheduler won't filter out the
   back ends based on a specific AZ, will filter out only the back ends that
   are not compatible with source share's pool. The list of compatible back
   ends will be weighed based on configured weigher classes.

2. **User provides the destination AZ.** The scheduler will filter out all back
   ends that do not belong to the user's specified AZ and all back ends that
   are not compatible with source share's pool. The list of compatible back
   ends will be weighed based on configured weigher classes.

Proposed Changes
================

The proposed changes will only affect the existing behavior if the option
``use_scheduler_creating_share_from_snapshot`` is enabled. Otherwise, the API
will skip the scheduler's processing, with the new share being created into the
same source share's pool. Even so, if the user specifies a destination AZ to
place the share from snapshot with the scheduler's processing disabled, the API
will return an error to the user about the erroneous usage of the parameter.

This spec proposes changes in order to have this functionality working
as expected:

1. Change how the API handles the availability zone parameter, which is
currently ignored when creating new shares from snapshots;

2. Create a new scheduler's filter ``CreateFromSnapshotFilter``, to identify
compatible back ends;

3. Create a new weigher, to improve the scheduler's selection of the best
destination pool;

4. Update the Share Manager's workflow when requesting a share creation from a
snapshot. The driver interface for ``create_share_from_snapshot`` will be
updated to return a model update when an asynchronous operation is needed.

5. Create a new periodic task to track and update the status of shares that
were created from snapshots.

6. Provide a proper configuration and guidelines for the feature.

The required changes are detailed in the following subsections.

API's Semantic
--------------
Currently, the AZ parameter is ignored in case a new share is created from a
snapshot (potentially a bug, as it should return an error stating that a
different AZ should not be specified). With the option
``use_scheduler_creating_share_from_snapshot`` enabled, the API will need to
forward the AZ parameter into the request sent to the scheduler, otherwise,
the request will fail and the user will be notified about the reasons of the
failure.


Scheduler's Improvement
-----------------------

The scheduler should be able to distinguish drivers that support creating new
shares from snapshots in pools or back ends different than the source share's.
This ability will be addressed by existing filters and the use of an already
implemented property:``replication_domain``.

The replication feature is tightly related to the ability of creating a share
in another pool. If two or more back ends are coupled for replication, they
commonly have a fast data link between each other, that can be used by some
storage vendors to move, copy and replicate data among the back ends in the
replication domain.

This spec proposes to create a new filter, ``CreateFromSnapshotFilter``,
to use the property ``replication_domain`` to identify compatible storage
back ends, i.e., remote pools and back ends capable of creating shares from
snapshots in an efficient way.

In this proposal it is considered that back ends that can replicate data
between each other should also be able to create new shares in an efficient
way, instead of depending on generic data copy methods. Although, drivers still
need to implement mechanisms to perform this kind of operation efficiently
based on back end capabilities.

At this point, this proposal would have solved the filtering problem, but not
efficiently or with good performance, considering that some back ends that
support this functionality have to create the new share from snapshot locally
and move it over to the destination. Moreover, consider that some back ends are
able to create these new shares from snapshots as COW (Copy-On-Write) clones,
using no significant additional space.

Therefore, we could enhance the aforementioned proposal by creating a new
weigher to assist on new share's placement when creating from a snapshot.
A new weigher called ``HostAffinityWeigher`` will always prefer to create the
new share, from a snapshot, closer to the source's pool, whenever it has space
available and the AZ is the same as the source's pool. The weigher will
classify the compatible back ends based on their relative location with the
source share's location. If the source and destination hosts are located on:

    1. same back ends and pools, the destination host is a perfect choice
       (e.g. 100 points)
    2. same back ends and different pools, the destination host is a very good
       choice (e.g. 75 points)
    3. different back ends with the same AZ: the destination host is a good
       choice (e.g. 50 points)
    4. different back ends and AZ's: the destination host isn't a good choice
       (e.g. 25 points)

Even so, this strategy still doesn't solve the used space balance use case at
all, i.e., if no AZ is specified, the source pool will still be used until it
runs out of space. This limitation won't be handled by the new weigher itself,
but it can be mitigated by the cloud administrator, who can specify filters or
goodness functions to fulfill her/his own needs. An example of a possible
configuration is presented below:

* If the administrator wants to use the source pool until the free capacity
  goes under 500 GiB, he/she can provide a filter function to avoid the
  selection of this pool:

  * filter_function =
    "share.snapshot_id and capabilities.free_capacity_gb < 500"

If COW clones are available at the source share's pool, it’s very unlikely that
the administrator will want to place the new share in another pool when
working within the same AZ. But at the end, it is up to him/her to decide when
a host should be filtered when creating shares from snapshot.

We can finally combine these approaches presented here and describe our
proposed scheduling algorithm improvement:

1. The scheduler request will be updated to include source share's host
   location needed by the new filter and weigher functions. It will be
   submitted through a list of scheduler filters, including the
   ``AvailabilityZoneFilter`` which will exclude back ends that don't satisfy
   an availability zone restriction, if specified by the user during the share
   creation.

2. Alongside the scope of back ends determined by regular filters, we submit
   them through the ``CreateFromSnapshotFilter`` in order to filter
   incompatible back ends that don't fulfill at least one of the restrictions:

   a. Pools located in the same parent's share back end.
   b. Back ends and respective pools that match the same parent's share string
      set in ``replication_domain``.

3. After that, we submit them through the regular weighers. If there are no
   valid weighed back ends, we error out with the usual ``No valid host found``
   message.

4. In case there is more than one candidate, the new ``HostAffinityWeigher``
   will be used to weigh hosts based on their proximity to the source's pool.

In this proposal, we assume that drivers that implement replication features
between compatible back ends shall also be capable of creating new shares from
snapshots in another pool or back end different from the source's share.

Share Manager's Updates
-----------------------

When creating new shares from snapshots, the driver interface
``create_share_from_snapshot`` is called and a string or a list of export
locations is expected as return in a success scenario. However, by enabling the
creation of shares from snapshot in another pool or back end can potentially
increase the time spent by the driver to complete the requested operation,
considering that it may lead to a data copy between different pools or
back ends. To avoid performance impacts, the driver interface will be updated to
work in an asynchronous way, by returning a model update, containing the status
of the new share and a list of export locations, if available at share creation
time. The driver can return one of the following status to indicate the current
status of the share creation from snapshot:

1. **STATUS_AVAILABLE**: the share was already created on the destination pool
and an export location should be available.

2. **STATUS_CREATING_FROM_SNAPSHOT**: the share is in ``creating`` status and
can take some time to be available to the user. The share manager will
periodically check the status of this share through a new periodic task.

For backward compatibility, the vendors can continue to suppress the share
status from the returned dictionary, that the share will be considered as
available to user.

A new intermediate status, ``STATUS_CREATING_FROM_SNAPSHOT``, will be added to
assist share manager on filtering shares that are still working on copying data
from a source snapshot. The new status will also be available to end users that
request shares information through the API.

The share manager will have a new periodic task responsible for checking,
using driver's interfaces, if shares that are in
``STATUS_CREATING_FROM_SNAPSHOT`` state have an update in their status. The
manager will apply changes to the database and notify the user about them.

Alternatives
------------

There are no known alternatives that could fully solve the before-mentioned use
cases. However, the alternatives below can address some of the current problems
and partially solve the use cases:

* **Remove the config option ``use_scheduler_creating_share_from_snapshot`` and
  thus make the functionality restrictive and consistent across all drivers:**
  This would also remove an existing functionality (even if it does not work as
  intended at all times). The API would be changed to return an error if the
  AZ parameter is specified when creating new shares from snapshots. This will
  not address the case of spreading out the creation of new shares from
  snapshots.

* **Use the Data Service for a generic implementation**: This could partially
  address the use case, as it would allow drivers that are compatible with the
  Data Service to be seamlessly compatible with each other. However, any
  driver-specific optimizations that could allow the operation to be performed
  more efficiently between compatible drivers would not be able to be used.
  This is an approach that could be used as a fallback from the main proposal,
  in case the AZ parameter is specified but there are no drivers at the
  destination AZ that are compatible with the source share's driver.


Data model impact
-----------------

A new ``progress`` field will be added to
``manila.db.sqlalchemy.models.ShareInstance`` indicating the progress of a
share creation.

REST API impact
---------------

Currently the AZ parameter is erroneously ignored when creating new shares from
snapshots. The proposal changes the API behavior to use the AZ parameter to
determine the AZ to schedule the new share from snapshot to, therefore the AZ
parameter will not be ignored anymore.
In the same way, if the requested AZ isn't the same as the source share's AZ
when the configuration option
``[DEFAULT]/use_scheduler_creating_share_from_snapshot`` is set to False, the
API will return a HTTP 400, Bad Request.

Driver impact
-------------

There is no impact for drivers that do not support or want to support the
proposed feature. If the cloud administrator enables the option
``use_scheduler_creating_share_from_snapshot``, the scheduler filter will
guarantee that unsupported back ends will not receive these kind of requests.
Vendors that want to support this feature will need to change their drivers
implementation to properly handle this use case. In order to have it working,
some prerequisites are needed:

1. Use the configuration option ``replication_domain`` and supply it to the
scheduler together with the storage pool stats.

2. The driver implementation for ``create_share_from_snapshot`` will need to be
modified to accept a destination pool different from the source share's pool,
and be prepared to return asynchronously if a slow copy operation is needed to
complete the share creation.

3. Implement a new driver interface, ``get_share_status``, that will be used by
the share manager to periodically check the status of shares created from
snapshots. The driver will be able to provide the current status, the creation
progress information in a percentage value, and the export locations for
shares that become available. If omitted, the progress information will be set
to **0%** for shares in ``STATUS_CREATING_FROM_SNAPSHOT`` and **100%** for
shares in ``STATUS_AVAILABLE``.

Security impact
---------------

None.

Notifications impact
--------------------

New scheduler code introduced will include error notifications for when a new
share from snapshot fails to be scheduled. The message and code for those
notifications will continue to be ``No valid host found``.
New manager code will include share status notifications when an asynchronous
creation mode is used to create a new share from snapshot. The user will be
notified that the share will take more time to be created and be notified when
the share status is updated by the share manager.

Other end user impact
---------------------

No changes to python-manilaclient are necessary. End users will be able to
create new shares from snapshots in AZs other than the source share's using
the current python-manilaclient.

Performance Impact
------------------

The performance impact should be minimal. The share manager will need to check
in regular intervals whether the driver has finished the share creation for
shares that remain with status equal to ``STATUS_CREATING_FROM_SNAPSHOT``.

Other deployer impact
---------------------

None.

Developer impact
----------------

None.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  dviroel


Work Items
----------

* Implement main patch for manila that includes:

    * Share API adjustments to pass AZ parameter;
    * ShareInstance's new ``progress`` field will be included to provide the
      total progress of a share creation.
    * Scheduler's new weigher will be added to rate host based on their
      proximity to the source's share;
    * Scheduler's new filter ``CreateFromSnapshotFilter`` will be added to
      filter out incompatible back ends, only when the source share's
      ``snapshot_id`` is provided.
    * Share manager will introduce new periodic checks for asynchronous
      operations, update share's status in the database and notify users about
      share's status changes;
    * New driver interface to check for updates on shares that were created
      from snapshots in an asynchronous mode.

* Testing:
    * Implement functionality in ZFSonLinux driver to validate correct
      scheduling and share creation from snapshot on different pools.

* Functional tests in manila-tempest-plugin.

* Docs update.


Dependencies
============

None.

Testing
=======

New functional tests will be added to create a new share from a snapshot in a
given AZ different than the existing one. Negative tests will check if user's
requested AZ is available and if the operation is compatible with the
configured option ``use_scheduler_creating_share_from_snapshot``.

The new tests will run at the gate for the ZFSonLinux driver configured with
at least two AZs. Vendors that implement support for the new capabilities in
their drivers will be encouraged to run the tests in their third party CI.


Documentation Impact
====================

The following documentation sections will be updated:

* API reference: Will update the Create Share API information, adding some
  detail on the impact of the ``availability_zone`` parameter when creating
  new shares from snapshots.

* Admin reference: Will add detailed information on how the Create Share API
  behaves according to the AZ parameter when creating new shares from
  snapshots.

* Developer reference: Will add information on how the functionality works, the
  optimizations and new capabilities.


References
==========

[1] https://etherpad.openstack.org/p/manila-ptg-planning-denver-2018
[2] https://etherpad.openstack.org/p/manila-ptg-train
[3] https://etherpad.openstack.org/p/shanghai-ptg-manila-virtual
