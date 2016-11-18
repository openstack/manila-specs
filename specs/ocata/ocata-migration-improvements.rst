..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
Share Migration Ocata Improvements
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
  snapshots.

* There has been a lot of debate on whether the default value of non-disruptive
  parameter should be True or False. Currently it is also inconsistent with all
  other driver-assisted related parameters, defaulting to False while all the
  other driver-assisted parameters default to True.

* The driver-assisted migration API parameters could be greyed out in the
  manila-ui "Start Migration" form so users better understand that "Force
  Host-assisted migration" negates the effects of the others.

* Currently the API does not immediately return an error on the obvious
  incorrect combination of "Force host-assisted migration" and other
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

1) Removing the snapshot restriction from the API layer, change it to work just
   like the other parameters' validation (writable, preserve_metadata, etc),
   thus adding an additional entry for migration_check_compatibility driver
   interface. Add the parameter to the API layer, named "preserve_snapshots".
   Also, this parameter, as the other ones, is not applicable for the
   host-assisted migration, preventing it from running if enabled.

2) Changing all REST API parameters to be mandatory, except for "force
   host-assisted migration". The purpose is to have better REST API support in
   the future, defaulting non-existing new REST API parameters to false when
   older microversions are used, and to avoid data loss in cases where the
   administrator does not specify a parameter by accident. For more
   information, see reference `[2]`_.

3) Grey out the driver-assisted options in manila-ui when "Force host-assisted
   migration" is selected.

4) Return an error when "Force host-assisted migration" API parameter is set
   along with other driver-assisted migration API parameters.

5) Improve the driver-assisted API parameters' descriptions in manila-ui and
   python-manilaclient to include "Enforce ..." in the name. Example: "Enforce
   Writable". Please note that this will not change the parameter name.

6) Changing the validation that rejects the API request when the parameters are
   the same as the source share to do so only when the combination of "new
   share network", "new share type" and "destination pool" are the same.

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

* The microversion will be bumped to include the changes proposed.

* The changes proposed will update the migration_start API as follows:

  - (POST, 202, 400, 409) migration-start: migrates share
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

* The same pool API restriction will be changed to not allow the same
  combination of "destination host", "share network" and "share type" as the
  source share.

* Specifying "force_host_assisted_migration" as True, along with any of
  "writable", "preserve_metadata", "preserve_snapshots" or "nondisruptive"
  being True, will also result in error 400, as it would be an incompatible
  combination.

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

This proposal will require updates to python-manilaclient. See example::

    manila migration-start <share> <host> <nondisruptive> <writable>
    <preserve_metadata> <preserve_snapshots> --force-host-assisted-migration
    --new-share-network --new-share-type

    manila migration-start share_1 ubuntu@generic1#GENERIC1 True True True True

Please note that during code implementation we may decide to change the syntax
shown in the example above with regards to the mandatory driver-assisted
parameters, in order to drive a better user experience with the CLI.

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

Implementation
==============

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
