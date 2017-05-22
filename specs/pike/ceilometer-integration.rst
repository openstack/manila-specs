..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
Integration with ceilometer
===========================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/manila/+spec/ceilometer-integration

At the moment of writing it is not possible to send usage notifications from
manila to ceilometer. Having this possiblity, users would be able to
monitor the status of the share service and also the shares themselves.

Problem description
===================

Currently there is no way to receive notifications about
events in the life-cycle of share service resources,
precisely, information such as:

* Share size
* Share capacity usage
* Share create (start/end)
* Share delete (start/end)
* Share update (start/end)
* Share extend (start/end)
* Share type create (start/end)
* Share type delete (start/end)
* Snapshot size
* Snapshot create (start/end)
* Snapshot delete (start/end)
* Revert to snapshot (start/end)

Size metrics are discrete values (gauge meters)
while create/delete/update operations are
changing over time values (delta meters).

And also it's not possible to know performance information about
the available shares, that is, information like:

* ops/s (number of operations per second issued to the filesystem)
* rops/s (number of read operations per second issued to the filesystem)
* wops/s (number of write operations per second issued to the filesystem)

The latter has been requested by users in the past `[1]`_.

For now, the community has shown interest on implementing
the required logic to retrieve metrics for the resources in manila `[2]`_.
And hence, this spec will focus on this.

Depending on the project plans in the future, we will decide if it is
on manila's scope to put hands on metrics for the different shares.

Use cases
=========

* Monitoring usage of resources would allow us to keep a better control
  on the health of the deployments and see when capacity limits are being
  approached.

* Operators would be able to act over the collected data
  and trigger actions when defined criteria are met.

Proposed change
===============

The proposed change consists of the following

* Add notifiers for all the resources we want to measure.
  Right now we have notifiers in share_types.py `[3]`_ but this is
  a remainder of refactoring from cinder's volumes_types.py in previous versions.
  We would need to add notifiers to shares and snapshots,
  and check that the notifiers in share-types publish the data we expect.

* Add meters in ceilometer for the desired resources.
  For this, we would need to follow ceilometer's guidelines `[4]`_.
  The resources we are considering in this spec are shares, share types
  and snapshots. Meters for other resources such as share groups will be
  added in follow up changes.

Alternatives
------------

Do not integrate with ceilometer and don't provide telemetry for manila.

Data model impact
-----------------

This change has no impact over the data model.

REST API impact
---------------

This change has no impact over the REST API.

Driver impact
-------------

This change has no impact over any driver.

Security impact
---------------

Notifications impact
--------------------

This change will allow sending notifications about the consumed resources.
This is something that operators should be able to enable or disable.
By default, it should be disabled.

Other end user impact
---------------------

End users will be able to see the notifications in ceilometer if enabled.

Performance impact
------------------

We need to consider in here the overhead produced by emiting notifications
to the messaging queue for every resource we will be measuring and for all
the actions we will be covering. Whereas it only retrieves data
that is being accessed already (i.e. we don't need to go to the control
database in order to retrieve more data),
is an action that will be performed in every interaction with the resource.

Other deployer impact
---------------------

We will need to add an option for enabling and disabling sending notifications.
This flag will affect manila deployed with any backend.
By default, this option will be set to false and manila won't be sending
notifications, keeping the behavior it currently has.

If operators want to start retrieving metrics on the manila resources,
they will need to enable sending notifications on the configuration file.
They also need to make sure they have a specified
version of ceilometer deployed. After this, the feature here
specified will be available for use.

Developer impact
----------------

After the framework is added, developers may need to add notifications
accordingly when adding new functionalities.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
    Victoria Martinez de la Cruz <victoria@redhat.com>

Work Items
----------

* Add notifiers for shares. Add unit tests.
  Add meters for shares. Add dev docs for shares meters.
* Add or update notifiers for share-types. Add unit tests.
  Add meters for share-types. Add dev docs for share-types meters.
* Add notifiers for snapshots. Add unit tests.
  Add meters for snapshots. Add dev docs for snapshot meters.

Dependencies
============

The blueprint on the ceilometer side can be accessed in `[5]`_.
We will add the desired meters as part of that blueprint.

No other dependencies considered at the moment of writing.

Testing
=======

Whereas having tempest coverage for this feature is desired,
it's not a priority.

Unit tests will be added for main functionality.

A CI job with manila and ceilometer enabled will be added
to exercise integration and compatibility of both services.

Documentation Impact
====================

With regard to documentation, we will need to add explicit docs
on how to configure manila to enable this feature and we will need to add
explicit instructions on which kind of data will be available.
Whereas this is something that we will cover on the developers docs,
it will be important to add some subsection under manila
on the operations manuals.


References
==========

* _`[1]` https://ask.openstack.org/en/question/58203/how-to-collect-telemetryceilometer-metrics-from-manila-file-share-service/

* _`[2]` https://etherpad.openstack.org/p/manila-pike-ptg-thursday

* _`[3]` https://github.com/openstack/manila/commit/0cb695fd54f90a94fe185ff7e34ba0b175b6c75b#diff-1117d59ee7142c5324a3d327d39b3a0fR45

* _`[4]` https://docs.openstack.org/developer/ceilometer/new_meters.html#add-new-meters

* _`[5]` https://blueprints.launchpad.net/ceilometer/+spec/manila-meters
