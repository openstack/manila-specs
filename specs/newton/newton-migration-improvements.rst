..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================
Newton Share Migration Improvements
===================================

https://blueprints.launchpad.net/manila/+spec/newton-migration-improvements

Share migration is a feature that allows an administrator to move a share
across backends. Ideally, the data moved should be exactly the same as before.
This operation is expected to be disruptive in most cases, because if data is
moved to another place or backend, the export location may need to change, thus
the client may need to re-mount the share. In order to handle this
disruptiveness, this spec proposes a 2-phase migration approach.

Problem description
===================

Whenever a share is created on a backend, it cannot be moved to another backend
in the use cases scenarios. The administrators would have to do it manually by
creating an empty share in the destination backend, mounting both shares,
copying data, and handling all possible difficulties by themselves,
while also being inefficient, because in this approach it would download and
re-upload all the data being copied. Since several administrators from several
cloud environments are prone to facing these scenarios, it justifies having a
feature that performs this in a common way.

Use Cases
=========

There are several scenarios for which a share may need to be migrated:

Administrator-oriented

* Maintenance/Evacuation

  * Evacuate a backend for hardware/software upgrades
  * Evacuate a backend experiencing failures
  * Evacuate a backend which is EOL

* Optimization

  * Defragment backends to create empty ones which can be taken offline to
    conserve power
  * Rebalance backends to maximize available performance
  * Move data and compute closer together to reduce network utilization and
    decrease latency/increase bandwidth

* True migration

  * Migrate from old hardware generation to a newer generation
  * Migrate from one vendor to another

User-oriented (through another feature, such as share-modify or share groups)

* Change share type
* Change AZ
* Change share protocol
* Change share network (for DHSS=true) shares
* Change share group
* Expand share when there's no available space

Proposed change
===============

This spec aims to re-evaluate the Share Migration design, including the
improvements discussed at Austin Summit 2016.

There are two possible ways of doing migration:

1) Driver understands destination backend and is able to move at backend level.

2) Manila migrates the share's data to another share created in the destination
   backend, using the Data Service.

In (1), the backend may use different mechanisms such as replication or
snapshots, may not require share to be mounted, may be able to do so without
changing the share to read-only, changing the export location and may also not
be disruptive. When the operation is completed, it should return a new list of
export locations to update the database, when necessary. This approach is
referred to as driver-assisted approach.

In (2), share must be changed to read-only while we are using non-incremental
copy approach, which is expected to cause downtime to tenant applications. This
approach may not be able to preserve file metadata and is also referred to as
host-assisted approach.

It is very important to note that when migrating a share from a backend to
another, the destination backend needs to support exactly the same share
protocol as the source share's. Migration itself does not change share
protocols, another feature called "Share Modify" that is yet to be implemented
should be responsible for doing this operation and may use share migration
feature to do so.

Some attributes such as 'Share Network', 'Share Type' and 'Availability Zone'
may be modified through migration, the administrator can do so through the API.

Since migration is expected to be disruptive, both if performed by the driver
(as in #1 above) or if performed by manila code (as in #2 above), it was
designed with a 2-phase possibility in mind. When invoking "migrationg-start",
a share will be migrated but will pause when the 1st phase is completed
without any disruptiveness, so the administrator can prepare for when to
invoke "migration-complete" to finish the migration, which may be disruptive.

Alternatives
------------

Do not have the feature implemented and administrator will need to do this
manually and less efficiently. Multiple cloud providers would need to create
scripts for doing this and handle each failure scenario themselves. This
spec proposes a common way of doing this.

Data model impact
-----------------

In order to track share migration's operations, support 2-phase migration and
properly handle errors, a field in the database is required. An additional
field in the "Share" table is able to meet these requirements until the Jobs
table (see [1]) is implemented. The field, named "task_state", works in a
similar way as a status and it can be reset via API "reset-task-state".

In order to have change types during a migration, at a certain point, a share
needs to have two instances with different types. At the moment, this is not
possible due to the 'share_type_id' field being within the 'Shares' table in
the database. So this spec includes a database migration to move the
'share_type_id' field to the 'ShareInstances' table.

REST API impact
---------------

Five admin-only new API methods are introduced:

1) (POST, 202, 400, 409) migration-start: migrates share

URL: /shares/<share_id>/action

Body::

  {
    'migration_start': {
      'force_host_assisted_migration': false,
      'preserve_metadata': true,
      'writable': true,
      'nondisruptive': true,
      'host': 'ubuntu@generic2#GENERIC2',
      'new_share_type_id': 'foo_share_type_id',
      'new_share_network_id': 'bar_share_network_id'
    }
  }

2) (POST, 202, 400) migration-complete: triggers 2nd phase of migration

URL: /shares/<share_id>/action

Body::

  {"migration_complete": {}}

3) (POST, 200, 400) migration-get-progress: attempts to obtain migration
   progress

URL: /shares/<share_id>/action

Body::

  {"migration_get_progress": {}}

Example response::

  RESP BODY: {
    "task_state": "data_copying_in_progress",
    "total_progress": 50,
  }

4) (POST, 202, 400) migration-cancel: attempts to cancel migration

URL: /shares/<share_id>/action

Body::

  {"migration_cancel": {}}

5) (POST, 202, 400) reset-task-state: reset task state field value to desired
   one

URL: /shares/<share_id>/action

Body::

  {"reset_task_state": {"task_state": "migration_error}}

API details:

1) ``migration-start [--force-host-assisted-migration <True/False>]
   [--preserve-metadata <True/False>] [--writable <True/False>]
   [--non-disruptive <True/False>] [--new-share-type <new_share_type>]
   [--new-share-network <new_share_network>] <share> <host@backend#pool>``

:force-host-assisted-migration (defaults False): forces the host-assisted
 approach to be used, thus using the Data Service to move copy data across
 backends. This skips the driver-assisted approach which would otherwise be run
 attempted first.

:preserve-metadata (defaults True): whether migration should enforce the
 preservation of metadata. If set to True, this will prevent host-assisted
 migration from running. Drivers are queried to validate this capability, and
 if not capable, driver-assisted approach will be skipped and migration will
 fail.

:writable (defaults to True): whether migration should only be performed if
 share remains writable. If set to True, this will prevent host-assisted
 migration from running. Drivers are queried to validate this capability, and
 if not capable, driver-assisted approach will be skipped and migration will
 fail.

:non-disruptive (defaults to False): whether migration should only be performed
 if share access is not disrupted during migration. For such, it is also
 expected that the export location does not change. If set to True, this will
 prevent host-assisted migration from running. Drivers are queried to validate
 this capability, and if not capable, driver-assisted approach will be skipped
 and migration will fail.

:new-share-type (defaults to None): the new share type that should be set in
 the migrated share.

:new-share-network (defaults to None): the new share network that should be set
 in the migrated share.

:share: share to be moved.

:host@backend#pool: string that combines host@backend#pool combination where
 share should be migrated to.

2) ``migration-complete <share>``

:share: share on which migration should be completed. Share must be in
 host-assisted's "copy completed" or driver-assisted's
 "driver phase 1 completed" task state to have its phase 2 migration invoked.

3) ``migration-get-progress <share>``

:share: share from which migration progress should be obtained. The total
 progress is displayed along with the current task state value. If the share is
 not being migrated or the driver cannot obtain progress then an error message
 is returned.

4) ``migration-cancel <share>``

:share: share from which migration should be cancelled. Share must be in
 host-assisted's "copy completed" or driver-assisted's
 "driver phase 1 completed" task state to be cancellable.

5) ``reset-task-state [--task-state <state>] <share>``

:task-state (defaults to None): value to reset the task state field to.

:share: share from which the task state field should be reset to the value
 provided.

Driver impact
-------------

Vendors can implement the driver-assisted migration in their drivers in order
to migrate data efficiently across backends from the same vendor or using
vendor-compatible protocols.

In order to support host-assisted migration, existing drivers which run in
DHSS=False mode should not need to implement any additional code, while code to
handle their share protocol should be present in the Data Service and the
networks need to be manually set up. However, it is highly recommend to support
admin network to require less network configuration effort.

For DHSS=True mode drivers, if existing drivers do not have admin network
support to allow connectivity between shares and nodes in admin network, the
host-assisted approach will not work.

Add driver interfaces::

    def migration_check_compatibility(
            self, context, source_share, destination_share,
            share_server=None, destination_share_server=None):
        """Checks destination compatibility for migration of a given share.

        .. note::
            Is called to test compatibility with destination backend.

        Driver should check if it is compatible with destination backend so
        driver-assisted migration can proceed.

        :param context: The 'context.RequestContext' object for the request.
        :param source_share: Reference to the share to be migrated.
        :param destination_share: Reference to the share model to be used by
            migrated share.
        :param share_server: Share server model or None.
        :param destination_share_server: Destination Share server model or
            None.
        :return: A dictionary containing values indicating if destination
            backend is compatible, if share can remain writable during
            migration, if it can preserve all file metadata and if it can
            perform migration of given share non-disruptively.

            Example::

            {
                'compatible': True,
                'writable': True,
                'preserve_metadata': True,
                'nondisruptive': True,
            }
        """
        return {
            'compatible': False,
            'writable': False,
            'preserve_metadata': False,
            'nondisruptive': False,
        }

    def migration_start(
            self, context, source_share, destination_share,
            share_server=None, destination_share_server=None):
        """Starts migration of a given share to another host.

        .. note::
           Is called in source share's backend to start migration.

        Driver should implement this method if willing to perform migration
        in a driver-assisted way, useful for when source share's backend driver
        is compatible with destination backend driver. This method should
        start the migration procedure in the backend and end. Following steps
        should be done in 'migration_continue'.

        :param context: The 'context.RequestContext' object for the request.
        :param source_share: Reference to the original share model.
        :param destination_share: Reference to the share model to be used by
            migrated share.
        :param share_server: Share server model or None.
        :param destination_share_server: Destination Share server model or
            None.
        """
        raise NotImplementedError()

    def migration_continue(
            self, context, source_share, destination_share,
            share_server=None, destination_share_server=None):
        """Continues migration of a given share to another host.

        .. note::
            Is called in source share's backend to continue migration.

        Driver should implement this method to continue monitor the migration
        progress in storage and perform following steps until 1st phase is
        completed.

        :param context: The 'context.RequestContext' object for the request.
        :param source_share: Reference to the original share model.
        :param destination_share: Reference to the share model to be used by
            migrated share.
        :param share_server: Share server model or None.
        :param destination_share_server: Destination Share server model or
            None.
        :return: Boolean value to indicate if 1st phase is finished.
        """
        raise NotImplementedError()

    def migration_complete(
            self, context, source_share, destination_share,
            share_server=None, destination_share_server=None):
        """Completes migration of a given share to another host.

        .. note::
            Is called in source share's backend to complete migration.

        If driver is implementing 2-phase migration, this method should
        perform the disruptive tasks related to the 2nd phase of migration,
        thus completing it. Driver should also delete all original share data
        from source backend.

        :param context: The 'context.RequestContext' object for the request.
        :param source_share: Reference to the original share model.
        :param destination_share: Reference to the share model to be used by
            migrated share.
        :param share_server: Share server model or None.
        :param destination_share_server: Destination Share server model or
            None.
        :return: List of export locations to update the share with.
        """
        raise NotImplementedError()

    def migration_cancel(
            self, context, source_share, destination_share,
            share_server=None, destination_share_server=None):
        """Cancels migration of a given share to another host.

        .. note::
           Is called in source share's backend to cancel migration.

        If possible, driver can implement a way to cancel an in-progress
        migration.

        :param context: The 'context.RequestContext' object for the request.
        :param source_share: Reference to the original share model.
        :param destination_share: Reference to the share model to be used by
            migrated share.
        :param share_server: Share server model or None.
        :param destination_share_server: Destination Share server model or
            None.
        """
        raise NotImplementedError()

    def migration_get_progress(
            self, context, source_share, destination_share,
            share_server=None, destination_share_server=None):
        """Obtains progress of migration of a given share to another host.

        .. note::
            Is called in source share's backend to obtain migration progress.

        If possible, driver can implement a way to return migration progress
        information.
        :param context: The 'context.RequestContext' object for the request.
        :param source_share: Reference to the original share model.
        :param destination_share: Reference to the share model to be used by
            migrated share.
        :param share_server: Share server model or None.
        :param destination_share_server: Destination Share server model or
            None.
        :return: A dictionary with at least 'total_progress' field containing
            the percentage value.
        """
        raise NotImplementedError()

    def connection_get_info(self, context, share, share_server=None):
        """Is called to provide necessary generic migration logic.

        :param context: The 'context.RequestContext' object for the request.
        :param share: Reference to the share being migrated.
        :param share_server: Share server model or None.
        :return: A dictionary with migration information.
        """

        Has a default implementation that can be overridden.

The general approach for a driver-assisted migration is that drivers will be
invoked to analyze compatibility with the destination backend and return a
dictionary containing information that describes whether they are compatible
and which capabilities, such as perserving metadata, remaining writable and
being non-disruptive, are supported. To obtain information to perform this
analysis, drivers are advised to read other related backends data from manila
configuration file. Ideally, they should be talking to each other through RPC
calls, but sensitive data such as passwords should not be included in RPC
responses, so such approach cannot be taken at this moment. At this point, it
is recommend that drivers also test for connectivity with the destination
backend.

If the destination share has a share network ID defined, it is implied that the
share requires a share server, so manila code will send a request to the
destination backend so it can provide a share server, where if one does not
exist, it will be created on the destination backend, invoked by manila.

Then, the migration_start method of the source backend driver is invoked, to
perform migration. This method should start the migration job in the storage
and return. Manila will invoke the method migration_continue according to a
periodic task so the driver can perform subsequent steps to continue migration
until the first phase is completed, in which the driver should return so.

The driver should also make sure that while migration is not completed,
the source share instance must be revertible to and its data intact.

At last, the administrator will invoke migration-complete to perform the last,
possibly disruptive, steps of migration. Drivers should remove the source share
at this moment. Manila will also apply the existing access rules to the
destination instance using update_access driver interface.

Additionally, if the current base driver class implementation for several
methods used by the host-assisted migration is not supported by a driver, the
driver can override those methods adding a special behavior to support the
host-assisted migration approach.

Security impact
---------------

In order to access a share's data, it must be mounted by an entity. The entity
mounting the share is responsible for the data. During migration, if the
host-assisted approach is used, the Data Service will be mounting the migrating
share and copying data, thus it will expose the share's contents to the Data
Service node during a limited period of time. The Data Service is accessible
through the administrator network, thus it grants access to data to whoever is
able to connect to the Data Service. Restricted access to this node is advised.

This change includes new entries to rootwrap permission file, for following
commands that must be run as root:

- ls -pA1 --group-directories-first %s
- touch --reference=%s %s

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

All the copy-related commands are resource-intensive and should be run in a
separate node where only the Data Service is installed, thus not disrupting the
other services.

Other deployer impact
---------------------

New configuration options are introduced. Most have default values, while a few
others require an administrator to input them. In order for the Data Service
to handle mounting shares of several different protocols, it needs to be
configured:

* The node must be set up in the admin network, and the config option
  'data_node_access_ip' must be set with the IP value of this node interface
  that connects it to the admin network. This is enough to mount shares which
  access rules are IP-based.

* Protocol libraries like for NFS and CIFS need to be installed in this node.

* For protocols which access rules are certificate-based, the certificate needs
  to be installed and the config option 'data_node_access_cert' must be set.

* For protocols which access rules are user-based, the user must be configured
  in the node and backend security service as an administrator. The
  username must be set in the 'data_node_access_admin_user' config option and
  the 'data_node_mount_options' config option must be set with the command
  parameters that include the username, password and domains required to mount
  as the admin user.

* Other protocols other than NFS and CIFS have not been tested, they may work
  if their access type is included among the supported ones.

* In order to properly check compatibility with destination backends, drivers
  will rely on their local configuration files to read information about other
  backends, so it is advisable that deployers try to keep configuration files
  of multiple manila-share nodes synchronized and the latest values loaded in
  the services' memory.

Developer impact
----------------

Driver vendors and CI maintainers are advised to enable migration tests to
validate whether the host-assisted approach works for their respective drivers
and share protocols.

Implementation
==============

Upon receiving the API request to migrate a share, the Share API layer will
perform the following validations:

Check if share has replicas:

* if True, return error 409 (Conflict). Migration of a share with replicas is
  not handled at this moment.

Check share's status:

* If not available, return error 400 (Invalid).

Check if share is busy with another task:

* If busy, return error 400 (Invalid).

Check if destination host is different:

* If it is the same, return error 400 (Invalid).

Check if there are snapshots:

* If there are, return error 400 (Invalid). Migration of a share with
  snapshots is not handled at this moment.

Check if destination host is available:

* If it is not, return error 400 (Invalid).

Check if the new_share_type and share_network_id supplied exist:

* If not, return error 400 (Invalid).

If all validations succeed, it should set task_state to MIGRATION_STARTING and
invoke the scheduler asynchronously to validate the host against the share
type. If host validation fails, scheduler will set task_state to
MIGRATION_ERROR (no notification). Else it will invoke the source share's
manager also asynchronously to proceed with migration.

If new_share_type is supplied, it will be used when validating the host in the
scheduler instead of the share's original one. The new_share_network, if
supplied, will be used when creating the share instance model that will be used
by the migrated share, thus triggering the creation of a new share server, if
necessary.

At the share manager, it will change the share's task_state to
MIGRATION_IN_PROGRESS and instance status to MIGRATING. Then, it will prepare
to invoke the driver-assisted migration if the force_host_assisted_migration
API parameter is set to False.

First, it will attempt to perform the driver-assisted migration, by creating a
destination share instance model, obtaining a share server for it and invoking
a method that checks for compatibility. If it succeeds and returned
capabilities correspond to the supplied API values for 'writable',
'preserve_metadata' and 'non-disruptive', the task_state is set to
MIGRATION_DRIVER_STARTING and the driver's migration_start method is invoked,
in which the driver is expected to start the migration job and return a list of
export locations to access the destination instance, if possible at this point.

At this point, the task state is set to MIGRATION_DRIVER_IN_PROGRESS and a
period task runs to invoke the driver's migration_continue method to perform
the next steps of migration until it returns True, signaling that the first
migration phase has completed, allowing the task state to be set to
MIGRATION_DRIVER_PHASE1_DONE.

If any exception is raised before the first migration phase is completed, all
data allocated, such as the destination share instance model and share server
is cleaned up. If an exception is raised during the second migration phase,
data is not cleaned up so the administrator can analyze the failure and
possibly fix manually.

If the driver-assisted migration fails up to the migration_start driver call,
the host-assisted approach takes over, if the variables 'preserve_metadata',
'writable' and 'nondisruptive' supplied API values are all False.

The host-assisted approach code consists in changing all of share's access
rules to read-only through the driver (rules are not changed in DB), creating a
new share in the destination host through RPCAPI asynchronously and waiting for
it to have "available" status, obtaining the connection_info dictionaries for
the source and destination backends and invoking the Data Service
asynchronously to perform the migration.

The RPCAPIs for the Data Service include "migration_start" which perform the
data copy with regards to the logic for migration (like setting proper statuses
and notifying the source backend when necessary), "data_copy_cancel" and
"data_copy_get_progress" which are detailed below and are applicable to any
data copy job.

The connection_info dictionary consists in the access mapping compatible with
the share being migrated and two templates, one for the mount command, and one
for the unmount command. The base driver class has a default implementation for
these templates, and can be overridden if the driver requires a particular
custom behavior. Both command templates can be customized through the
manila.conf configuration file for each backend section, although the Data
Service expects at least the "%(path)s" section to be present in the template
so it can be replaced by the appropriate export location. Other template
elements need to be overridden by customizing the "connection_get_info" method.

The default mount template is: mount -vt %(proto)s %(options) %(export)s
%(path)s
The default unmount template is: umount -v %(path)s

The following access mapping is predefined in the driver base class::

  {
    'ip': ['nfs'],
    'user': ['cifs'],
  }

The Data Service does the hard work of migration, it is responsible for calling
the API to add the proper access rule to be able to mount both the source and
destination shares, mount, copy, unmount, and delete the access rules.

The way to determine the proper access rule type is according to the share's
protocol and access mapping configured by the driver. The share's protocol will
select the access types where the protocol entry is present and then intersect
with the other backend's access mapping to add an access rule that does not
cause errors for any of the involved drivers.

All access rules related to the access types present in the intersected access
mapping will be added and later removed after migration. To properly fill the
'access_to' field of the access rule entries, the Data Service reads a config
option for each type, as follows:

* If access type is 'user', it will read 'data_node_admin_user' config option.
  In this case, it is also expected that the administrator has filled the
  'data_node_mount_options' such as '-o user=foo,pass=bar' if that is necessary
  to mount the share.

* If access type is 'cert', it will read 'data_node_access_cert' config option.

* If access type is 'ip', it will read 'data_node_access_ip' config option.

* Else it will throw an error message that the access type provided is not
  supported.

It is expected that the Data Service node is configured properly in the admin
network, has proper libraries and certificates installed and security service
user configured.

The copy work is done by specialized copy class. This class is responsible for
iterating through files recursively, copy them attempting to preserve all
metadata (cp -P --preserve=all), optionally verify if the SHA-2 hashes of the
source and destination match, and finally attempt to apply the metadata of all
the files. All operations are performed as root so user restrictions can be
bypassed and set back to files after copy. Since all copy operations are
performed by running linux commands through rootwrap, the Data Service thread
sleeps until the process exits, thus allowing the Data Service to process
multiple RPC requests while the node is copying bytes. If any of the
above-mentioned operations fail or are not validated, it will be retried once,
and if it fails a second time, it will return an error and migration will fail.

After copying, the Data Service will set the task state to
DATA_COPYING_COMPLETED, allowing the admin to invoke migration-complete.

The share's manager migration_complete method will first check the migrating
share's task_state value to decide whether to invoke the driver's
migration_complete method or host-assisted approach one. The driver is expected
to perform the last disruptive steps of migration and return the list of
export locations pertaining to the migrated instance. The host-assisted
migration_complete applies the access rules to the new share according to the
DB (the original rules), sets the destination share status to "available",
deletes the source share and sets the task_state to MIGRATION_SUCCESS.

The migration_cancel API can only be invoked during copying or first phase is
completed. The migration_get_progress API can be invoked at any time, but it
will only query the driver or the Data Service for the progress if they are at
the step of migrating or copying files, respectively.

Assignee(s)
-----------

Primary assignee:
  ganso

Work Items
----------

* Implement changes agreed in this spec. Since they are improvements to
  existing code, they could be implemented as a single patch.
* Update python-manilaclient with the CLI commands.
* Update manila-ui.
* Document the implementation (see below).

Dependencies
============

* Update Access interface implemented in drivers.

Testing
=======

* Unit tests
* Tempest tests

Documentation Impact
====================

- Docstrings
- Devref
- Security guide
- User guide
- Release notes

References
==========

[1] Newton design summit etherpad discussion::

    https://etherpad.openstack.org/p/newton-manila-data-service-migration

[2] Mitaka design summit etherpad discussion::

    https://etherpad.openstack.org/p/mitaka-manila-migration-improvements

[3] Mitaka merged main patches::

    https://review.openstack.org/#/c/244286/
    https://review.openstack.org/#/c/250515/

[4] Liberty design summit etherpad discussion::

    https://etherpad.openstack.org/p/YVR-manila-liberty-share-migration

[5] Liberty merged main patch::

    https://review.openstack.org/#/c/179790/

[6] Access support mapping::

    http://docs.openstack.org/developer/manila/devref/share_back_ends_feature_support_mapping.html

