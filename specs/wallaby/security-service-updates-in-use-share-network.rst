..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================================
Security Service Updates for In Use Share Networks
==================================================

https://blueprints.launchpad.net/manila/+spec/improve-security-service-update

https://blueprints.launchpad.net/manila/+spec/add-security-service-in-use-share-networks

Manila security service is an entity that stores client configuration used for
authentication and authorization. It abstracts a set of configuration data
that can be used by back ends that support it, to configure a server that can
join specific authentication domains.

Manila supports different types of security services such as LDAP, Kerberos and
Microsoft Active Directory [1]. Users can create, update, view and delete
security services, however, in order to apply its configuration, manila needs
to maintain an association between security services and share networks.
Although, there is no way of providing such association or update any
of its parameters after a share network starts being used by a share server.

This spec proposes an improvement on both security service update and share
network association with security services, that will allow users update their
share networks to replace or to add new security services, which can reflect
on modifications in all related share servers.

Problem Description
===================

Security service has configurations that might change while it's still
associated to share servers, leaving the latter outdated and possibly with
problems to authenticate within domains. Furthermore, users won't be able to
grant access to shares that live on these share servers, and Manila doesn't
provide an easy way of updating such configurations, since the current
implementation only supports update of ``name`` and ``description`` attributes
[2] since they do not affect share server's configuration.

Likewise, share network association with security services can't be changed
after a share server has been provisioned, and new authentication server
configurations can't be added to the already deployed share servers. In this
scenario, users can only create new share networks that will be used to create
new share servers.

Use Cases
=========

Security service attributes such as ``dns`` and ``password`` might change due
to network changes or password update policies and hence affect one or more
share servers. Users must be able to modify these attributes in all affected
security services which must reflect such configuration change in all
associated share servers. A security service is qualified to be updated only
if all its associated share servers have support of such operation,
otherwise it must be denied.

Moreover, new security services might need to be associated with in use share
networks, in order to bring more flexibility to users that want to create new
shares using different authentication methods. Users must be able to include
new security service associations to their share networks, which must reflect
on security service updates in all associated share servers. A new security
service association is qualified to be configured only if all affected share
servers support such capability.

This specification focuses on both scenarios where share networks are being
used by share servers and an update or a new association of security
service is requested by the user, which will end up with one or more share
servers being updated in the back end storages.

Proposed Solution
=================

To solve these use cases, the share network entity will need be improved to
support updates and association of new security services, even when it is
already in use. A new ``status`` field will be added to represent the current
state of a share network. While under modification, the share network
will stay in a maintenance mode, meaning that back end configurations are being
changed. The update will finish when all affected resources finish their
respective configurations. Users will be able to check the current status of a
share network and will be blocked of requesting new operations that can be
affected by the current modification in progress.

To have these new operations working, the following changes are being proposed:

* Share network will have a new ``status`` attribute;
* Share network API will be updated to allow association and update of security
  services for in use share networks. The API will also be improved to accept
  share network ``reset-status`` operation.
* Update share server model to include a new capability, for supporting
  security service updates.
* Add a new share server state, indicating that it's under network
  modification.
* New share network states will be added to cover all scenarios.
* Add a new driver interface for share server update;
* Add a new back end capability to identify drivers that support such
  functionality;
* A new share network property will be added to indicate if it supports
  security service updates based on its share servers' capabilities.

API Changes
-----------

When receiving a request for security service update or association, the API
will check if there are share servers associated with the share network. If the
list of share servers is not empty, the API will need to check the new
share network's property, ``security_service_update_support``, to see if the
request can be completed or not and fail earlier if doesn't support it.

A new API will be create to handle security service update, which will allow
the replacement of an already configured security service by another one, that
should be of the same type.

The new share network's status will need to be validated in many other
resources' operations, which end up on backend changes. These operations should
wait until the associated share network become available again, in order to
proceed with the new modifications.

Share API and Manager Changes
-----------------------------

Share network updates can be an one-to-many operation, which might trigger
multiple share server updates, for different back ends in different share
services. To update a security service information associated with a share
network, the share API will need to validate if all affected resources are
healthy before proceeding with the request.

After that, both share network and share servers status will be updated to
``network_change``, the share network and security service association in the
database will be updated and the respective share services will be called to
initiate back end updates.

The share manager will be responsible for updating share servers' backend
details, and for calling the driver's interface with the corresponding
network update. At the end of the share server update operation, the share
server's status will be updated accordingly, while the share network will
be updated only if all share servers under modification already finished
their configuration.

If the update ends up on a failure, the share server status will be set to
``error`` along with all affected shares access rules. The share server's
backend details will remain with the most recent security service information,
but will be up to the administrator to check in the storage system if the
configuration is correct, before reset the status back to ``active``,
using the ``share-server-reset-state`` API. The network status won't be
affected by errors raised by back end drivers, and will have its status
updated to ``active`` again at the end of the security service update.

Scheduler Capability
--------------------

Backends that support this functionality will be able to report the
`security_service_update_support` capability as `True` to the scheduler, which
can further be used to select backend pools that support it.

Manila Manage
-------------

By adding a new capability to the share server model, it's important to
consider that existing share servers will need to update this field in the
future, based on driver's support, to have this functionality enabled.
This will be achieved by providing a new management command for
``manila-manage`` tool that will let administrators update their share servers
accordingly.

Alternatives
------------

Alternatively, the security service resource could handle the update operation
and trigger back end modification for all share servers associated with the
affected share networks. This alternative can lead to scenarios with lots of
back end modification at once, and as possible result, many failed share
servers if the new parameters or associations are invalid.

Impacts
=======

Data model impact
-----------------

A new ``security_service_update_support`` capability field will be added to
``manila.db.sqlalchemy.models.ShareServer`` indicating if the driver and the
back end where this share server resides support the new security service
update operations. In database migration upgrade, the new column will be
added with a default value set to ``False``, meaning that all share servers
already deployed won't be able to update their security service configuration
even if the driver supports it. In database migration downgrade the column will
be dropped.

A new ``status`` field will be added to the
``manila.db.sqlalchemy.models.ShareNetwork`` to hold new states that will
assist on different share network operations. At this moment, only three status
could be assigned to share networks: ``active``, ``error`` and
``network_change``. The share network is considered healthy and is available
to be used only if its status is ``active``. The status ``network_change``
represents a share network that is under modification and can't be changed or
used until it becomes ``active`` again. For specific failure scenarios, that
can't be recovered without administrator intervention, the share network
will receive an ``error`` status.

A new ``security_service_update_support`` property will be added to
``manila.db.sqlalchemy.models.ShareNetwork`` to indicate whether a share
network supports or not the new security service update operations. This
property will inherited its value from all current associated share servers. If
all associated share servers support ``security_service_update_support``, the
share network property will be set to ``True``, otherwise it will be set to
``False``.

REST API impact
---------------

* Both share server and share network view will include a new response
  parameter, the ``security_service_update_support`` that will indicate if the
  share server (or share network) is capable of updating security service
  configuration.
* Share network view will include a new ``status`` parameter that will indicate
  its current status.
* All share network dependant operations will be blocked in the API for share
  networks with status different from ``active``, to avoid conflicting
  backend configurations.
* Semantic changes are expected in the share network API. Associating new
  security services for in use share networks will require that all share
  servers affected by the change support such operation, or the API will
  will respond with ``403 Forbidden``. If the destination share service
  is unavailable, the API will respond with ``409 Conflict``.
* New share network APIs will be added:

1) ``share-network-security-service-update``

   Updates an existing security service association::

     POST /v2/{project_id}/share-networks/{share_network_id}/action

   Body::

     {
       "update_security_service": {
         "current_service_id": "bab0debd-fa50-4ade-80c5-ce85e2fc2614",
         "new_service_id": "1ec35b2b-5d0c-4c46-a6ca-71f9eab49ce2"
       }
     }

   Response::

     Code: 200 OK

     {
       "name": "my_network",
       "id": "620a1050-1711-4961-908a-bd6f7c0b1d00",
       "project_id": "f227b9e2cbdc40e78ed761e5e22c5fb4",
       "description": "This is my share network",
       "created_at": "2020-11-27T11:26:10.000000"
       "updated_at": null,
       "status": "network_change",
       "security_service_update_support": true,
       "share_network_subnets": [
         {
           "created_at": "2020-11-27T11:26:10.000000",
           "id": "aa7a1269-703b-4832-a3df-17ed954c276c",
           "availability_zone": null,
           "segmentation_id": null,
           "neutron_subnet_id": "12c1490a-e82c-4f5e-bcb1-fb267a58cf10",
           "updated_at": null,
           "neutron_net_id": "998b42ee-2cee-4d36-8b95-67b5ca1f2109",
           "ip_version": null,
           "cidr": null,
           "network_type": null,
           "gateway": null,
           "mtu": null
         }
       ]
     }

   If the provided `new_service_id` doesn't have the same type as the current
   one, or one of the security services' ID don't exist, the API will respond
   with `400 Bad Request`. If the provided `share-network-id` doesn't exist,
   the API will respond with ``404 Not Found``. If at least one of the
   destinations share services is unavailable, the API will respond with
   ``409 Conflict``.

2) ``share-network-reset-status``:

   Reset the status of a share network::

     POST /v2/{project_id}/share-networks/{share_network_id}/action

   Body::

     {
       "reset_status": {
       "status": "active"
       }
     }

   If the provided `share-network-id` doesn't exist, the API will respond with
   ``404 Not Found``. If the user doesn't have permission to execute this
   operation, the API will respond ``403 Forbidden``.

Security impact
---------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

None.

Performance impact
------------------

None.

Other deployer impact
---------------------

* No new configurations are expected to be added.
* The back end capability will help deployers to identify pools that support
  new security service association.
* All existing share servers will have their
  ``security_service_update_support`` set to ``False``, even if the
  driver supports it. New share servers will have the correspondent capability
  set according to the back end capability reported by the drivers.
  Administrators will need to manually update share server's  capability using
  ``manila manage`` commands.

Developer impact
----------------

None

Driver impact
-------------
There is no impact for drivers that don't want to support the proposed feature.

A new driver interface will be included to support share server security
service update. Both share server and network info will be provided during this
call. Drivers that want to support this operation will need to implement this
call and report the capability ``security_service_update_support`` as ``True``.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  | dviroel

Work Items
----------

* Implement core changes that must include:

  * Add share network and share server model attributes;
  * Change API behavior when associating new security services with a share
    network;
  * Create new share network APIs to update security service and reset share
    network status;
  * New Share API interface for updating security service information;
  * New driver interface will be included to update share server network
    configuration;
  * New back end capability for supporting security service association will be
    added to share driver.

* Implementation in a first party driver
* Add new functional test in manila-tempest-plugin
* Add new command to manila-manage for share server capability update
* Update manila documentation

Dependencies
============

None.

Testing
=======

Unit test coverage will be added/maintained as per community standards.
New tempest tests will be added to cover new security service association
scenarios. The container or the dummy driver will be improved to properly
configure security services and be used to validate the proposed changes.

Documentation Impact
====================

The following OpenStack documentation will be updated to reflect this change:

* Admin Guide: document the premises for having new security service
  association applied to share servers;
* API Reference: include share server new status and new capability field;
* Developer Reference: add information about how to implement and add support
  for this new functionality.

References
==========

[1]: https://docs.openstack.org/manila/latest/admin/shared-file-systems-security-services.html

[2]: https://docs.openstack.org/api-ref/shared-file-system/#update-security-service
