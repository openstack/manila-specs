..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================================
Making manila share networks span multiple subnets (with AZs)
=============================================================

https://blueprints.launchpad.net/manila/+spec/share-network-multiple-subnets

Manila can be deployed with share drivers that handle the creation and
management of share servers. In such a deployment, users create share
networks by specifying network and subnet IDs from neutron. If the share driver
is configured accordingly, a share server will be created with network
information from the specified neutron subnet. This network information
includes the IP addresses for the share server creation which are reflected in
the export locations of shares created by such drivers.

If the share driver is configured with a ``standalone`` network or with a
``single`` network configuration [1], the share server is created with the
network details from the ``manila.conf`` file. Network information from the
user specified share network is (currently) ignored in this case. There is
future work envisioned to provide network tunneling and hence, connectivity
between the configured and user specified networks in such cases.

Users use share networks to create shares. A share is associated with a
single share network and is exported off a share server that is also
associated with the share network. Users can also associate security
services such as LDAP, Active Directory or Kerberos with a share network,
thereby affecting all the shares on the share network.


Problem description
===================

When share replication [2] was designed in the Mitaka release of
OpenStack, we decided to make replication work within and across
availability zones. However, this feature was limited only to drivers with
pre-configured share servers (``DHSS = False`` mode). We realized that we
could not support replication of shares created with drivers that support
share server handling without making changes to the design of share networks.

With share replication, we recommend the use of availability zones as
**failure domains** within an OpenStack cloud. ``manila-share`` services
would be running across different availability zones when share
replication is used for disaster recovery workloads. Users would create their
primary share in one availability zone and replicas in different availability
zones. Therefore, share servers that are created to handle and export these
shares need to be created within respective availability zones.

Availability zones are used across other OpenStack services, such as nova
and cinder as a way of logically partitioning resources within an OpenStack
deployment. This partitioning can be motivated by the fact that these
resources are meant for a particular task, or because they are
co-located. However, the most common criterion is because these resources
are associated with a common point of failure, such as being connected to a
common power source. In the Mitaka release of OpenStack, neutron added support
to availability zones [3][4]. With this, it is effectively possible for
users to allocate network resources to availability zones for high
availability.

In the Liberty release, when share instances were added to support
features such as share replication and share migration, we acknowledged that
the user's share resources can live in multiple places simultaneously. These
places may be across different network subnets and/or availability zones.

Currently, there is no provision for users to isolate their manila network
resources, i.e share networks and share servers within availability zones that
they create their compute host aggregates or block/shared file system storage
resources within.


Use Cases
=========

- Providing a way for a share network to span multiple subnets allows for a
  more realistic representation of an internal data center network when the
  user means for a share network to signify network connectivity between
  different instances of his/her resources.

  Currently, when a user brings up a compute instance or a shared file
  system on a subnet, all other resources on that subnet have network
  connectivity to this resource. Resources on other subnets can "access"
  this resource if there is a router that can send and receive traffic
  between these subnets.

  When users create their shared file systems in a distributed manner,
  there is a very plausible chance that they mean for their mirrors (share
  replicas) or migration targets (share instances) to be on different subnets.

  At the same time, there is a possibility that deployers would choose to
  make giant subnets that span all the availability zones.

- Share servers in manila consume network allocations from a subnet. For
  advanced workloads such as share replication and share migration, they
  might often need to communicate between each other.

  Therefore, it makes semantic sense for users to create one share network
  and assign subnets to the share network. This would essentially tag
  all share servers that are managing share instances of a particular share
  with one network. Each individual share instance however is exported off
  a particular subnet.


Proposed change
===============

To solve the problem of allowing users to manage their share instances
(and consequently share servers) within failure domains and to allow for a
better representation of the instances of a shared file system across the
user's OpenStack cloud:

- Share networks will span multiple subnets. These subnets would be associated
  with an availability zone each.
- There can be a subnet associated with a share network that is
  associated with all storage availability zones. For the purposes of this
  specification, we will call this a ``default`` subnet. While not being
  associated with any particular availability zone, this ``default`` subnet
  is meant to service all availability zones.
- Each share network can have no more than one ``default`` subnet.
- A share network need not have a ``default`` subnet.
- Users will have the ability to assign one subnet per configured storage
  availability zone per share network. They can assign the same subnet to
  different AZs if their setup permits, but they cannot have more than one
  subnet serving an AZ in particular, per share network.

.. note::

    We could consider allowing more than one fall-back or ``default``
    subnet per share network; Similarly, we could allow more than one subnet
    to be specified per share network for a given availability zone.

    However, when a user requests to create a resource in a particular AZ,
    manila will then have no way of picking the right subnet of the choices
    it would have.

    To avoid any indeterminate (and hence wrong) behavior, we will not allow
    more than one ``default`` subnet per share network; or more than one
    subnet per availability zone.

- API changes would involve extending the ability to modify a share
  network by adding and removing subnets (with pre-conditions
  related to existing shares and share servers).
- Users would need to pick the share network and an AZ during share
  creation if they expect a share to be provisioned with a specific subnet.
  They only need to pick the AZ during share replica creation.

Things that will not change:

- A share would still only be associated with one share network.
- A share server would still be allocated IPs from a single subnet's
  allocation pool.
- A security service would continue to be associated with the share
  network. When share servers are created on new subnets, they will derive
  all the security server information from the share network.


Work-flows affected
-------------------

**Creating a share network**

Users can continue to create a share network with a single subnet, but they
would have to provide an availability zone to assign the subnet to. The user
may also intend to create a subnet that spans all availability zones. This
subnet could act as the ``default`` or ``fallback`` subnet when a dedicated
subnet is not available to service a given availability zone. However, the
user may not have more than one of these ``default`` subnets per share network.

Alternatively, users can also create "empty" share networks, as is the case
today.

**Creation of a share server**

Share servers will still be created by manila-share service with network
allocations from a single subnet.

**Creating a share**

Users can provide the share network to create the share with, as before.
However, if there is more than one subnet associated with the share network,
they would ideally provide an availability zone as well to create it on a
particular subnet (perhaps for co-location purposes with other resources
they care about on their OpenStack cloud).

If the user provides a share network and an availability zone to create the
share within, the API will validate that the share network supports that AZ
(i.e, either has a subnet assigned to the specified AZ or has a ``default``
subnet). The API will respond with a ``400 BadRequest`` if this condition
fails.

**Creating a share replica**

Users would provide the share network and AZ during creation of the primary
share. They would only need to provide an availability zone to create the
share replica within. The share network is inherited from the parent share
object and the specific subnet is chosen in association with the AZ provided.

.. note::

    Currently, the API to create a share replica accepts
    ``share_network_id`` as an optional parameter. This parameter will be
    removed to support this workflow. Since there are no share drivers that
    work in ``DHSS = True`` mode and support share replication, we can
    remove this unused parameter. See associated launchpad bug [5].

**Assigning a security service**

Users will continue to assign security services to share networks, no
changes are proposed to this workflow.


Alternatives
------------

We can keep the existing design of share networks, but allow related
resources to be created in multiple share networks. A share network will
not span subnets, and users would require to use a different share
network for each instance of a share. They would have to configure security
services to each of these networks. For share replication, effectively, the
share network of the share would be the share network in which the ``active``
replica instance is created.

There are limitations of this approach when the scope of future expansion is
to be considered. Ideally, we would like users to be able to control their
resources with respect to availability zones, and this approach lends itself
poorly to that goal.

Quotas are currently enforced on the number of share networks. If a user
needs to deal with multiple subnets as multiple share networks, quota
limitations would be a problem. As a user, I might want to divide my network
resources by availability zones, without consuming the network quotas,
because they are all being used to manage the same share (and its replicas
or instances), or the same security service.


Data model impact
-----------------

The ``manila.db.sqlalchemy.models.ShareNetwork`` model will no longer
contain the subnet specific information: ``neutron_net_id``,
``neutron_subnet_id``, ``network_type``, ``segmentation_id``, ``cidr`` and
``ip_version``. These keys will be part of a new model ``manila.db
.sqlalchemy.models.ShareNetworkSubnet`` which has a primary key: ``id``
(we'll refer to this as ``share_network_subnet_id`` for the purposes of this
document) and foreign keys, ``availability_zone_id`` and
``share_network_id``. A share network subnet is only associated with
exactly one share network and one availability zone.

The ``manila.db.sqlalchemy.models.ShareServer`` model will stop having a
foreign key binding to ``share_network_id`` and switch to having a
foreign key reference to ``share_network_subnet_id`` instead.

No key changes are going to be made to
``manila.db.sqlalchemy.models.ShareNetworkSecurityServiceAssociation``,
``manila.db.sqlalchemy.models.SecurityService``, ``manila.db.sqlalchemy
.models.ShareInstance`` or ``manila.db.sqlalchemy.models.ConsistencyGroup``
models. All these resources will continue to  contain a foreign key reference
to the ``manila.db.sqlalchemy.models.ShareNetwork`` model.

The database upgrade step will create a new share network subnet per
existing share network. The ``availability_zone_id`` field will be ``null``
indicating that these are ``default`` subnets, unconfined to a given
availability zone.

The database downgrade step will collapse existing share network subnets
into the share network table. This step may result in loss of information
if multiple subnets exist per share network. Hence, it is not recommended in a
production cloud.

REST API impact
---------------

Please note the current state of these APIs in our
`API reference <http://developer.openstack.org/api-ref-share-v2.html>`_.

**Creating a share network**::

    POST /v2/​{tenant_id}​/share-networks

Request::

    {
        "share_network": {
            "neutron_net_id": "998b42ee-2cee-4d36-8b95-67b5ca1f2109",
            "neutron_subnet_id": "53482b62-2c84-4a53-b6ab-30d9d9800d06",
            "name": "my_network",
            "description": "This is my share network",
            "availability_zone": "london",
        }
    }

Subnet details ``neutron_net_id`` and ``neutron_subnet_id`` are optional.
Although, whenever one of them is specified, both must to be specified,
otherwise the API will respond with ``400 Bad Request``.

``availability_zone`` is optional if one of the following conditions are
met:

* user is creating a subnet that is meant to span all availability zones.
* ``manila-share`` services are only configured with a single availability
  zone, or,
* when subnet details are not provided.

If the ``availability_zone`` is not known to manila, the API will respond
with ``400 Bad Request``.

If the tenant's share network quota has exceeded, the API will respond with
``403 Forbidden``.

The API will not validate the subnet information with the network provider
(neutron). The validation will be performed at the share manager layer.

Response::

    Code: 202 Accepted

    {
        "share_network": {
            "name": "my_network",
            "created_at": "2016-06-01T21:12:12.617687",
            "id": "77eb3421-4549-4789-ac39-0d5185d68c29",
            "project_id": "e10a683c20da41248cfd5e1ab3d88c62",
            "description": "This is my share network",
            "updated_at": null
        }
    }


**Adding a subnet to a share network**::

    POST /v2/​{tenant_id}​/share-networks/{share_network_id}/subnets

Request::

    {
        "share-network-subnet" : {
            "neutron_net_id": "998b42ee-2cee-4d36-8b95-67b5ca1f2109",
            "neutron_subnet_id": "12c1490a-e82c-4f5e-bcb1-fb267a58cf10",
            "availability_zone": "paris"
        }
    }

The same conditions exist for these parameters as with the create API above.

If the share network ID is invalid, the API will respond with ``404
NotFound``.
If there already is a subnet defined for the availability zone specified,
the API will respond with ``409 Conflict``.

Response::

    Code: 202 Accepted

    {
        "share_network_subnet": {
            "created_at": "2016-06-01T21:12:14.843836",
            "id": "aa7a1269-703b-4832-a3df-17ed954c276c",
            "share_network_id": "77eb3421-4549-4789-ac39-0d5185d68c29",
            "availability_zone": "paris",
            "segmentation_id": null,
            "neutron_subnet_id": "12c1490a-e82c-4f5e-bcb1-fb267a58cf10",
            "updated_at": null,
            "neutron_net_id": "998b42ee-2cee-4d36-8b95-67b5ca1f2109",
            "ip_version": null,
            "cidr": null,
            "network_type": null
        }
    }


**Removing a subnet from a share network**::

    DELETE /v2/​{tenant_id}​/share-networks/{share_network_id}/subnets/{share-network-subnet-id}

If the share network ID or the share network subnet ID is invalid, the API
will respond with ``404 NotFound``.
If there is a share server on the share network subnet, it cannot be removed;
the API will respond with ``409 Conflict``.

Response::

    202 Accepted


**Summary listing of share networks**::

    GET /v2/​{tenant_id}​/share-networks

Response::

    Code: 200 OK

    {
        "share_networks": [
            {
                "id": "77eb3421-4549-4789-ac39-0d5185d68c29",
                "name": "my_network"
            },
            {
                "id": "00cc3770-2558-4370-a088-de03b055dcff",
                "name": "my_network_2"
            }
        ]
    }

No changes from the previous version of this API


**Detailed listing of share networks**::

    GET /v2/​{tenant_id}​/share-networks/detail

Response::

    Code: 200 OK

    {
        "share_networks": [
            {
                "name": "my_network",
                "id": "77eb3421-4549-4789-ac39-0d5185d68c29",
                "project_id": "e10a683c20da41248cfd5e1ab3d88c62",
                "description": "This is my share network",
                "updated_at": null,
                "share_network_subnets": [
                    {
                        "created_at": "2016-06-01T21:12:14.843836",
                        "id": "aa7a1269-703b-4832-a3df-17ed954c276c",
                        "availability_zone": "paris",
                        "segmentation_id": null,
                        "neutron_subnet_id": "12c1490a-e82c-4f5e-bcb1-fb267a58cf10",
                        "updated_at": null,
                        "neutron_net_id": "998b42ee-2cee-4d36-8b95-67b5ca1f2109",
                        "ip_version": null,
                        "cidr": null,
                        "network_type": null
                    },
                    {
                        "created_at": "2016-06-01T21:12:14.843836",
                        "id": "9c5dbe11-cdf6-48a4-b6ca-9582ef5af193",
                        "availability_zone": "london",
                        "segmentation_id": null,
                        "neutron_subnet_id": "53482b62-2c84-4a53-b6ab-30d9d9800d06",
                        "updated_at": null,
                        "neutron_net_id": "998b42ee-2cee-4d36-8b95-67b5ca1f2109",
                        "ip_version": null,
                        "cidr": null,
                        "network_type": null
                    }
                ]
            },
            {
                "name": "my_network_2",
                "id": "00cc3770-2558-4370-a088-de03b055dcff",
                "project_id": "e10a683c20da41248cfd5e1ab3d88c62",
                "description": "This is also my share network",
                "updated_at": null,
                "share_network_subnets": [
                    {
                        "created_at": "2016-06-01T21:12:14.843836",
                        "id": "1feffbd9-9747-413d-b162-83b97981f0ba",
                        "availability_zone": "paris",
                        "segmentation_id": null,
                        "neutron_subnet_id": "647cf190-e439-4159-98f9-17cb266d6d00",
                        "updated_at": null,
                        "neutron_net_id": "3b49335d-273e-4829-9b08-b7b1df157e69",
                        "ip_version": null,
                        "cidr": null,
                        "network_type": null
                    }
                ]
            }
        ]
    }


**Showing details of a single share network**::

    GET /v2/​{tenant_id}​/share-networks/{share_network_id}/detail

If the share network ID is invalid, the API will respond with ``404 NotFound``.

Response::

    Code: 200 OK

    {
        "name": "my_network",
        "id": "77eb3421-4549-4789-ac39-0d5185d68c29",
        "project_id": "e10a683c20da41248cfd5e1ab3d88c62",
        "description": "This is my share network",
        "updated_at": null,
        "share_network_subnets": [
            {
                "created_at": "2016-06-01T21:12:14.843836",
                "id": "aa7a1269-703b-4832-a3df-17ed954c276c",
                "availability_zone": "paris",
                "segmentation_id": null,
                "neutron_subnet_id": "12c1490a-e82c-4f5e-bcb1-fb267a58cf10",
                "updated_at": null,
                "neutron_net_id": "998b42ee-2cee-4d36-8b95-67b5ca1f2109",
                "ip_version": null,
                "cidr": null,
                "network_type": null
            },
            {
                "created_at": "2016-06-01T21:12:14.843836",
                "id": "9c5dbe11-cdf6-48a4-b6ca-9582ef5af193",
                "availability_zone": "london",
                "segmentation_id": null,
                "neutron_subnet_id": "53482b62-2c84-4a53-b6ab-30d9d9800d06",
                "updated_at": null,
                "neutron_net_id": "998b42ee-2cee-4d36-8b95-67b5ca1f2109",
                "ip_version": null,
                "cidr": null,
                "network_type": null
            }
        ]
    }

**Showing details of a subnet**::

    GET /v2/​{tenant_id}​/share-networks/{share_network_id}/subnets/{share_network_subnet_id}

If the share network ID or the share network subnet ID is invalid, the API
will respond with ``404 NotFound``.

Response::

    Code: 200 OK

    {
        "share_network_subnet": {
            "created_at": "2016-06-01T21:12:14.843836",
            "id": "aa7a1269-703b-4832-a3df-17ed954c276c",
            "share_network_id": "77eb3421-4549-4789-ac39-0d5185d68c29",
            "availability_zone": "paris",
            "segmentation_id": null,
            "neutron_subnet_id": "12c1490a-e82c-4f5e-bcb1-fb267a58cf10",
            "updated_at": null,
            "neutron_net_id": "998b42ee-2cee-4d36-8b95-67b5ca1f2109",
            "ip_version": null,
            "cidr": null,
            "network_type": null
        }
    }


**Deleting a share network**::

    DELETE /v2/​{tenant_id}​/share-networks/​{share_network_id}​

A share network with multiple subnets cannot be deleted atomically. The API
will respond with ``409 Conflict``. Users would have to iteratively remove
subnets until one or lesser subnets remain in the share network before
attempting to delete the share network.

Response::

    Code: 202 Accepted



Security impact
---------------

Security services are still associated with share networks, but the
coverage of the share network is increasing to span multiple subnets. So for
an external user, the only visible change is that the share network could now
be "bigger" than it previously was. Hence, due care must be taken in adding
subnets to existing share networks. Users may assign as many subnets to a
particular share network as there are availability zones in a deployment.
They can also define a subnet that can span all availability zones.


Notifications impact
--------------------

We will asynchronously validate the network information during share creation,
and if there is a mismatch with what is specified in the configured network
plugin, we will raise an error user message.

Other end user impact
---------------------

- End users will have to specify an availability zone parameter during
  share network creation unless they would like to create the share network
  with a subnet that would span all availability zones.
- End users would have to add new subnets to an existing network iteratively
  to extend the share network across AZs.


Performance impact
------------------

Validating that users provide the correct storage availability zones will be
performed at the API layer. However, the validation of the availability
zones with respect to their configured networks will be done at the network
plugin layer, as existing network checks are done today. This is not
expected to cause a performance impact, but will keep our existing share
network validation isolated from the API allowing for further enhancements
instead of gating changes at the API layer.

In other words, to minimize the performance impact, no further validation of
network information at the API is recommended by the design introduced in
this spec.


Other deployer impact
---------------------

* No new configuration options are expected to be added. However, neutron
  performs AZ validation when users create networks with
  availability_zone_hints. Deployers must ensure that neutron services are
  running in the desired availability_zones to allow for the network creation
  to succeed.
* All existing share networks will have their subnet (if existing) designated
  as ``default`` after the database upgrade to this version of the
  database changes.

Developer impact
----------------

None

Driver impact
-------------

None. Drivers must be agnostic to all these changes since they will be
handled at the manila API service and the share manager services.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  | lseki


Work Items
----------

* Introduce share network subnets in the database and map them to
  availability zones.
* Changes to share network APIs - create, list, modify
* CLI implementation
* manila-ui support

Dependencies
============

The evolution of share replication to cover the ``DHSS = True`` use case is
dependant on this change. Once this work is completed, share drivers that
support ``DHSS = True`` mode can support share replication. As a
pre-requisite correction, the Share replica API should no longer support
specifying the ``share_network_id``. [5]

Testing
=======

Unit test coverage will be added/maintained as per community standards.
Tempest tests will be modified/added to cover new share network API changes.
AZ awareness will be coded into the tests, with fall-backs like the share
replication tempest tests.

Setting up multiple AZs on the gate is not in the scope of this work.
However, these tests can be run manually with multi-AZ configurations.


Documentation Impact
====================

The following OpenStack documentation will be updated to reflect this change:

* OpenStack User Guide: will document the changes in creation and
  modification of share networks.
* OpenStack Admin Guide: will document the APIs to list/delete share
  network instances.
* OpenStack API Reference: All API changes will be documented
* Manila Developer Reference: the low level implementation considerations
  and design of this feature will be documented.
* OpenStack Security Guide: Readers will be made aware of the high level
  picture of adding AZ awareness to share networks.

References
==========

[1]: http://docs.openstack.org/admin-guide/shared_file_systems_share_networks.html

[2]: https://wiki.openstack.org/wiki/Manila/design/manila-mitaka-data-replication

[3]: http://specs.openstack.org/openstack/neutron-specs/specs/liberty/availability-zone.html

[4]: http://docs.openstack.org/mitaka/networking-guide/adv-config-availability-zone.html

[5]: https://bugs.launchpad.net/manila/+bug/1588144
