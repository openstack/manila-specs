=======================================
Support Manila APIs in the OpenStackSDK
=======================================

Blueprint link:

https://tree.taiga.io/project/ashrod98-openstacksdk-manila-support/kanban

Problem Description
===================

Manila is an open source OpenStack Shared File System service that is a software that exists within the OpenStack cloud. It is a collection of microservices and other components that provide self-service management of elastic file system storage infrastructure that can allow you to work with and provision storage via 30+ storage technologies over file system protocols. It is unique in how it is able to add RESTful semantics to consuming shared storage in a reliable and scalable manner.
We aim to improve the common user experience by adding Manila support to the OpenStackSDK, and to reuse OpenStack Identity, Logging, Configuration integration modules without duplicating them into a separate SDK.
This project is part of the resolution defined by the TC Stance on OpenStackSDK and the OpenStackClient.

Use Cases
=========

OpenStack Client will be affected as current manila commands are being implemented using python-manilaclient SDK.
OpenStack's Ansible Collections will be affected as it directly uses of the OpenStack SDK.
Developers that use automation tooling will also benefit from incorporating Manila into OpenStack SDK.

Proposed Changes
================

Resources and methods are listed in order of priority


The following tasks will be a part of the ``Wallaby Cycle``


Priority 1
------------

* Shares
* Share Export Locations
* Share networks
* Share network subnets
* Share types

Priority 2
------------

* Share actions - grant access, revoke access, list access rules, change share size, revert to snapshot
* Security services
* User messages
* Share access rules


The following will be part of the cycle(s) following Wallaby

Priority 3
------------

* Share metadata
* Share snapshots - List share snapshots, Show share snapshot details, Create share snapshot, * Update share snapshot, Delete share snapshot
* Scheduler stats
* Availability zones
* Quota sets
* Quota class sets

Priority 4
------------

* Share replicas
* Share replica export locations
* Share servers
* Services
* Share access rule metadata
* Share groups
* Share group types
* Share group snapshots

Priority 5
------------

* Share Actions - Reset share state, force delete share, unmanage share
* Share Snapshot - manage share snapshot, unmanage share snapshot, reset share snapshot state, force delete share snapshot
* Share Replicas - resync share replica, reset status of share replica, reset state of share replica, force delete share replica
* Share Servers - reset status of share server

Priority 6
------------

* Share snapshot instances
* Share instances
* Share instance export location
* Experimental api/share migration


Alternatives
---------------

Continue using python-manilaclient SDK.
Continue maintenance for python-manilaclient SDK.
Do not implement support for manila in OpenStackSDK.

These alternative are not inline with TC resolution:
https://review.opendev.org/c/openstack/governance/+/759904

Data model impact
--------------------------

No impact on the data model.

Security impact
---------------------

No security impact as authentication is already provided by OpenStackCloud before OpenStackSDK is used.

Notifications impact
---------------------------

No notifications impact.

Other end user impact
------------------------------

Improves user experience by providing user the ability to interact with Manila API through OpenStackSDK.

Performance Impact
----------------------------

No current performance impact. Future impact may include mitigating the use of plug-in clients.

Other deployer impact
-----------------------------

No other deployer impact.

Developer impact
------------------------
Implementing Manila support may make it easier for developers to code scripts using the SDK.
Manila will not stand out within the OpenStackSDK as it is part of a cohesive design.

Implementation
==============

Assignee(s)
----------------

Primary assignee:
* Ashley Rodriguez <ashrod98>
* Nicole Chen  <arkaruki>
* Mark Tony

Other contributors:


Work Items
----------------

* Implement proposed changes
* Write unit tests
* Write documentation
* Write functional tests
* Write end user guide

Dependencies
============

No dependencies at the moment of writing for this project.

Testing
=======

Unit tests will be required for each resource.
Functional tests will also be written.

Documentation Impact
====================

For each resource coded, we will concurrently write its documentation.
We will also write an End User Guide.

References
==========

* End User Guide OSSDK: https://docs.openstack.org/openstacksdk/latest/user/guides
* Manila API Ref: https://docs.openstack.org/api-ref/shared-file-system/
* TC Stance: https://review.opendev.org/c/openstack/governance/+/759904

