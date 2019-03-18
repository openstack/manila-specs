..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================
OSC support for Manila
======================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/python-manilaclient/+spec/openstack-client-support

Problem description
===================

Python-Openstackclient is the default command line client for many
OpenStack projects.

Use Cases
=========

An end user can interact with manila using the same client they use for
other services in OpenStack through the python-openstackclient.

Proposed change
===============

The intent of this spec is to identify the commands to be implemented
and establish conventions for command and argument names.
This spec is not intended to be a full and correct specification of
command and argument names. The details can be left to the code reviews
for the commands themselves.

The following conventions will be adopted for argument flags:

- We are not planning to implement any different / new arguments
  to existing ones in python-manilaclient.

The following ``manila`` commands will be implemented for ``openstack``


Limits
------

::

    manila absolute-limits
    openstack limits show --absolute

    manila rate-limits
    openstack limits show --rate


Shares
------

::

    manila create
    openstack share create

    manila delete
    openstack share delete

    manila force-delete
    openstack share delete --force

    manila list
    openstack share list

    manila show
    openstack share show

    manila update
    openstack share set
    openstack share unset


Share export locations
----------------------

::

    manila share-export-location-list
    openstack share export location list

    manila share export-location-show
    openstack share export location show

Share metadata
--------------

::

    manila metadata
    openstack share set --property
    openstack share unset --property

    manila metadata-show
    openstack share metadata show


Share actions
-------------

::

    manila reset-state
    openstack share set --status

    manila reset-task-state
    openstack share set --task-state

    manila extend
    manila shrink
    openstack share resize

    manila revert-to-snapshot
    openstack share revert

Share snapshots
---------------

::

    manila snapshot-access-allow
    openstack share snapshot access create

    manila snapshot-access-deny
    openstack share snapshot access delete

    manila snapshot-access-list
    openstack share snapshot access list

    manila snapshot-create
    openstack share snapshot create

    manila snapshot-delete
    openstack share snapshot delete

    manila snapshot-export-location-list
    openstack share snapshot export location list

    manila snapshot-export-location-show
    openstack share snapshot export location show

    manila snapshot-force-delete
    openstack share snapshot delete --force

    manila snapshot-list
    openstack share snapshot list

    manila snapshot-manage
    openstack share snapshot adopt

    manila snapshot-unmanage
    openstack share snapshot abandon

    manila snapshot-rename
    openstack share snapshot set --name
    openstack share snapshot unset --name

    manila snapshot-reset-state
    openstack share snapshot set --status

    manila snapshot-show
    openstack share snapshot show


Share snapshot instances
------------------------

::

    manila snapshot-instance-list
    openstack share snapshot instance list

    manila snapshot-instance-show
    openstack share snapshot instance show

    manila snapshot-instance-reset-state
    openstack share snapshot instance set --status

    manila snapshot-instance-export-location list
    openstack share snapshot instance export location list

    manila snapshot-instance-export-location-show
    openstack share snapshot instance export location show

Share networks
--------------

::

    manila share-network-create
    openstack share network create

    manila share-network-delete
    openstack share network delete

    manila share-network-list
    openstack share network list

    manila share-network-show
    openstack share network show

    manila share-network-update
    openstack share network set
    openstack share network unset

    manila share-network-security-service-add
    openstack share network security service create

    manila share-network-security-service-list
    openstack share network security service list

    manila share-network-security-service-remove
    openstack share network security service delete

Security services
-----------------

::

    manila security-service-create
    openstack share security service create

    manila security-service-delete
    openstack share security service delete

    manila security-service-list
    openstack share security service list

    manila security-service-show
    openstack share security service show

    manila security-service-update
    openstack share security service set
    openstack share security service unset

Share servers
-------------

::

    manila share-server-delete
    openstack share server delete

    manila share-server-details
    manila share server show
    openstack share server show

    manila share-server-list
    openstack share server list

    manila share-server-manage
    openstack share server adopt

    manila share-server-unmanage
    openstack share server abandon

    manila share-server-reset-state
    openstack share server set --status


Share instances
---------------

::

    manila share-instance-force-delete
    openstack share instance delete

    manila share-instance-list
    openstack share instance list

    manila share-instance-reset-state
    openstack share instance set --status

    manila share-instance-show
    openstack share instance show

Share instance export locations
-------------------------------

::

    manila share-instance-export-location-list
    openstack share instance export location list

    manila share-instance-export-location-show
    openstack share instance export location show

Share types
-----------

::

    manila type-create
    openstack share type create

    manila type-delete
    openstack share type delete

    manila type-key
    openstack share type set
    openstack share type unset

    manila type-list
    openstack share type list

    manila type-show
    manila extra-specs-list
    openstack share type show

    manila type-access-add
    openstack share type access create

    manila type-access-list
    openstack share type access list

    manila type-access-remove
    openstack share type access delete

Storage pools
-------------

::

    manila pool-list
    openstack share pool list

Services
--------

::

    manila service-enable
    manila service-disable
    openstack share service set

    manila service-list
    openstack share service list

Availability zones
------------------

::

    manila availability-zone-list

We must implement this as a subcommand to the existing
``openstack availability zone list`` command.

Manage and unmanage shares
--------------------------

::

    manila manage
    openstack share adopt

    manila unmanage
    openstack share abandon

Quota sets
----------

::

    manila quota-defaults
    openstack quota defaults

    manila quota-delete
    openstack quota delete

    manila quota-show
    openstack quota show

    manila quota-update
    openstack quota set

Quota class set
---------------

::

    manila quota-class-show
    openstack share quota class show

    manila quota-class-update
    openstack share quota class set

User messages
-------------

::

    manila message-delete
    openstack share message delete

    manila message-list
    openstack share message list

    manila message-show
    openstack share message show

Share access rules
------------------

::

    manila access-allow
    openstack share access create

    manila access-deny
    openstack share access delete

    manila access-list
    openstack share access list

    manila access-show
    openstack share access show

Share access rule metadata
--------------------------

::

    manila access-metadata
    openstack share access set --property
    openstack share access unset --property

Experimental APIs
-----------------

Share migration
---------------

::

    manila migration-start
    openstack share migration start

    manila migration-cancel
    openstack share migration cancel

    manila migration-complete
    openstack share migration complete

    manila migration-get-progress
    openstack share migration show

Share replicas
--------------

::

    manila share-replica-create
    openstack share replica create

    manila share-replica-delete
    openstack share replica delete

    manila share-replica-list
    openstack share replica list

    manila share-replica-promote
    openstack share replica promote

    manila share-replica-reset-replica-state
    manila share-replica-reset-state
    openstack share replica set --replica-state
    openstack share replica set --status

    manila share-replica-resync
    openstack share replica resync

    manila share-replica-show
    openstack share replica show

Share replica export locations
------------------------------

::

    manila share-replica-export-location-list
    openstack share replica export location list

    manila share-replica-export-location-show
    openstack share replica export location show

Share groups
------------

::

    manila share-group-create
    openstack share group create

    manila share-group-delete
    openstack share group delete

    manila share-group-list
    openstack share group list

    manila share-group-reset-state
    openstack share group set --status

    manila share-group-show
    openstack share group show

    manila share-group-update
    openstack share group set
    openstack share group unset

Share group types
-----------------

::

    manila share-group-type-access-add
    openstack share group type access create

    manila share-group-type-access-list
    openstack share group type list

    manila share-group-type-access-remove
    openstack share group type delete

    manila share-group-type-create
    openstack share group type create

    manila share-group-type-delete
    openstack share group type delete

    manila share-group-type-key
    openstack share group type set --key
    openstack share group type unset --key

    manila share-group-type-list
    openstack share group type list


Share group snapshots
---------------------

::

    manila share-group-snapshot-create
    openstack share group snapshot create

    manila share-group-snapshot-delete
    openstack share group snapshot delete

    manila share-group-snapshot-list
    openstack share group snapshot list

    manila share-group-snapshot-list-members
    openstack share group snapshot members list

    manila share-group-snapshot-reset-state
    openstack share group snapshot unset

    manila share-group-snapshot-show
    openstack share group snapshot show

    manila share-group-snapshot-update
    openstack share group snapshot set
    openstack share group snapshot unset

Alternatives
------------

* Continue using python-manilaclient as the manila client. Continue
  maintenance for python-manilaclient. Do not implement any openstack
  command.

Data model impact
-----------------

No impact on the data model.


REST API impact
---------------

No impact on the REST API.

Driver impact
-------------

No impact on the drivers.

Security impact
---------------

No impact on security.

Notifications impact
--------------------

No impact on notifications.

Other end user impact
---------------------

Users will be able to interact with Manila through python-openstackcli.
On the other hand, users may keep using manila outside of traditional
openstackcli.

Performance Impact
------------------

No impact on performance.

Other deployer impact
---------------------

No deployer impact.

Developer impact
----------------

No developer impact.


Implementation
==============

Assignee(s)
-----------

Primary assignee:

* Soledad Kuczala <sol.kuczala@gmail.com>

Other contributors:

* Goutham Pacha Ravi <gouthampravi@gmail.com>
* Sofia Enriquez <senrique@redhat.com>
* Victoria Martinez de la Cruz <victoria@redhat.com>

Work Items
----------

* Implement basic python-openstackclient shell support
* Implement shares and share-types support
* Implement limits
* Implement storage pools
* Implement services
* Implement share export locations, share metadata and share actions
* Implement share snapshots and share snapshot instances
* Implement share networks
* Implement security services
* Implement share servers
* Implement share instances and share instance export locations
* Implement availability zones
* Implement manage and unmanage shares
* Implement quota and quota class sets
* Implement user messages
* Implement share access rules and share access rule metadata
* Implement share migration (experimental)
* Implement share replicas and share replicas export locations (experimental)
* Implement share groups, share groups types and share group snapshots (experimental)

Dependencies
============

No dependencies at the moment of writing for this project.


Testing
=======

Unit tests will be required as part of the implementation for each of
the openstack commands.


Documentation Impact
====================

- End User guide
- Documentation in which there is CLI usage will need to be updated.
  For consistency sake, we expect to have good coverage before changing
  those docs. In order words, at least non-experimental functionality
  needs to be implemented before changing manila commands docs for
  OpenStack commands.


References
==========

* http://lists.openstack.org/pipermail/openstack-discuss/2019-January/002271.html
