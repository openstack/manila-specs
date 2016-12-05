..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
Share migration Ocata improvements
==================================

https://blueprints.launchpad.net/manila/+spec/ocata-migration-improvements

Proposal for Share Migration improvements and changes, based on
Barcelona Summit feedback.

Problem description
===================

At Barcelona Summit we gathered feedback on Share Migration Experimental API
feature `[1]`_. There were a few improvements suggested:

* For driver-assisted migration, drivers may be able to migrate a share and its
  snapshots, but the API layer is blocking the request when there are
  snapshots. This should only be the case when "force-host-assisted-migration"
  parameter is used.

* There has been a lot of debate on whether the default value of
  "nondisruptive" parameter should be True or False. Currently it is also
  inconsistent with all other driver-assisted related parameters, defaulting to
  False while all the other driver-assisted parameters default to True.

* The driver-assisted migration API parameters could be greyed out in the
  manila-ui "Start Migration" form so users better understand that "Force
  Host-assisted migration" negates the effects of the others.

* Currently the API does not immediately return an error on the obvious
  incorrect combination of "force-host-assisted-migration" and other
  driver-assisted parameters set to True. This will be useful for when the
  manila-ui is not used.

* The driver-assisted migration API parameters' descriptions in manila-ui and
  python-manilaclient could be improved to state that these parameters are
  "Enforced", as opposed to "not preserving metadata" when these properties
  are not selected.

* Currently the Share Migration API prevents migrations to the same
  storage pool. This prevents the cases of migrating shares between share
  servers or simply retyping. In such cases, it may be desired for the share to
  remain in the same storage pool, while being able to change the share's type
  or share network (and consequently the share server in the latter).

Use Cases
=========

The implementation of the changes suggested addresses existing problems and
enables use cases as seen below:

* Migrating shares along with snapshots
* Better user experience when choosing the API parameters
* Load balancing among share servers
* Retyping a share

Proposed change
===============

This spec proposes:

#) Removing the snapshot restriction from the API layer when
   "force-host-assisted-migration" is False, changing it to work just like the
   other parameters' validation (writable, preserve_metadata, etc), thus adding
   an additional entry for migration_check_compatibility driver interface. Add
   the parameter to the API layer, named "preserve_snapshots". Also, this
   parameter, as the other ones, is not applicable for the host-assisted
   migration, preventing it from running if enabled.

#) Changing all REST API parameters to be mandatory, except for
   "force-host-assisted-migration". The purpose is to have better REST API
   support in the future, defaulting non-existing new REST API parameters to
   False when older microversions are used, and to avoid data loss in cases
   where the administrator does not specify a parameter by accident. For more
   information, see reference `[2]`_.

#) Return an error when "force-host-assisted-migration" API parameter is set
   along with other driver-assisted migration API parameters.

#) Improve the driver-assisted API parameters' descriptions in manila-ui and
   python-manilaclient to include "Enforce ..." in the name. Example: "Enforce
   Writable". Please note that this will not change the parameter name.

#) Changing the validation that rejects the API request when the parameters are
   the same as the source share. The correct behavior should be to return
   HTTP 200 "SUCCESS" if the parameters requested match the share's properties,
   already having the same "share network", "share type" and "pool".

#) Migration should not be allowed to start when "access_rules_status" is in
   "ERROR" state. This is cause any migration to fail and should be blocked at
   the API layer. This spec proposes adding a validation for this.

#) Export locations should not be shown for the destination share instance
   during a migration, due to the fact that this instance cannot be
   mounted (because new access rules cannot be added during a migration) thus
   confusing the user that can see two export locations while being unable to
   determine which one is mountable.

#) With the addition of optional extra_specs saved in the Share model such as
   mount_snapshot_support, revert_to_snapshot_support and
   create_share_from_snapshot_support, it created the need to update these
   properties after a migration to match the destination instance share_type if
   it has been changed.

Alternatives
------------

An alternative solution has been discussed with regards to proposal #2 above,
where all options would be changed to False. This will imply that the default
migration-start command will be able to run the host-assisted migration if the
driver-assisted approach is unsupported or fails. We decided not to take this
approach because we wanted above all else to avoid surprises when performing
migration. We also considered defaulting all the driver-assisted parameters to
True, to avoid the surprise situation, but that would compromise the sanity of
the REST API with regards to future updates. We addressed these concerns by
insisting that clients always send a value of True or False for each option.
For more details see `[2]`_.

Data model impact
-----------------

None

REST API impact
---------------

* The microversion will be bumped to include the changes proposed. Support to
  previous microversion of migration-start will be dropped as we will only
  support the one with mandatory parameters.

* The changes proposed will update the migration_start API as follows:

  - (POST, 200, 202, 400, 409) migration-start: migrates share
    URL: /shares/<share_id>/action
    Body::

      {
        'migration_start': {
          'force_host_assisted_migration': false,
          'preserve_metadata': true,
          'writable': true,
          'nondisruptive': true,
          'preserve_snapshots': true,
          'host': 'ubuntu@generic2#GENERIC2',
          'new_share_type_id': 'foo_share_type_id',
          'new_share_network_id': 'bar_share_network_id'
        }
      }

* This API will not return an error immediately anymore when a share has
  snapshots, unless the "Force host-assisted migration" API parameter option
  is set.

* Not specifying any value for "nondisruptive", "preserve_snapshots",
  "writable" or "preserve_metadata" will now return error 400.

* The same pool API restriction will be changed to return 200 when the
  combination of "destination host", "share network" and "share type" is the
  same as the source share.

* Specifying "force_host_assisted_migration" as True, along with any of
  "writable", "preserve_metadata", "preserve_snapshots" or "nondisruptive"
  being True, will also result in error 400, as it would be an incompatible
  combination.

* Attempting a migration while "access_rules_status" is in "ERROR" will return
  error 400.

* Export locations for the destination migration instance are no longer shown
  until migration is complete.

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

This proposal will require updates to python-manilaclient. See example::

    manila migration-start <share> <host> --nondisruptive <True|False>
    --writable <True|False> --preserve-metadata <True|False>
    --preserve-snapshots <True|False> [--new-share-network <share_net>]
    [--new-share-type <share_type>]
    [--force-host-assisted-migration <True|False>]

    manila migration-start share_1 ubuntu@generic1#GENERIC1 --writable True
    --nondisruptive False --preserve-metadata True --preserve-snapshots False


As for manila-ui, there will be a new checkbox "Preserve Snapshots".

Performance impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

None

Driver impact
-------------

Driver maintainers will be prompted to update their driver-assisted migration
"migration_check_compatibility" implementation according to the new API
parameter 'preserve_snapshots' introduced. In this change, it will be added to
the existing implementations as "False".

All existing migration driver interfaces will be updated to include
snapshot-related parameters. See the new updated interfaces below::

    def migration_start(
            self, context, source_share, destination_share,
            source_snapshots, snapshot_mappings, share_server=None,
            destination_share_server=None):
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
        :param source_snapshots: List of snapshots owned by the source share.
        :param snapshot_mappings: Mapping of source snapshot IDs to
            destination snapshot models.
        :param share_server: Share server model or None.
        :param destination_share_server: Destination Share server model or
            None.
        """
        raise NotImplementedError()

    def migration_continue(
            self, context, source_share, destination_share, source_snapshots,
            snapshot_mappings, share_server=None,
            destination_share_server=None):
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
        :param source_snapshots: List of snapshots owned by the source share.
        :param snapshot_mappings: Mapping of source snapshot IDs to
            destination snapshot models.
        :param share_server: Share server model or None.
        :param destination_share_server: Destination Share server model or
            None.
        :return: Boolean value to indicate if 1st phase is finished.
        """
        raise NotImplementedError()

    def migration_complete(
            self, context, source_share, destination_share, source_snapshots,
            snapshot_mappings, share_server=None,
            destination_share_server=None):
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
        :param source_snapshots: List of snapshots owned by the source share.
        :param snapshot_mappings: Mapping of source snapshot IDs to
            destination snapshot models.
        :param share_server: Share server model or None.
        :param destination_share_server: Destination Share server model or
            None.
        :return: If the migration changes the export locations or snapshot
            provider locations, this method should return a dictionary with
            the relevant info. In such case, a dictionary containing a list of
            export locations and a list of model updates for each snapshot
            indexed by their IDs.

            Example::

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
                    'snapshot_updates':
                    {
                        'bc4e3b28-0832-4168-b688-67fdc3e9d408':
                        {
                        'provider_location': '/snapshots/foo/bar_1'
                        },
                        '2e62b7ea-4e30-445f-bc05-fd523ca62941':
                        {
                        'provider_location': '/snapshots/foo/bar_2'
                        },
                    },
                }

        """
        raise NotImplementedError()

    def migration_cancel(
            self, context, source_share, destination_share, source_snapshots,
            snapshot_mappings, share_server=None,
            destination_share_server=None):
        """Cancels migration of a given share to another host.

        .. note::
           Is called in source share's backend to cancel migration.

        If possible, driver can implement a way to cancel an in-progress
        migration.

        :param context: The 'context.RequestContext' object for the request.
        :param source_share: Reference to the original share model.
        :param destination_share: Reference to the share model to be used by
            migrated share.
        :param source_snapshots: List of snapshots owned by the source share.
        :param snapshot_mappings: Mapping of source snapshot IDs to
            destination snapshot models.
        :param share_server: Share server model or None.
        :param destination_share_server: Destination Share server model or
            None.
        """
        raise NotImplementedError()

    def migration_get_progress(
            self, context, source_share, destination_share, source_snapshots,
            snapshot_mappings, share_server=None,
            destination_share_server=None):
        """Obtains progress of migration of a given share to another host.

        .. note::
            Is called in source share's backend to obtain migration progress.

        If possible, driver can implement a way to return migration progress
        information.
        :param context: The 'context.RequestContext' object for the request.
        :param source_share: Reference to the original share model.
        :param destination_share: Reference to the share model to be used by
            migrated share.
        :param source_snapshots: List of snapshots owned by the source share.
        :param snapshot_mappings: Mapping of source snapshot IDs to
            destination snapshot models.
        :param share_server: Share server model or None.
        :param destination_share_server: Destination Share server model or
            None.
        :return: A dictionary with at least 'total_progress' field containing
            the percentage value.
        """
        raise NotImplementedError()


As can be noted above, the migration_complete driver interfaces had its
return value changed to return a dictionary structure containing the export
locations and a dictionary of snapshot updates, containing model updates for
each snapshot, in order to update the provider location in manila's database.

Implementation
==============

When starting a driver-assisted migration, it will be checked if drivers can
support "preserve_snapshots" regardless of the API option specified, due to the
fact that the migrating share may have existing snapshots. In case the driver
does not support "preserve_snapshots", an error message will be raised, stating
that driver-assisted migration cannot proceed while the share has snapshots.

If the driver does support "preserve_snapshots", it will be checked if all
existing snapshots have 'available' status. If so, destination snapshot
instances respective to each source snapshot instance will be created in the
database. Finally, a list of source snapshot instances and a mapping
dictionary, that comprises of destination snapshot instances indexed by source
snapshot instance IDs, will be passed to drivers in the updated driver
interfaces.

This mapping and list of snapshots will be easily retrieved from the database
at later stages such as when invoking migration_continue and
migration_complete. After migration is complete, the snapshot instances will be
updated according to the model updates returned by the driver, with fields such
as "provider_location" and "export_locations".

As for host-assisted migration, the validation of existing snapshots present in
in the API layer, has been copied to before starting a host-assisted migration,
as it will prevent the host-assisted migration from running.

Finally, the optional extra_specs of the share model are updated according to
the destination share type.

Assignee(s)
-----------

Primary assignee:
  ganso

Work Items
----------

* Implement main patch for manila that includes:

  - Updated Tempest tests
  - Updated Unit tests
  - API changes described in this proposal
  - Share migration host-assisted and driver-assisted changes required

* Implement additions and changes in python-manilaclient with:

  - Unit tests
  - Functional tests

* Implement additions and changes in manila-ui with:

  - Unit tests

* Update documentation for this feature (see `Documentation Impact`_ section)

Dependencies
============

No previous dependencies on other Ocata patches so far

Testing
=======

- Unit tests in manila, manila-ui and python-manilaclient
- Tempest API tests in manila and python-manilaclient

_`Documentation Impact`
=======================

- Docstrings
- Release notes
- Developer reference
- Admin reference
- API reference

References
==========

_`[1]` https://etherpad.openstack.org/p/ocata-manila-contributor-meetup

_`[2]` http://lists.openstack.org/pipermail/openstack-dev/2016-November/107186.html
