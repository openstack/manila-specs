..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================
Share servers with multiple subnets
===================================

https://blueprints.launchpad.net/manila/+spec/multiple-subnet-share-servers

This spec proposes changes to Manila with the purpose of addressing the need
for having share servers with network allocations on multiple subnets. To do
so, the spec is proposing allowing share network to span multiple subnets
in the same AZ.

Problem description
===================

One of two modes that a manila driver can operate in,
``driver_handles_share_servers = True``, requires a share server to be
provisioned in a neutron subnet that can be either IPv4 or IPv6. The share
server provisioned there has its shares accessible from hosts connected to same
subnet. When IPv6 support was added to Manila, it allowed for new use cases and
some limitations became apparent, namely that a share server could not serve
shares with IPv4 and IPv6 export locations at the same time.

In a world that has transitioning infrastructure from IPv4 to IPv6, it is no
longer unusual to expect connectivity in both address families. The fundamental
concept of file services is to have data shared across multiple hosts, thus it
is sensible to expect shares to be accessible from both IPv4 and IPv6 address
families simultaneously as well, as storage back ends are capable of
accomplishing that.

Manila drivers implement support for share servers by receiving a list of
network allocations that the back end should create. The amount of network
allocations supplied varies by driver, as each driver reports the amount it
needs before the allocation definitions are obtained from Neutron.

Prior to the implementation of `[1]`_, the share network entity held
information from Neutron networks and subnets, and Manila referred to its data
when trying to dynamically allocate addresses. Since the implementation of
`[1]`_, the share network entity was modified to do not hold Neutron
information, it was instead modified to span multiple share network subnets in
different availability zones. Since then, share servers are created and
associated with a single share network subnet, which contains neutron network
and subnet information.

A share network subnet is created by the cloud-user and its neutron network and
subnet associations are also defined at creation time. No driver implementation
was required to handle more than one subnet. The result is that, if more than
the expected amount of allocations are provided, all drivers as currently
implemented would fail to handle the additional allocations, which would be
needed to meet the requirements for serving a share in both IP versions.

Since the implementation of `[2]`_ , Manila has the ability to change the share
networks' security services even if such networks were being used by share
servers. The subnet associations still cannot be changed, which could
result in scenarios where users end up creating subnets to just serve shares,
while resources reside in other more flexible networks, having traffic routed
between them with reduced performance.

Use Cases
=========

By allowing share servers to be provisioned in more than one subnet, the
following use cases can be addressed:

* Ability to serve shares with both IPv4 and IPv6 export locations
  simultaneously.

* Ability to serve shares in multiple subnets. In an overloaded
  environment, the Neutron subnet may be running out of IPs (to assign to
  clients), requiring a new subnet to client access, instead of:

  * Having to route data between them.

  * Having to create one share network for each subnet, which will result in
    spawning new share servers each with their own shares.

* Update network allocations of existing share server by adding share network
  neutron subnet associations.


Proposed change
===============

New boolean driver capabilities will be introduced:

* **``share_server_multiple_subnet_support``**: determines whether the driver
  supports share server configuration on multiple subnets. If so, this
  capability is reported as ``True``. Otherwise, the value is ``False``.
* **``network_allocation_update_support``**: determines whether the driver
  implements a new interfaces ``update_share_server_network_allocations`` and
  ``check_update_share_server_network_allocations``. If so, this capability is
  reported as ``True``. Otherwise, the value is ``False``.


The purpose of the new capability ``share_server_multiple_subnet_support``
is so that the scheduler can determine the proper back end when creating
new shares in share networks that already have more than one subnet per AZ.

The new driver interface ``update_share_server_network_allocations`` enables
drivers update share server network allocations whenever neutron subnets are
added from share networks. Also, the interface must return the new share's
export locations, since they will change together with share server subnet
change.

Before calling the update driver interface, the Manila will call the
``check_update_share_server_network_allocations`` to check that the
update is available across all drivers that have servers in the share network.
If one of the drivers deny the update, the entire update operation will fail.
In this case, no resource goes to error state, only logging why the operation
was denied.

A new boolean column ``network_allocation_update_support`` will be added to the
share server model to distinguish existing share servers that can support the
update functionality. It will inherit from its driver capability.

The workflows affected by this proposal are:

* When adding share network subnets to existing share network that results in
  more than one subnet per AZ (or more than one ``default`` subnet), the
  restriction of one subnet per AZ will only exist if there are any existing
  associated share servers that have ``network_allocation_update_support``
  field set to ``False``. In such case, the API call will fail with
  ``409 Conflict``. Otherwise, the API call will succeed and the share manager
  will be invoked to update the share server network allocations.

* When creating shares in share networks with more than one subnet per AZ, the
  scheduler will attempt to find a valid back end that has
  ``share_server_multiple_subnet_support`` set to ``True``. If that is not
  the case, the request will result in ``No valid host`` scheduling error.

It is possible for a driver to support these capabilities independent of each
other. If a driver reports ``share_server_multiple_subnet_support=True`` and
``network_allocation_update_support=False``, the share network will be
immutable after the first share server has been provisioned on it. So if a
cloud administrator wants both capabilities they must explicitly request each
via share type extra specifications.

Existing share servers whose drivers implement neither interface will continue
to work without changes. The API will never allow their associated share
networks to have neutron subnets added if it results in more than one subnet
per AZ. New shares will be allowed to be scheduled to them only when
the associated share network does not have more than one neutron subnet per AZ.

If a driver decides to implement ``update_share_server_network_allocations`` in
later releases, after the ``network_allocation_update_support`` field has
already been set to either ``True`` or ``False``, the share server's
``network_allocation_update_support`` field will need to be reset using the
``manila-manage`` tool.

Share network updates can be an one-to-many operation, which might trigger
multiple share server updates, for different back ends in different share
services. To add a subnet associated with a share network, the share API
will need to validate if all affected resources are healthy before proceeding
with the request. After that, both share network and share servers status will
be updated to ``network_change``.

If the share server setup using multiple subnet fails to configure the server
for one of the given subnets, the entire setup will fail. Likewise, if the
``update_share_server_network_allocations`` fails, the share server status will
be set to ``error`` along with all affected shares access rules. The network
status won't be affected by errors raised by back end drivers, share network
will be active and its resources will be updated (subnets added).

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

Since there is no way to address the use cases introducing changes in manila
while re-using the same driver interfaces, the only alternative is to not
implement the proposed changes and continue with the limitations.

Data model impact
-----------------

A new ``network_allocation_update_support`` capability column will be added to
``manila.db.sqlalchemy.models.ShareServer`` indicating if the driver and the
back end where this share server resides support the update multiple subnet per
AZ operations. In database migration upgrade, the new column will be
added with a default value set to ``False``, meaning that all share servers
already deployed won't be able to update subnets per AZ configuration
even if the driver supports it. In database migration downgrade the column will
be dropped.

Given that the ``ShareServer`` can have multiple ``ShareNetworkSubnet`` and
the ``ShareNetworkSubnet`` can also be in multiple ``ShareServer``, a new table
``ShareServerShareNetworkSubnetMapping`` will be added. The field
``share_network_subnet_id`` in the ``ShareServer`` will be removed. The
database migration upgrade will build this new table based on the current
``ShareServer`` table and remove the field. Downgrade is not
recommended, though, since the mapping could have more than one entry per
``ShareServer``.

A new ``network_allocation_update_support`` property will be added to
``manila.db.sqlalchemy.models.ShareNetwork`` to indicate whether a share
network supports or not the update multiple subnet per AZ operations.
This property will inherited its value from all current associated share
servers. If all associated share servers support
``network_allocation_update_support``, the share network property will be set
to ``True``, otherwise it will be set to ``False``.

Since the current Manila data model design has the entity
``manila.db.sqlalchemy.models.ShareNetworkSubnet`` containing the foreign key
``availability_zone_id``, not the inverse, nothing is blocking the data model
to have multiple subnets in the same AZ.

REST API impact
---------------

There are no new API introduced, but the changes to the existing APIs will be
microversioned. Namely:

* **Displaying a share server**: after the microversion bump, the share server
  view will include the field ``network_allocation_update_support``. Also, the
  ``share_network_subnet_id`` will be a list of IDs.

* **Adding a neutron subnet to a share network**: after the microversion bump,
  this API will no longer always return an error when the operation result
  would be more than one subnet per AZ. Instead, it will check if the existing
  associated share servers have ``network_allocation_update_support`` field set
  to ``True``. If that is not the case, it will return ``409 Conflict``.

Security impact
---------------

None.

Notifications impact
--------------------

There are no new notifications introduced. The existing notifications already
accommodate the changes proposed.

Other end user impact
---------------------

None other than the restriction changes when adding or removing subnets from
a share network.

Performance impact
------------------

None.

Other deployer impact
---------------------

* No new configurations are expected to be added.
* The back end capability will help deployers to identify pools that support
  multiple subnet share server support.
* All existing share servers will have their
  ``network_allocation_update_support`` set to ``False``, even if the
  driver supports it. New share servers will have the correspondent capability
  set according to the back end capability reported by the drivers.
  Administrators will need to manually update share server's  capability using
  ``manila manage`` commands.

Developer impact
----------------

None.

Driver impact
-------------

Drivers that wish to support the update functionality must implement the new
check and update driver interfaces::

    def check_update_share_server_network_allocations(
        self, network_info, server_details, shares, snapshots):
    """Updates a share server's network allocations.

    :param network_info: Dictionary containing network parameters for share
        server update, with the map of network allocations and security
        services among them.
    :param server_details: Back end details for share server being updated.
    :param shares: All shares in the share server.
    :param snapshots: All snapshots in the share server.
    :return Boolean indicating whether the update is possible or not. It is
        the driver responsibility to log the reason why not accepting the
        update.

    def update_share_server_network_allocations(
        self, network_info, server_details, shares, snapshots):
    """Updates a share server's network allocations.

    :param network_info: Dictionary containing network parameters for share
        server update, with the map of network allocations and security
        services among them.
    :param server_details: Back end details for share server being updated.
    :param shares: All shares in the share server.
    :param snapshots: All snapshots in the share server.
    :return If the update changes the shares export locations or snapshots
            export locations, this method should return a dictionary containing
            a list of share instances and snapshot instances indexed by their
            id's, where each instance should provide a dict with the relevant
            information that need to be updated. Also, the returned dict
            contains the updated back end details to be saved in the database.

            Example::

                {
                    'share_updates':
                    {
                        '4363eb92-23ca-4888-9e24-502387816e2a':
                        {
                        'export_locations':
                        [
                            {
                            'path': '1.2.3.4:/foo',
                            'metadata': {},
                            'is_admin_only': False
                            },
                            {
                            'path': '5.6.7.8:/foo',
                            'metadata': {},
                            'is_admin_only': True
                            },
                        ],
                        'pool_name': 'poolA',
                        },
                    },
                    'snapshot_updates':
                    {
                        'bc4e3b28-0832-4168-b688-67fdc3e9d408':
                        {
                        'provider_location': '/snapshots/foo/bar_1',
                        'export_locations':
                        [
                            {
                            'path': '1.2.3.4:/snapshots/foo/bar_1',
                            'is_admin_only': False,
                            },
                            {
                            'path': '5.6.7.8:/snapshots/foo/bar_1',
                            'is_admin_only': True,
                            },
                        ],
                        },
                        '2e62b7ea-4e30-445f-bc05-fd523ca62941':
                        {
                        'provider_location': '/snapshots/foo/bar_2',
                        'export_locations':
                        [
                            {
                            'path': '1.2.3.4:/snapshots/foo/bar_2',
                            'is_admin_only': False,
                            },
                            {
                            'path': '5.6.7.8:/snapshots/foo/bar_2',
                            'is_admin_only': True,
                            },
                        ],
                        },
                    }
                    'backend_details':
                    {
                        'new_share_server_info_key':
                            'new_share_server_info_value',
                    },
                }
        Example::

            {'server_name': 'my_share_server'}

        """
        raise NotImplementedError()


Drivers should expect to receive multiple network allocations. The total number
will be the number of subnets associated with the same AZ multiplied by the
number of allocations reported by the driver.

The pre-existent driver interface ``_setup_server`` will be modified. The
network allocations dictionary entry ``network_info`` will now be a
list of dictionary, representing allocations in each subnet. To keep the
compatibility, the drivers will be changed to consume the first element of the
list and an email informing this interface change will be sent in the openstack
email list.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  | felipe_rodrigues


Work Items
----------
* Implement core changes that must include:

  * Add share network and share server model attributes;
  * Add database migration.
  * Update Add Share network neutron subnet API validation based on new
    Share Server model field.
  * Add new driver interfaces.
  * Add new capability.
  * Add scheduler capability check.

* Implementation in a first party driver
* Add new functional test in manila-tempest-plugin
* Add new command to manila-manage for share server capability update
* Update manila documentation

This spec may require more than one release to be delivered covering all use
cases. So, it can be splitted in two major deliverables, being:

1. Add ability to define multiple subnets in the same share network AZ
2. Add ability to "update" subnets in a share network AZ

This deliverable approach would split the test and validation effort a long
the releases.

Dependencies
============

None.

Testing
=======

Unit test coverage will be added/maintained as per community standards.
New tempest tests will be added to cover multiple subnets per AZ
scenarios. The container or the dummy driver will be improved to properly
configure security services and be used to validate the proposed changes..

Documentation Impact
====================

The following OpenStack documentation will be updated to reflect this change:

* OpenStack User Guide: will document the changes to Share Networks APIs.

* OpenStack Admin Guide: will document the changes to Share Servers.

* OpenStack API Reference: All API changes will be documented.

* Manila Developer Reference: the low level implementation considerations,
  feature design and guidelines for driver implementation will be documented.


References
==========

_`[1]`:  https://specs.openstack.org/openstack/manila-specs/specs/train/share-network-multiple-subnets.html

_`[2]`: https://specs.openstack.org/openstack/manila-specs/specs/wallaby/security-service-updates-in-use-share-network.html

