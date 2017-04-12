..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================
Add share groups to Manila
==========================

Blueprint: https://blueprints.launchpad.net/manila/+spec/manila-share-groups

Manila needs a grouping construct that, like shares, is a 1st-class atomic
data type. Our experience with CGs has demonstrated the complexity of adding a
grouping capability, yet there are other use cases such as migration,
replication, and backup in which some storage controllers could only offer
such features on share groups. CGs also highlighted the poor optics of an
advanced feature with comparatively little potential for vendor support. And
adding new grouping constructs for each new feature is not technically
feasible. All of the above may be addressed by share groups, which we think
is a clean extension to the original architecture of Manila.


Problem description
===================

Manila is at an exciting time in its development, where all the simple
features are available and the community is focused on adding high-value
features such as migration, replication, and consistency groups. Experimental
code is available for each of these, but little consideration has been given
to the obvious need for these complex features to interact not only with each
other but also with whatever additional features are added later. And looking
a little deeper, it is apparent that each of these has limitations that could
be solved by a single architectural enhancement.

Consistency groups
------------------

Consistency groups (CGs) are the only construct currently in Manila that
operate on groups of shares. For some, a CG implies a guarantee of ordered
writes by a storage controller, while for others a CG is a mechanism for
taking consistent point-in-time snapshots of a set of shares. A group with
either attribute may have value by itself, but the CGs implementation
doesn’t distinguish between them.

CGs are a highly specialized construct with dedicated CLI commands, REST APIs,
database tables, scheduler filters, and driver APIs that totaled over 9900
lines. None of these are reusable for anything else, and none of the rest of
Manila has any awareness of CGs. Even worse, despite all the complexity of the
CG feature, only a small minority of storage backends can support them, so the
code-to-value ratio is very low and the limited availability of CGs ensures
the user experience is inconsistent between clouds.

Replication
-----------

Replication is another high value feature, and the Manila implementation is
arguably clean and flexible, but as constituted there is no core ability to
replicate groups of shares. It seems reasonable to implement the feature
iteratively, beginning with replication of single shares. However, there has
been the tacit acknowledgement that some backends would not be able to
replicate individual shares, even though those same backends could potentially
replicate a group of shares if the group were constituted in a certain way.

The proposed approach would be to have group extra specs that dictate whether
the shares in the group are replicated in a consistent way, individually, or
neither.

Migration
---------

Just as some backends may only be able to replicate shares in groups, it
follows that those backends may also need to migrate shares in groups. And in
the case of CGs, it doesn’t make sense to migrate CG members individually; the
whole group must be moved, requiring the migration engine to be CG aware.

Use Cases
=========

Consider the following reasonable use cases:

* Replicate a consistency group
* Snapshot a consistency group
* Snapshot a replication group
* Migrate a replication group
* Retype shares in a consistency group
* Retype shares in a replication group
* Backup a consistency group
* Backup a replication group

On the current path, the support matrix has a dimension dedicated solely to
features, so each feature must be coded to be interoperable with every other
feature. This quickly becomes a support and testing nightmare, and it becomes
exponentially more complicated to add more features going forward.

Fundamentally, the problem is that we are adding features without an
underlying architectural framework on which to hang them. To escape the
matrix, we must step back and rethink a few things.

Proposed change
===============

So to arrive at a solution, let’s enumerate what we have:

* Primitive objects (shares) with supported operations controlled by share
  types that vary by backend
* A very specific group object (CGs) with limited support potential by
  backends
* A number of APIs (CGs, etc.) that many users can’t use at all and appear
  grafted into the project

And what would we prefer:

* A uniform API and user experience that varies as little as possible by
  backend
* A clean way to group shares so we can do CGs, multi-object replication,
  migration, etc.

How do we get there?

* Universally available groups of primitive objects with supported operations
  that vary by backend

Share Groups
------------

Just as Manila has ‘shares’, it should also have ‘share groups’. In its
simplest form, unlike CGs, a share group should not guarantee any specialized
operation. Instead, it should merely constitute an atomic Manila data type on
which nearly any Manila action is available. For example, given a share group,
the user should be able to select the group in the Manila UI and invoke
features such as snapshot, clone, backup, migrate, replicate, retype, etc.
Shares should be able to become a part of a share group only on share creation
step. If share was created without provided "share_group_id" then this share
won't be able to become a part of any share group. To do it, you will need
to use "share migration" feature. If share was created as part of a group, then
it can be moved out of a group only when it is either migrated or deleted.

In the general case, group actions are handled by the common share driver
superclass. For example, invoking ‘snapshot’ on a group causes the share
manager to take a snapshot of each group member individually. The resulting
group snapshot object may be used to create a new group, not unlike how CG
snapshots were implemented.

There are numerous advantages to this approach. Every driver, no matter how
unsophisticated, can leverage the manageability goodness of groups. The user
experience is uniform, since groups are always available regardless of which
backends are present and because nearly every Manila action available on
shares is also available on groups.

Of course, feature-rich backends would like to add their secret sauce, whether
it be CGs, group-based replication, or whatever else comes along that might be
easier/cheaper/faster to do to groups of shares. That brings us to a related
idea.

Share Group Types
-----------------
Just as Manila has ‘share types’, it should also have ‘share group types’.
Any driver that can perform a group operation in an advantaged way may
report that as a group capability, such as:

* Ordered writes
* Consistent snapshots
* Group replication
* Group backup

As with share types, the cloud administrator predefines share group types that
may contain group specs corresponding to the group capabilities reported by
the backends. The admin also specifies which share type(s) a given group type
may contain. If admin does not specify it, then manila will use 'default'
share type. The scheduler then creates the group on one of the backends that
match the specified share type(s) and share group type.

Anytime a group action, such as ‘snapshot’, comes into the share manager, the
manager checks whether its driver offers an advantaged implementation of that
operation. If not, the manager handles the workflow itself as described above.
But if so, the manager routes the workflow to its driver for fulfillment.

The advantages of this approach should be obvious. Development and testing are
simplified because there isn’t a need to define and test a different set of
group management APIs for each feature, or to test every combination of every
feature. Instead of becoming an N-by-N matrix of interacting features, Manila
largely becomes an N-by-2 matrix of actions that may be invoked on either
individual shares or share groups. Users and admins are already familiar with
share types, so introducing share group types would seem a natural and
consistent evolution of the same foundational concept.

====================== ==========================================
Share Action           Share Group Action
====================== ==========================================
Create (share type)    Create (share types, group type)
Delete                 Delete (group)
Snapshot               Snapshot (may or may not be "CG" snapshot)
Create from snapshot   Create from group snapshot
Clone                  Clone group (and all members) (planned)
Replicate              Replicate
Backup                 Backup (planned)
Retype                 Retype (planned)
Migrate                Migrate
Extend/shrink          N/A
====================== ==========================================

Details
-------

There are few things to note, several of which were already solved during the
CG work.

Groups are first-class objects in Manila, and operations on groups are treated
as atomic. To enable support by as many backends as possible, Manila will
still maintain DB objects for both group and member snapshots, just as was
done with CGs.

The capabilities of a group will all be public group specs, similar to
snapshot_support in share types. Users will need to know what a group can do,
as will tools like manila-ui.

Group quotas could be added later and described with separate spec. They are
out of scope for this spec, because this spec is, mostly, about porting CGs to
share groups and CGs haven't had quota implementation.

A few actions, such as extend & shrink, are inherently applicable only to
individual shares. One could theoretically apply extend to a group, increasing
the size of each member, but this would not be a use-case covered initially.
Any actions in this category must remain available to group members, and other
actions such as taking snapshots of group members can be allowed, but
actions such as migration or replication would be available only at the
group level and not on its members.

A group is limited to a single backend. Allowing groups that span backends is
theoretically possible, but that would require fanout of operations from the
API layer to multiple share managers across the asynchronous event bus, which
would lead to complicated synchronization and state management for little
operational benefit.

As was done with Manila CGs, a driver may optionally limit a group to either
the confines of a pool or an entire backend. It is known that pools are the
unit of data motion (i.e. replication or migration) for some backends, so we
think drivers need this flexibility.

Consistent with CGs, a grouped share must spend its entire lifecycle in the
group.  Adding or removing shares from groups at other than creation/deletion
time might be possible for some backends, while others might have to move the
data.  The migration engine is envisioned as the means of moving shares into
or out of a group.

We considered reusing share types for groups as well. But share types include
a set of public extra specs that may not map well to groups. And by adding
group types as a separate object, there can be little confusion about their
purpose and use.

The scheduler treats multiple extra specs as an AND operation, where
all features must be available for a backend to be chosen. It is conceivable
that even if a backend can replicate a group or take consistent snapshots of a
group, it might not be able to perform both operations on the same group. But
this problem already exists with share types and hasn’t been a serious issue.
Creating types is an admin-only operation and the burden remains on the admin
to understand the capabilities of the backends in use and to create the share
types and share group types appropriately.

Note that this proposal explicitly does not address pool or backend
replication, which is fundamentally different. Actions on shares or share
groups are intended for tenants, whereas a pool or backend can contain data
from multiple tenants. So pool or backend operations, while serving
potentially valuable use cases, are inherently admin-only workflows that would
be designed and exercised differently.

Alternatives
------------

One possible alternative is just removal of CGs. Where advantage is
simplification of code base and disadvantage is losing of a feature.

Other alternative is keeping CGs as already implemented and add other groups
for replication, etc., as needed.  As noted above, the downside is greater
complexity, larger codebase, and inconsistent user experience.

A higher-level, looser association construct could be added using tags, which
would allow overlapping membership as well as members in multiple backends,
but this would preclude the functionality sought in this feature that is only
available for shares on a single backend.  The tag-based association could be
added later as a parallel feature.

Data model impact
-----------------

The data model for groups should be virtually identical to that for CGs.  In
fact, generic share groups supersedes CGs, so the CG tables will be replaced
by those for share groups.

The data model for group types should be virtually identical to that for share
types, with the additional list of share types allowed for a group type.

REST API impact
---------------

It is possible to design a REST API that seamlessly handles both shares and
share groups with little duplication of APIs. But at this point in Manila's
development, it is arguably too late to radically redesign the API.

It may be possible to overload some of the existing APIs to handle both shares
and share groups. For example, POST /shares/{id}/action could accept the ID of
a share or share group and just do the right thing. But other APIs, such as
GET /shares are less practical to overload, since a single endpoint would be
returning objects of different types.

It seems better to merely duplicate a few APIs with group versions as needed.
For example:

* POST /shares --> POST /share-groups
* POST /shares/{share_id}/action --> POST /share-groups/{group_id}/action
* POST /snapshots --> POST /share-group-snapshots

Share group APIs:

* Create share group

  * URL: /share-groups
  * Method: POST
  * JSON body:

    .. code:: json

        {
          'share_group': {
            'name': 'fake_name',
            'description': 'fake_description',
            'availability_zone': 'fake_az',
            'group_type_id': 'fake_group_type_id',
            'share_network_id': 'fake_sn_id',
            'source_group_snapshot_id': 'fake_snap_id',
            'source_group_backup_id': 'fake_backup_id',
            'source_group_clone_id': 'fake_clone_id',
            'share_types': ['fake_st_id_1', 'fake_st_id_2', 'fake_st_id_n']
          }
        }

  * Notes: Keys 'source_group_backup_id' and 'source_group_clone_id' will
    be implemented only when appropriate features appear in manila.

* List share groups

  * URL: /share-groups
  * Method: GET
  * URL args:

    * all_tenants - integer, either 0 or 1
    * offset - integer, 0 or bigger
    * limit - integer, 1 or bigger
    * sort_key - string, key to be sorted (i.e. 'created_at' or 'status')
    * sort_dir - string, sort direction, should be either 'asc' or 'desc'.

* List share groups (detailed)

  * URL: /share-groups/detail
  * Method: GET
  * URL args:

    * all_tenants - integer, either 0 or 1
    * offset - integer, 0 or bigger
    * limit - integer, 1 or bigger
    * sort_key - string, key to be sorted (i.e. 'created_at' or 'status')
    * sort_dir - string, sort direction, should be either 'asc' or 'desc'.

* Show share group

  * URL: /share-groups/{share_group_id}
  * Method: GET

* Delete share group

  * URL: /share-groups/{share_group_id}
  * Method: DELETE
  * Note: Share group can only be deleted if it does not contain any shares.

* Force delete share group

  * URL: /share-groups/{share_group_id}/action
  * Method: POST
  * JSON body:

    .. code:: json

      {
        'force_delete': None
      }

  * Notes: it is admin-only API. Difference between this and common 'delete'
    command is that this one ignores status and raised exceptions.

* Reset share group status

  * URL: /share-groups/{share_group_id}/action
  * Method: POST
  * JSON body:

    .. code:: json

      {
        'reset_status': '%status%'
      }

  * Notes: it is admin-only API.

* Update share group

  * URL: /share-groups/{share_group_id}/action
  * Method: POST
  * JSON body:

    .. code:: json

      {
        'share_group': {
          'name': 'new_name',
          'description': 'new description'
        }
      }

Share group snapshot APIs:

* Create share group snapshot

  * URL: /share-group-snapshots
  * Method: POST
  * JSON body:

    .. code:: json

        {
          'share_group_snapshot': {
            'name': 'fake_name',
            'description': 'fake_description',
            'share_group_id': 'fake_share_group_id'
          }
        }

* Show share group snapshot

  * URL: /share-group-snapshots/{share_group_snapshot_id}
  * Method: GET

* List share group snapshots

  * URL: /share-group-snapshots
  * Method: GET
  * URL args:

    * offset - integer, 0 or bigger
    * limit - integer, 1 or bigger
    * sort_key - string, key to be sorted (i.e. 'created_at' or 'status')
    * sort_dir - string, sort direction, should be either 'asc' or 'desc'.

* List share group snapshots (detailed)

  * URL: /share-group-snapshots/detail
  * Method: GET
  * URL args:

    * offset - integer, 0 or bigger
    * limit - integer, 1 or bigger
    * sort_key - string, key to be sorted (i.e. 'created_at' or 'status')
    * sort_dir - string, sort direction, should be either 'asc' or 'desc'.

* Delete share group snapshot

  * URL: /share-group-snapshots/{share_group_snapshot_id}
  * Method: DELETE

* Force delete share group snapshot

  * URL: /share-group-snapshots/{share_group_snapshot_id}/action
  * Method: POST
  * JSON body:

    .. code:: json

      {
        'force_delete': None
      }

  * Notes: it is admin-only API. Difference between this and common 'delete'
    command is that this one ignores status and raised exceptions.

* Reset share group snapshot state

  * URL: /share-group-snapshots/{share_group_snapshot_id}/action
  * Method: POST
  * JSON body:

    .. code:: json

      {
        'reset_status': '%status%'
      }

  * Notes: it is admin-only API.

* Update share group snapshot

  * URL: /share-group-snapshots/{share_group_snapshot_id}/action
  * Method: POST
  * JSON body:

    .. code:: json

      {
        'share_group_snapshot': {
          'name': 'new_name',
          'description': 'new description'
        }
      }


Share group type APIs:

* Create share group type

  * URL: /share-group-types
  * Method: POST
  * JSON body:

    .. code:: json

      {
        'share_group_type': {
          'name': '%name%',
          'is_public': '%boolean%',
          'group_specs': {
            'foo_key': 'foo_value',
            'bar_key': 'bar_value'
          },
          'share_types': [
            'fake_share_type_id_1',
            'fake_share_type_id_2',
            '..',
            'fake_share_type_id_n',
          ]
        }
      }

* Show share group type

  * URL: /share-group-types/{share_group_type_id}
  * Method: GET

* Set share group type specs

  * URL: /share-group-types/{share_type_id}/group-specs
  * Method: POST
  * JSON body:

    .. code:: json

      {
        'group_specs': {
          'driver_handles_share_servers': True,
          'snapshot_support': True,
          'storage_protocol': 'NFS'
        }
      }

* Unset share group type specs

  * URL: /share-group-types/{share_group_type_id}/group-specs/{group_spec}
  * Method: DELETE

* List share group type specs

  * URL: /share-group-types/{share_group_type_id}/group-specs
  * Method: GET

* List share group types

  * URL: /share-group-types
  * Method: GET
  * URL args:

    * is_public - string, that contains either boolean-like value or 'all'.
      It will have the same behavior as in 'list share types' API.
      If not set or set to True, then admin/user sees only public share group
      types.
      If set to False, then admin sees only private share group types and
      user sees only those private share group types that are allowed to user's
      project.
      If set to 'all', then admin sees all share group types and user sees
      all except those private share group types that are allowed to user's
      project.

* Delete share group type

  * URL: /share-group-types/{share_group_type_id}
  * Method: DELETE

Share group type access APIs:

* Add project to access list

  * URL: /share-group-types/{share_group_type_id}/action
  * Method: POST
  * JSON body:

    .. code:: json

      {
        'addProjectAccess': {
          'project': '%project_id%'
        }
      }
  * Notes: applicable only to private group types

* Remove project from access list

  * URL: /share-group-types/{share_group_type_id}/action
  * Method: POST
  * JSON body:

    .. code:: json

      {
        'removeProjectAccess': {
          'project': '%project_id%'
        }
      }

  * Notes: applicable only to private group types

* List allowed projects

  * URL: /share-group-types/{share_group_type_id}/group-type-access
  * Method: GET
  * Notes: applicable only to private group types

* Changes to existing 'Create share' API:

  * New optional 'share_group_id' param will be added.

* Changes to existing 'Delete share' API:

  * New 'share_group_id' param will be required in URL as GET argument,
    when share is part of a share group.

The share group APIs will initially be experimental.

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

The Manila client, CLI, and GUI should ultimately be extended to
support share groups, ideally all in Ocata.  The client and CLI will be
updated first, by removing CG commands and adding group commands, to enable
Tempest coverage.

Performance impact
------------------

A group action may take longer if the share manager must iterate over several
individual shares, and drivers and storage backends must be able to handle
such requests.  Otherwise, performance should be comparable to the existing
consistency groups implementation.

Other deployer impact
---------------------

Share groups feature is optional and can be ignored by deployers in case
they do not need it. Also, they can disable all group-related APIs using
policy.json file.

If using groups, administrators must define group types and ensure their cloud
supports the combinations of group/share types they advertise.

Developer impact
----------------

One of the goals of this feature is to simplify maintenance and development
of Manila.  New feature authors will have to ensure any new feature works with
groups as well as individual shares (possibly in a phased implementation).
Driver authors will want to consider supporting group actions in an advantaged
way if possible on their storage systems, with the generic implementation
attempting to function with existing drivers.


Implementation
==============

Assignee(s)
-----------

Primary assignee:

* alex-meade

Other contributors:

* clintonk
* dustin-schoenbrun
* vponomaryov

Work Items
----------

Implementing generic groups in Manila should be a straightforward series of
steps:

#. Implement generic groups. Because we did CGs, we already know all parts of
   the codebase that must change to support any kind of group. So the simplest
   approach is to modify the CG code to morph it into the generic groups
   feature.
#. Add the share group type feature by duplicating and customizing the share
   type code.
#. Enhance the scheduler to place groups according to share group types. Like
   #1, this is already informed by the CG project.
#. Implement group snapshots in the share manager to demonstrate group
   snapshots in any driver.
#. Plumb group snapshots to a CG-capable driver to demonstrate CG
   functionality in the new framework.
#. Update Tempest to cover all of the above.
#. Add functional tests to manila client.
#. Add support of share group driver interfaces to dummy driver.
#. Update Manila client to change CGs to share groups.
#. Enhance Manila client with share group types.
#. Add share group support to manila-ui. We never built CG support into the UI,
   so this is all-new work that now has much broader appeal and applicability.
   It should be optional, but enabled by default, in the same way as
   share migration and replication feature support is implemented.
#. Add additional group actions over time (migrate, replicate, retype, clone,
   …). At this point, new group capabilities become vertical slices that are
   simple to add incrementally.

Because we implemented CGs just recently as an experimental feature, we have
the freedom to replace that code without deprecation or upgrade
considerations. Steps 1-8 would get Manila to parity with the CG feature added
in Liberty, would require only a few person-weeks of effort, and would better
position Manila for long-term evolution and supportability.


Dependencies
============

None


Testing
=======

Because share groups require no explicit driver support, they should be fully
testable using Tempest tests in the gate.  Because the minimal viable product
for this feature in Ocata is feature parity with CGs, the existing CG tests
should be adaptable to share groups. Manilaclient tests will be written from
scratch, because there is no CG tests for it right now.


Documentation Impact
====================

Share groups is a major feature that should be fully documented. At a minimum,
user-guide, admin-guide, in-tree API-ref and in-tree devref must be covered,
as with other experimental features so far.


References
==========

#. https://wiki.openstack.org/wiki/Manila/design/manila-generic-groups
#. https://etherpad.openstack.org/p/newton-manila-share-groups
#. https://etherpad.openstack.org/p/share-group-api-wip
