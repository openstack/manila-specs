..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================
Storage Availability Zone improvements in Stein
===============================================

https://blueprints.launchpad.net/manila/+spec/per-backend-availability-zones

https://blueprints.launchpad.net/manila/+spec/export-locations-az

https://blueprints.launchpad.net/manila/+spec/share-type-supported-azs

Manila supports the concept of storage availability zones (AZ) (configuration
option: ``storage_availability_zone``). A storage AZ is a string that
provides a loose, flexible and conceptual representation of physical storage
resources that share a common fate in terms of failure. Manila provides an
API to discover what AZs have been configured. A user can request for
their share to be scheduled to a specific AZ. Shares can also be replicated
between AZs.

Availability zones are also used across other OpenStack services, such as nova
and cinder, as a way of logically partitioning resources within an OpenStack
deployment. In Nova, this partitioning can be motivated by the fact that these
resources are meant for a particular task, or because they are
co-located. However, the most common criterion is because these resources
are associated with a common point of failure, such as being connected to a
common power source. In the Mitaka release of OpenStack, neutron added support
to availability zones. With this, it is effectively possible for
users to allocate network resources to AZs for high availability.

As new use cases of OpenStack evolve, the concept of storage availability
zones becomes more important to build features around. OpenStack's use in
Edge computing is built around the concept of stretch OpenStack clusters and
split control planes spanning multiple "zones" in a single "region" [#]_. It
is important for manila to clearly define its availability zones construct
so it can be used in an extensible manner in all use cases.


Problem description
===================

This spec highlights three problems with the existing support for storage
AZs:

The coupling between Service Availability and Storage Availability
------------------------------------------------------------------

Manila expects the ``storage_availability_zone`` config option to be specified
in the ``[DEFAULT]`` section of the configuration file for the share manager
service. At service startup, all back ends configured within the same
configuration file register with the AZ specified. There are two instances
where this is not desirable:

Service availability is not always the same thing as storage availability.
The manila-share manager process can be run directly on
dedicated "storage" nodes and this flavor is usually seen when
using cloud-local storage such as LVM, ZFSOnLinux or cinder-backed block
storage (Generic Driver). In such cases, it is expected that the service
AZ is the same as storage AZ.

However, in many deployments, manila's control plane is not in the same failure
domain as the storage it manages. For example, consider an OpenStack cloud
that has 3 "controller" nodes, and manila's control plane (api, scheduler,
share-manager and data processes) runs on all three controller nodes. This
cloud can have a third party storage system that is external to the
cloud as a manila back end. In this model, the share-manager process and
its storage back end are in distinct failure domains. Typically, to achieve
high availability of the control plane, cloud administrators run multiple
copies of the manila share-manager service (e.g. on every OpenStack
controller node) and orchestrate the high availability outside of manila
with tools such as Pacemaker/Corosync or just by configuring each service to
listen to the same service message queue channel/topic (configuration option:
``host``).

The second limitation is the inflexibility in case of multi-backend.
Consider a case of a deployment with both cloud-local storage and
third-party external storage managed by the same share-manager service.
The share-manager service forks a separate process to manage each back
end. However, since the parent service consumes a single configuration file,
administrators can only specify one ``storage_availability_zone`` for both
back ends, even while that is truly not the case.

Discovering the right exports to use
------------------------------------

This problem is specific to the user experience with replicated shares.
There may sometimes be data path connectivity across AZs. However,
more often than not, high bandwidth data networks (which shares are exported
on) are local and distinct to AZs. Regardless, users must be able to
discover which export path should be used within a given AZ when consuming a
replicated share. Currently, users do not have a way to query manila for export
locations that pertain to a specific AZ.

No relationship between share types and availability zones
----------------------------------------------------------

Cloud administrators create share types and allocate back ends within
storage availability zones, however, end users currently will not know if a
particular share type is supported within a given AZ. If there are no back
ends that match the share type used when shares are scheduled to an AZ,
an asynchronous scheduling failure occurs.


Use Cases
=========

- Cloud Administrators must be able to configure
  ``storage_availability_zone`` for each back end
- Users must be able to discover export locations for a given availability
  zone
- Users must be able to discover share types available within a given
  availability zone and availability zones supported by a given share type


Proposed change
===============

**Allow configuring AZ per back end**

We will move the configuration option ``storage_availability_zone`` from the
``[DEFAULT]`` section to the driver specific section. Configuring
``storage_availability_zone`` in the ``[DEFAULT]`` section will be
deprecated, but not unsupported, keeping in mind upgrade impact to the
deployers. See corresponding cinder change here: [#]_

**Export Locations changes**

The export locations API (``GET shares/{share_id}/export_locations``) will be
modified to only present export locations from ``active`` replicas of a
share. This API change will be micro-versioned. This implies that when the
API is invoked on a non-replicated share, it will present all export
locations of the share. Users of replicated shares can query export
locations of replicas via a new API. The new API will present the export
locations, their existing driver driven "metadata" and relevant replica
information (replica ID, replica state, availability zone) to discern which
exports are appropriate to be used. These API changes make the APIs leaner,
but the UI will be designed to collate information from different APIs where
necessary to present relevant information together.

**Share Type changes**

A new optional user/tenant-visible extra-spec will be created, called
"availability_zones". The default value of this extra-spec when not
specified will be '*', and this value is shown to users and administrators.
Cloud administrators can override this default with a comma separated list
of availability zones that the share type is supported within. Share types
can be filtered with ``availability_zones`` by specifying a comma separated
list of availability zones.

.. note::

   Share types are mutable. Any changes to the extra-specs associated with a
   share type do not affect existing shares of that type. Cloud
   administrators can update the value of the ``availability_zones``
   extra-spec at any time, but they must be aware that modifying
   tenant-visible extra-specs (without also modifying affected properties of
   pre-existing shares) may confuse end users.

**AZ Scheduling changes**

If an AZ is not chosen by the user to create a new share, the Availability
Zone scheduler filter (``manila.scheduler.filters.availability_zone
.AvailabilityZoneFilter``) will consider the ``availability_zones``
extra-spec to filter backends to scheduler the share.

See the corresponding changes to cinder's volume type here [#]_ and here [#]_


Workflows affected
------------------

**Retrieving Share Types**

The share types API will be modified to support ``availability_zones`` as a
user-visible "optional" extra-spec. When not set, its value is set to "*"
signifying that all availability zones are supported. This field can be
filtered with one or more availability zones. Share types can also be
filtered by ``availability_zones=*`` which will retrieve only those share
types that support all availability zones. On the UI, if a share type is
chosen, only a list of supported availability zones will be displayed and
vice-versa.

**Creating a share**

The share API will validate the share type's support of an availability
zone if the share is requested to be created within an availability zone. If
the share type does not support the requested availability zone, the API
will return HTTP 400. When using the CLI, users typically list share types
and availability zones and pick a share type and availability zone to invoke
the ``manila create`` command. With this change, the ``manila
share-type-list`` command will display the ``availability_zones``
extra-spec so they can make a wise choice. There will be no validation on
the CLI with respect to the share-types and AZs to prevent a performance
regression, the API will carry a clear error message that should suffice.

**Retrieving export locations**

Users will no longer be able to retrieve the export locations of replicas when
using the export locations API. The share instance export locations
API will not be altered, so consumers of that API will not be affected. The
share replica export locations API can be used to retrieve details of all
replica export locations for a given share, or export locations for a specific
replica of a share or detailed export location information for a specific
export.

**Creating a share group**

A share group can be created within a specified availability zone. The share
group API will check whether the availability zone is supported within the
share types used when specified. On the UI, when an availability zone is
chosen to schedule a share group, the list of share types will be filtered
based on support for that availability zone.


Alternatives
============

Currently, when administrators want to configure multiple availability
zones, they configure multiple manila share-manager services, each with its
own configuration file. This requires deployer side tooling and
duplicates the effort of the share-manager service itself spawning processes
to manage its multiple back ends.

Users currently have hacky/non-documented ways of figuring out if export
locations are optimized (or even work) in a specific AZ. One approach is to
rely on the possibility that export locations contains the share "instance" ID
as a substring. However, this approach doesn't work for non-replicated
shares, since users (by virtue of default policy) cannot list share
instances. They can only see IDs of replicas of a share (which are share
instances under the hood). Another approach is to attempt to mount the share
with each export location in the list, and sticking with the one that
connects, or one that is the fastest (as determined by some user-driven test).

Currently share types being unavailable in specific AZs causes an
asynchronous failure which can be diagnosed through user messages. We could
live with this user experience.


Data model impact
=================

No database schema changes are proposed. Therefore, no database migrations
will be committed.

REST API impact
===============

Please note the current state of these APIs in our
`API reference
<https://developer.openstack.org/api-ref/shared-file-system/index.html>`_.

**List Share types**::

    GET /v2/{tenant_id}/types?availability_zones=az1

Response::

    Code: 200 OK

    {
        "volume_types": [
            ..
        ],
        "share_types": [
            {
               "required_extra_specs": {
                    "driver_handles_share_servers": "True"
                },
               "share_type_access:is_public": true,
               "extra_specs": {
                    "driver_handles_share_servers": "True",
                    "mount_snapshot_support": "False",
                    "revert_to_snapshot_support": "False",
                    "create_share_from_snapshot_support": "True",
                    "snapshot_support": "True",
                    "availability_zones": "az1,az4"
                },
                "id": "7fa1342b-de9d-4d89-bdc8-af67795c0e52",
                "name": "testing",
                "is_default": false,
                "description": "share type description"
            }
        ]
    }


**Get share type**::

    GET /v2/{tenant_id}/types/{share_type_id}

Response::

    Code: 200 OK
    {
        "share_type": {
            "required_extra_specs": {
                "driver_handles_share_servers": "True"
            },
            "share_type_access:is_public": true,
            "extra_specs": {
                "driver_handles_share_servers": "True",
                "mount_snapshot_support": "False",
                "revert_to_snapshot_support": "False",
                "create_share_from_snapshot_support": "True",
                "snapshot_support": "True",
                "availability_zones": "*"
            },
            "id": "2780fc88-526b-464a-a72c-ecb83f0e3929",
            "name": "default-share-type",
            "is_default": true,
            "description": "default share type"
        },
        "volume_type": {
            ..
        }
    }


A similar schema change will be done for the following APIs::

    GET /v2/{tenant_id}/types/default
    GET /v2/{tenant_id}/types/{share_type_id}/extra_specs
    POST /v2/{tenant_id}/types


**List share export locations**:

The change to this API is to remove non-active replica locations from the
response schema::

    GET /v2/{tenant_id}/shares/{share_id}/export_locations

Response::

    Code: 200 OK
    {
        "export_locations": [
            {
                "path": "10.254.0.3:/shares/share-e1c2d35e-fe67-4028-ad7a-45f668732b1d",
                "id": "b6bd76ce-12a2-42a9-a30a-8a43b503867d",
                "preferred": false,
            },
            {
                "path": "[db9f:6954::7766]:/shares/share-e1c2d35e-fe67-4028-ad7a-45f668732b1d",
                "id": "a16ef0be-9181-40af-a61c-7764816bc08d",
                "preferred": true,
            }
        ]
    }

**Share replica's export locations**::

    GET /v2/{tenant_id}/share-replicas/{share_replica_id}/export_locations

Response::

    Code: 200 OK
    {
        "export_locations": [
            {
                "path": "10.254.0.3:/shares/share-8acb2fc3-7139-434f-8637-1ad7f49ee881",
                "share_replica_id": "8acb2fc3-7139-434f-8637-1ad7f49ee881",
                "replica_state": "in_sync",
                "id": "8da0a189-8365-4c28-919b-9f07c4f06c65",
                "preferred": false,
                "availability_zone": "northYVZ"
            },
            {
                "path": "[db9f:6954::7766]:/shares/share-e1c2d35e-fe67-4028-ad7a-45f668732b1d",
                "share_replica_id": "e1c2d35e-fe67-4028-ad7a-45f668732b1d",
                "replica_state": "in_sync",
                "id": "ad19597c-4d04-4869-af5f-c8173c2bcd51",
                "preferred": true,
                "availability_zone": "northYVZ"
            }
        ]
    }


Share replica export location::

    GET /v2/{tenant_id}/share-replicas/{share_replica_id}/export_locations/{export_location_id}

Response::

    Code: 200 OK
    {
        "export_location": {
                "path": "10.254.0.3:/shares/share-8acb2fc3-7139-434f-8637-1ad7f49ee881",
                "share_replica_id": "8acb2fc3-7139-434f-8637-1ad7f49ee881",
                "replica_state": "in_sync",
                "id": "8da0a189-8365-4c28-919b-9f07c4f06c65",
                "preferred": false,
                "availability_zone": "northYVZ",
                "created_at": 2018-11-27T22:54:32.000000,
                "updated_at": 2018-11-27T22:54:38.000000
            }
    }

Security impact
===============

None


Notifications impact
====================

None


Other end user impact
=====================

If administrators do not configure the ``availability_zones`` extra-spec, no
API change will be observed by the end user. CLI and UI impact are as follows:

- manilaclient and CLI: ``manila show`` and ``manila
  share-export-location-list`` will no longer show export locations of
  non-active replicas of a given share. A new command will be added ``manila
  share-replica-export-location-list`` which will accept the replica ID as a
  parameter to retrieve the replica export locations. Users can also use the
  ``manila share-replica-export-location-show`` command along with the
  replica ID and export location ID to retrieve details about a specific
  replica export location. These commands will have corresponding
  manilaclient implementations that will allow users of the python package
  to retrieve share replica export locations.
- manila UI: Users will only see active replica export locations on
  the UI in the share details page. The UI will use the newly created client
  integrations to invoke the share replica export locations API to retrieve
  export locations and display them in the share replica details page. This
  approach will resolve LP 1787016 [#]_.


Performance impact
==================

Filtering the non-active replicas out of the export locations API is
expected to add a slight/negligible performance regression since we will be
performing a joined load of the export locations and the share instances
associated with them.


Other deployer impact
=====================

- ``storage_availability_zone`` will be deprecated from the ``[DEFAULT]``
  group.


Developer impact
================

API microversion will be bumped to expose new functionality. Backwards
compatibility will be strictly maintained.


Driver impact
=============

There is no third-party driver change anticipated. The availability zone
configuration changes will be done in a generic fashion within the base
share driver. All other proposed changes do not affect share drivers directly.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  | gouthamr


Work Items
----------

* Move config-opt ``storage_availability_zone`` to driver/back end sections
* Modify the export locations API and introduce share replica export
  locations APIs.
* Add support for configuring ``availability_zones`` in share types
* Filter share types in the UI by availability zones and vice-versa during
  share and share group creation phases.

Dependencies
============

None

Testing
=======

Unit test coverage will be added/maintained as per community standards.
Tempest tests will be modified/added to cover new API changes. Allowing
multiple AZs in a single multi-backend style configuration file will
simplify test environment setup. The DevStack plugin already supports
multi-backend, it can support multi-AZ without significant changes to either
the plugin or the scripts invoking it.

The dummy driver and LVM driver job configuration on the gate will be modified
to support multiple availability zones on a single share-manager service.


Documentation Impact
====================

The following OpenStack documentation will be updated to reflect this change:

* User Guide: document the changes in export location APIs,
  CLI and GUI and the changes to share types
* Admin Guide: document the theory of supporting multiple availability zones
  per share-manager service.
* API Reference: All API changes will be documented
* Manila Developer Reference: the low level implementation considerations
  and design of this feature will be documented.

References
==========

.. [#] Edge computing architectures https://wiki.openstack.org/wiki/Edge_Computing_Group/Edge_Reference_Architectures
.. [#] Change to move AZ config to back ends in Cinder: https://review.openstack.org/#/c/433437/
.. [#] Specification for AZ support in Block Storage Volume Types: http://specs.openstack.org/openstack/cinder-specs/specs/rocky/support-az-in-volume-type.html
.. [#] Code changes to support AZs in Block Storage Volume Types: https://review.openstack.org/#/c/552243/
.. [#] Manila UI bug: Unable to retrieve replica details as non-admin user: https://bugs.launchpad.net/manila-ui/+bug/1787016
