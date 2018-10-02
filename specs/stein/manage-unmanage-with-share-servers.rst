..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
Manage / Unmanage with Share Servers
====================================

https://blueprints.launchpad.net/manila/+spec/manage-unmanage-with-share-servers

This spec proposes enhancement to manila's Manage/Unmanage functionality so
that drivers running in ``DHSS=True`` mode can:

* import existing shares and snapshots, bringing them under manila's
  management.
* release shares and their snapshots from manila's management without
  destroying them.


Problem description
===================

Cloud administrators cannot bring pre-existing shares under manila's management
when those shares pertain to a back end driver operating in
``driver_handles_share_servers = True`` mode `[1]`_. For brevity, we will refer
to this mode as ``DHSS = True`` in this document. Since ``DHSS = True`` is the
mode in which manila guarantees secure multi-tenant isolation, cloud-users end
up having to make an unfortunate tradeoff, using their imported shares in
``DHSS = False`` mode.

We lack manage/unmanage for ``DHSS=True`` mode drivers today because of the
following complexities:

* **Share server setup and networking:** In ``DHSS = True`` mode, manila
  provisions and manages the lifecycle of share servers for each of the
  tenant's networks. When creating the share servers, manila has to allocate
  network ports (and later de-allocate them when share servers are deleted).
  However, in ``DHSS = False`` mode the concept of share servers do not apply,
  it is up to the administrator and the driver to do whatever is necessary for
  the back end's export locations to be accessible from the client hosts.

* **Managing shares:** In ``DHSS = False``, manila requests the driver to find
  a pre-existing share in the configured back end based on an export location
  given at the API call. Shares being managed are expected to be accessible
  from client hosts as any share created from manila would. In ``DHSS = True``
  mode, any existing share would be in a share server, therefore to manage a
  share, one would have to manage its share server first and then manage the
  share within the share server.

* **Managing snapshots:** Since currently there is no way to manage a share
  in manila in ``DHSS = True`` mode, there is no way to manage snapshots as
  well. Additionally, unmanaging of snapshots is currently disallowed.

* **Lifecycle of a share server:** In ``DHSS = True`` mode, manila provisions
  the share servers and network resources associated with it, therefore it also
  deletes them when appropriate (like when the share server is not serving any
  shares). In ``DHSS = False``, manila only manages the lifecycle of the
  resources it creates, such as shares, snapshots, replicas and access rules.

As per the aspects mentioned previously, it becomes clear that to manage a
share or snapshot in ``DHSS = True`` mode, one needs an API to manage a share
server first.


Use Cases
=========

Cloud administrators that have been using storage devices to provision shares
and their snapshots should have the ability to import the existing shares when
migrating over to manila. Manila drivers that support ``DHSS = True`` mode
could accomplish that if there was such API implemented in Manila.

Similarly, manila shares and snapshots could be unmanaged in ``DHSS = True``
mode to be migrated to another system or to have maintenance performed, but
this would only make sense if they could be managed back.


Proposed change
===============

The following changes are proposed:

* **Add a Manage Share Server API:** Through this API, share servers shall be
  managed. Their state transitions will be similar to when managing shares. In
  other words, they will be created with the status ``manage_starting`` and
  transition either to ``available`` in case of success or to ``manage_error``
  otherwise. The following parameters are expected to be supplied:

  * **Host:** back end name ("<node_hostname>@<backend_stanza>").
  * **Share Network:** share network associated with the neutron network the
    share server is connected to.
  * **Identifier:** a driver-specific share server identifier required by the
    driver to manage the share server.
  * **Driver Options:** optional list of driver-specific key-value pairs that
    may be necessary to assist the driver managing the share server.

  * **Network allocations pre-requisites:** Since connectivity is expected
    between an existing share server and the client hosts, the share server
    will already have interfaces with MAC and IP addresses but neutron and
    manila know nothing of them. Before taking the share server into manila's
    management, the cloud administrator must create neutron ports corresponding
    to these share server interfaces, allocate the proper addresses to these
    ports, and set each of the port's ``device_owner`` to ``manila:share``.
    When managing the share server, the share server IP addresses will be
    requested by the driver and once retrieved, matched with neutron ports
    owned by ``manila:share`` present in the neutron subnet associated with the
    share network provided or configured network plugin. If not all allocations
    are found, the operation is aborted with an error. When using the
    Standalone Network Plugin, this step is not required since Neutron is not
    involved.

  * **Security services pre-requisites:** Any pre-existing authentication
    services set up with share servers to be managed must be configured as
    security services associated with the share network before managing a share
    server. Manila offers no capability to update security services on existing
    manila provisioned share servers today. This specification does not add
    an ability to update security services on managed share servers.

  * **Implementation:** When a request to manage a share server is received,
    the API parameters are validated and then a share server model with the
    ``is_imported`` field set is created. The share service that runs the
    back end stanza specified in the host specified is invoked through RPC, and
    the driver method ``manage_server`` is invoked to obtain the back end
    details and network allocations of the share server to be managed. The
    back end details returned are saved in the database and the network
    allocations retrieved are then passed to the configured network plugins to
    validate the network allocations previously created by the admin. After the
    validation, the network allocation is saved in manila's
    ``network_allocations`` database table.

* **Add a Unmanage Share Server API:** Through this API, share servers shall be
  unmanaged. No parameters are required beyond the ID of the share server to be
  unmanaged. The share server specified is removed from manila database. We are
  not going to remove the allocations, as for any existing untracked resource
  in a network, it will be bound to result in conflicts in the future if the
  ports are de-allocated. It will be up to the administrator to de-allocate the
  ports if desired. The state transitions will be similar to unmanaging a
  share, thus it would transition from ``available`` to ``unmanage_starting``,
  then either to ``deleted`` in case of success or to ``unmanage_error``
  otherwise.

  * **Implementation:** When a request to unmanage a share server is received,
    the API parameters are validated and then the share service responsible by
    the given share server is invoked through RPC. The driver is then invoked
    to perform any operation that may be necessary to proceed with unmanaging,
    and finally the network allocations are deleted from manila's
    ``network_allocations`` database table.

* **Add share_server_id parameter to Manage Share API:** Whenever managing a
  share and passing a share type defined with ``driver_handles_share_servers``
  set to ``True``, the parameter share_server_id will be required, else the API
  will return ``400 BadRequest``. The Manage Snapshot API does not require this
  parameter since it will read it from the parent share's model.

* **Allow unmanaging of shares and snapshots in ``DHSS = True`` mode:**
  Currently it is not allowed to unmanage shares and snapshots that were
  created in this mode. We will change the API to allow it (and thus no longer
  return an error) when the newer microversion is specified. There are no
  behavioral changes required other than this one at the API layer.

* **Update driver interface of Manage/Unmanage Share and Snapshot to pass
  the share server:** Since the existing implementation has never expected to
  work in ``DHSS = True`` driver mode, the driver interfaces do not include a
  share server parameter. The driver interfaces will be updated to receive the
  share server model to perform for manage and unmanage operations of shares
  and snapshots. The parameter is optional, so it will not affect existing
  ``DHSS = False`` driver implementations in any way.

* **Prevent automatic deletion of managed share servers:** As opposed to share
  servers created by manila, we will not attempt to automatically delete
  managed share servers that have no manila shares, as there may be existing
  shares unknown to manila within it.

* **Manual deletion of managed share servers:** Share servers managed by manila
  may contain existing shares and not all of those shares may be managed by
  manila. If admins decide to delete the share server, it will be up to the
  admin to try to delete it along with any existing shares that are unknown to
  manila. However, it will be up to the driver to allow the operation to
  succeed, as some back ends do not allow the share server to be deleted if
  there are remaining shares. If that is the case, the share server will go to
  error state and the admin will have to either unmanage it, or delete the
  remaining shares (that are unknown to manila) either manually in the back end
  or by managing them.

Alternatives
------------

Instead of adding the Manage Share Server API, a previously discussed approach
was a Manage Share API that would have two phases in ``DHSS = True`` mode. Such
approach presents the following characteristics:

* Same complexity and amount of technical work as the proposed solution, but
  done by a single API.
* Several additional parameters that make sense only for ``DHSS = True`` mode.
* More complex error handling, as there would need to be statuses and errors
  specific to each phase (managing a share server or managing a share within).
* The user experience degrades when we have a single API behaving so
  differently, as the user can get confused when using it on both driver modes.

Alternatively to requiring the neutron ports to be created by the admin before
managing a share server, we could have manila create the network allocations
when managing the share servers, as this would be easier for the admin when
doing a "bulk" manage of share servers. The downsides to this approach are:

* It is reasonable to assume that the share servers we are managing are already
  connected to the share networks and accessible by hosts. Therefore, it is
  correct to also assume that a port in neutron should already exist to prevent
  the network allocations of those share servers from conflicting with other
  resources in the same subnet.

* When unmanaging a share server, we would not remove the network allocations,
  since it is assumed these share servers may remain connected to the share
  networks and would need those network allocations to prevent conflicts with
  other resources in the same subnet. This behavior would be asymmetrical to
  creating the ports when managing share servers.

If resorting to not change the APIs, there is no way to import existing
resources to be managed by manila. If this is not implemented, the
administrators would need to manage their storage devices outside of manila, or
accept the limitation of using only ``DHSS = False`` mode, lacking the benefits
present in the ``DHSS = True``.


Data model impact
-----------------

Since there is the need to distinguish share servers created by manila from
managed ones, we proposed the addition of a boolean column in the
``ShareServer`` table named ``is_imported`` defaulting to ``False``. The
database schema upgrade will add the column with the value ``False`` for all
existing share servers, while the database schema downgrade will remove the
column.


REST API impact
---------------

There are a couple of new APIs introduced and a few others changed. The policy
for both new APIs are admin-only like the existing manage/unmanage APIs. There
are no changes for the existing policies. The API microversion will be bumped
to the next one for the listed changes.

**Managing a share server**::

    POST /v2/{tenant-id}/share-servers/manage

Request parameters::

        {
            "share_server": {
                "host": "host@backend",
                "share_network_id": "e76be4e9-4054-4df3-9e5c-178e68fb0949",
                "identifier": "0e73a5e1-e233-4635-b6df-db568307385f",
                "driver_options": {
                    "key1": "value1",
                    "key2": "value2",
                    "key3": "value3"
                }
            }
        }

Parameter ``driver_options`` is optional. If any of the other parameters
is missing or invalid, the API will return ``400 BadRequest``.

Response::

        Code: 202 Accepted

        {
            "share_server":
                "status": "manage_starting",
                "created_at": 2018-09-17T18:05:34.000000,
                "updated_at": 2018-09-17T18:05:34.000000,
                "share_network_id": "e76be4e9-4054-4df3-9e5c-178e68fb0949",
                "share_network_name": "my_share_net",
                "host": "host@backend",
                "project_id": "0ebbe03068554da9b9d9ad11983bb08a",
                "id": "fa4e4d78-d3d9-46cf-8e61-514da5008cee",
                "is_imported": "True",
                "backend_details": {
                    "key1": "value1",
                    "key2": "value2",
                    "key3": "value3"
                }
            }
        }


**Unmanaging a share server**::

    POST /v2/{tenant-id}/share-servers/{share_server_id}/action

Request parameters::

        {
            "unmanage": null
        }

Response::

        Code: 202 Accepted

* If the share server does not exist the API will return ``404 NotFound``.
* If the share server status is not in ``error``, ``active``, ``inactive``,
  ``manage_error`` or ``unmanage_error``, the API will return
  ``400 BadRequest``.
* If the share server has shares registered in manila, it will return
  ``409 Conflict``.


**Displaying a share server**:


After the microversion bump, the share server view will include the field
``is_imported`` whenever a newer microversion is used.


**Managing a share**::

    POST /v2/{tenant-id}/shares/manage

Request parameters::

    {
        "share": {
            "protocol": "NFS",
            "name": "my_share",
            "share_type": "my_type",
            "description": null,
            "driver_options": {},
            "is_public": false,
            "service_host": "host@backend#pool",
            "export_path": "192.168.10.100/my_export",
            "share_server_id": "fa4e4d78-d3d9-46cf-8e61-514da5008cee"
        }
    }

The new parameter ``share_server_id`` is required if the share type given
specifies ``DHSS = True`` mode. If ``share_server_id``is not given, or is given
while the given share type specifies ``DHSS = False`` mode, the API will return
``400 BadRequest``.

Response::

        Code: 202 Accepted

There are no changes proposed to the response body.


**Unmanaging a share**::

    POST /v2/{tenant-id}/shares/{share_id}/action

Request parameters::

    {
        "unmanage": null
    }

Response::

        Code: 202 Accepted


This API will no longer return ``403 Forbiddden`` when attempting to unmanage a
share that was created in ``DHSS = True`` mode.

**Managing a snapshot**:


No API changes are required. The API currently does not validate if the
snapshot being managed is associated with a share that was created in
``DHSS = True`` mode.


**Unmanaging a snapshot**::

    POST /v2/{tenant-id}/snapshots/{snapshot_id}/action

Request parameters::

    {
        "unmanage": null
    }

Response::

        Code: 202 Accepted


This API will no longer return ``403 Forbiddden`` when attempting to unmanage a
snapshot of a share that was created in ``DHSS = True`` mode.


Driver impact
-------------

A new driver interface is introduced in order to get share server network
allocation data. This data shall be used to validate the network allocations
previously created by the admin. Drivers that support ``DHSS = True`` mode
must implement this interface to support the Manage Share Servers
functionality::

    def manage_server(self, context, share_server, identifier, driver_options):
        """Return compiled back end details and network allocations.

        :param context: Current context.
        :param share_server: Share server model.
        :param identifier: A driver-specific share server identifier
        :param driver-options: Dictionary of driver options to assist managing
            the share server
        :return Dictionary with back end details to be saved in the database
            and a list containing IP addresses allocated in the back end.

        Example::

            {'server_name': 'my_old_server'},['192.168.10.10', 'fd11::2000']

        """
        raise NotImplementedError()

If the driver does not implement this interface, an exception will be raised
and the operation will be aborted.

The driver interfaces **manage_existing**, **unmanage**,
**manage_existing_snapshot** and **unmanage_snapshot** will be updated to
receive the share server model parameter. All drivers which implement those
interface will have their method definition updated to avoid issues. In
detail::

    def manage_existing(self, share, driver_options, share_server=None):
        """Brings an existing share under Manila management.

        If the provided share is not valid, then raise a
        ManageInvalidShare exception, specifying a reason for the failure.

        If the provided share is not in a state that can be managed, such as
        being replicated on the backend, the driver *MUST* raise
        ManageInvalidShare exception with an appropriate message.

        The share has a share_type, and the driver can inspect that and
        compare against the properties of the referenced backend share.
        If they are incompatible, raise a
        ManageExistingShareTypeMismatch, specifying a reason for the failure.

        :param share: Share model
        :param driver_options: Driver-specific options provided by admin.
        :param share_server: Share server model or None.
        :return: share_update dictionary with required key 'size',
                 which should contain size of the share.
        """
        raise NotImplementedError()

    def unmanage(self, share, share_server=None):
        """Removes the specified share from Manila management.

        Does not delete the underlying backend share.

        For most drivers, this will not need to do anything.  However, some
        drivers might use this call as an opportunity to clean up any
        Manila-specific configuration that they have associated with the
        backend share.

        If provided share cannot be unmanaged, then raise an
        UnmanageInvalidShare exception, specifying a reason for the failure.
        """

    def manage_existing_snapshot(self, snapshot, driver_options,
                                 share_server=None):
        """Brings an existing snapshot under Manila management.

        If provided snapshot is not valid, then raise a
        ManageInvalidShareSnapshot exception, specifying a reason for
        the failure.

        :param snapshot: ShareSnapshotInstance model with ShareSnapshot data.

        Example::
            {
            'id': <instance id>,
            'snapshot_id': < snapshot id>,
            'provider_location': <location>,
            ...
            }

        :param driver_options: Optional driver-specific options provided
            by admin.

        Example::

            {
            'key': 'value',
            ...
            }

        :param share_server: Share server model or None.
        :return: model_update dictionary with required key 'size',
            which should contain size of the share snapshot, and key
            'export_locations' containing a list of export locations, if
            snapshots can be mounted.
        """
        raise NotImplementedError()

    def unmanage_snapshot(self, snapshot, share_server=None):
        """Removes the specified snapshot from Manila management.

        Does not delete the underlying backend share snapshot.

        For most drivers, this will not need to do anything.  However, some
        drivers might use this call as an opportunity to clean up any
        Manila-specific configuration that they have associated with the
        backend share snapshot.

        If provided share snapshot cannot be unmanaged, then raise an
        UnmanageInvalidShareSnapshot exception, specifying a reason for
        the failure.
        """


Security impact
---------------

As admin-only APIs, all the information necessary for managing share servers
and the shares within is restricted to the admin by virtue of default policy.

Notifications impact
--------------------

Since the new API proposed is an admin-only API, there is no need for a user
message notification.


Other end user impact
---------------------

New commands will be added to python-manilaclient::

  manila share-server-manage <host> <share_network_id> <identifier> [--driver_options key1=value1 [key2=value2] ...]

  manila share-server-unmanage <share_server_id>

One command will be updated::

  manila manage [--name <name>] [--description <description>]
                [--share_type <share-type>]
                [--share_server_id <share-server-id>]
                [--driver_options [<key=value> [<key=value> ...]]]
                [--public]
                <service_host> <protocol> <export_path>


As for manila-ui, there will be a new button "Manage Share Server" and a
context option "Unmanage Share Server" which will only be displayed if there
are no manila shares associated with the given share server.


Performance Impact
------------------

No significant performance impact is expected.

Other deployer impact
---------------------

None.

Developer impact
----------------

The work proposed by this spec is impacted by [2]. The ``share_network_id``
parameter for the Manage Share Server will need to be replaced in favor of
``share_network_subnet_id``.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  ganso


Work Items
----------

* Implement main patch for manila that includes:

  * Database schema migration
  * Manage Share Server API
  * Unmanage Share Server API
  * Updates to Manage Share, Unmanage Share, Manage Snapshot and Unmanage
    Snapshot APIs
  * Add ``manage_server`` driver interface
  * Update existing affected driver interfaces
  * Share Manager adjustments to prevent automatic deletion of managed share
    servers
  * Adaptations to network plugins

* Implementation in a First Party Driver

* Functional Tests in manila-tempest-plugin

* Python-manilaclient update
* Docs update
* Manila-UI update


Dependencies
============

None.

Testing
=======

The functional tests of this change will consist of creating a share (which
will create a regular share server), obtain its details, unmanage it, manage
it, create another share in it, and then delete the share and the server.

The existing config option "run_manage_unmanage_tests", now in combination with
"multitenancy_enabled" will control whether those tests will run. If any of
those are disabled the tests will be skipped.


Documentation Impact
====================

The following documentation sections will be updated:

* API reference: Will add the Manage Share Server API information and parameter
  details. The Manage Share API will be updated to include the
  ``share_server_id`` parameter as well.

* Admin reference: Will add instructions on how about to manage shares in
  ``DHSS = True`` mode (including pre-requisite steps) and how to the use the
  new/updated CLI commands.

* Developer reference: Will add information on how the functionality works and
  how to implement support for it in drivers.


References
==========

_`[1]` https://docs.openstack.org/manila/latest/contributor/driver_requirements.html#at-least-one-driver-mode-dhss-true-false

_`[2]` https://blueprints.launchpad.net/manila/+spec/share-replication-enhancements-for-dhss

_`[3]` https://etherpad.openstack.org/p/manila-ptg-planning-denver-2018
