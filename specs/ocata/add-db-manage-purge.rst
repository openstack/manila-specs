..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
Manila db purge utility
=======================

https://blueprints.launchpad.net/manila/+spec/clean-deleted-row-in-db

This spec adds the ability to sanely and safely purge soft-deleted rows from
the manila database for all relevant tables. Presently, we keep all deleted
rows. And this is unmaintainable as we move towards more upgradable releases.
Today, most operators depend on manual DB queries to delete this data, but
this opens up to human errors.

The goal is to have this be an extension to the manila-manage db command.
Similar specs are being submitted to all the various projects that touch
a database.

Problem description
===================

Very long lived OpenStack installations will carry around database rows
for years and years. This brings the following problems:

* If deleted data is kept in the DB, the number of rows can grow very large
  taking up the disk space of nodes. Larger disk space means more worry
  for disaster recovery, long running non-differential backups, etc.

* Large number of deleted rows also means that an admin or authorized owner
  querying for the corresponding rows will get 5xx responses timing out
  on the DB, eventually slowing down other queries and API performance.

* DB upgrade ability is a big challenge if the older data style are less
  or inconsistent with the latest formats. An example would be the image
  locations string where older location string styles are different
  from the latest.

To date, there is no "mechanism" to programmatically purge the deleted data.

Use cases
=========

Operators should have the ability to purge deleted rows, possibly on a
schedule (cron job) or as needed (Before an upgrade, prior to maintenance).
The intended use would be to specify a number of days prior to today for
deletion, e.g. "manila-manage db purge 10" would purge deleted rows that
have the "deleted_at" column greater than 10 days ago

Proposed change
===============

The proposal is to add a "purge" command to manila-manage db command
collection. This will take a non-negative number (0 will delete the
rows "up to now") of days argument and use that for a data match. Like:

    DELETE FROM shares
      WHERE  deleted_at <= NOW() - INTERVAL <specified_days> DAY

Note: row with attribute(s) used as a foreign key will not be deleted
even if it satisfies the deleted_at time requirement.

To accomplish this, we rearrange the table list in case of foreign
key constraint, To make the logic simple and direct, we would hard
code the whole table list with variable 'PURGE_TABLE_LIST', the existing
tables in order are below (30 in total, 'manila_nodes' is excluded as it
does not exist in db):

* availability_zones
* services
* quotas
* project_user_quotas
* quota_classes
* reservations
* quota_usages
* cgsnapshots
* cgsnapshot_members
* share_instance_access_map
* share_access_map
* share_snapshot_instances,
* share_instance_export_locations_metadata,
* share_instance_export_locations
* share_snapshots
* share_instances
* share_type_projects
* share_type_extra_specs
* share_types
* share_metadata
* shares
* consistency_groups
* consistency_group_share_type_mappings
* share_network_security_service_association
* security_services
* share_server_backend_details
* network_allocations
* share_servers
* share_networks
* drivers_private_data

This list should be synchronized when the database schema is changed.

Alternatives
------------

Today, this can be accomplished manually with SQL commands, or via script.

Data model impact
-----------------

None

REST API impact
---------------

None

CLI impact
----------

A new manila-manage command will be added:

   manila-manage db purge <age_in_days>

Driver impact
-------------

None


Security impact
---------------

Low, This only touches already soft deleted rows.

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance impact
------------------

This has the potential to improve performance for very large databases.
Very long-lived installations can suffer from inefficient operations
on large tables.
This would have negative DB performance impact while the purge is running.

Other deployer impact
---------------------

None

Developer impact
----------------

Developer should update the table list whenever the tables' relationship
changed.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

* zhongjun(jun.zhongjun2@gmail.com)
* TommyLike(tommylikehu@gmail.com)

Work items
----------

* Implement 'db purge' command.
* Add related unit tests.
* Add documentation of this feature.

Dependencies
============

None

Testing
=======

Unit testcases which focus on algorithm:

* Single table with rows vary in 'deleted_at' time.
* Multiple tables with inner relationship and their rows vary in
  'deleted_at' time.

Unit testcases which focus on the hard code list (PURGE_TABLE_LIST)'s
consistency:

* All tables in manila should be added to list (exception to this also can
  be added here in purpose).
* Each child table that uses foreign key(s) should come before its parent
  table(s).

Documentation impact
====================

Documentation of this feature will be added to the admin guide and
developer reference.

References
==========

This is already discussed and accepted in other OpenStack components,
such as Glance [1] and Cinder [2].

[1] https://specs.openstack.org/openstack/glance-specs/specs/mitaka/database-purge.html
[2] https://specs.openstack.org/openstack/cinder-specs/specs/kilo/database-purge.html
