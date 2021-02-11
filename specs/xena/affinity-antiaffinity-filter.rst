..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================
Affinity and anti-affinity scheduler filter
===========================================

https://blueprints.launchpad.net/manila/+spec/affinity-antiaffinity-filter

Copy of cinder feature
https://blueprints.launchpad.net/cinder/+spec/affinity-antiaffinity-filter

To add scheduler filter to manila that allows scheduler to make placement
decision based on affinity relationship between existing shares and new
share (the one being scheduled). The affinity relationship here means
the location of shares ('host' of share).

Problem description
===================

Manila has done a good job hiding the details of storage back ends from end
users by using share types.  However there are use cases where users who
build their application on top of shares would like to be able to 'choose'
where a share be created on.  How can manila provide such capability without
hurting the simplicity we have been keeping?  Affinity/anti-affinity is one
of the flexibility we can provide without exposing back end details.

The term affinity/anti-affinity here is to describe the relationship
between two sets of shares in terms of location.  To limit the scope, we
describe one share has affinity with the other one only when they reside in
the same share back end; on the contrary, 'anti-affinity'
relation between two sets of shares simply implies they are on different
manila back ends (hosts).

This affinity/anti-affinity filter filters manila back end based on hint
specified by end user. The hint expresses the affinity or anti-affinity
relation between new shares and existing share(s). This allows end
users to provide hints like 'please put this share to a place that is
different from where share-XYZ resides in'.

Use Cases
=========

DB team builds MySQL master onto one share, they'd prefer to put new
shares for slave DBs to different back ends from where the master DB
resides in, for the sake of high availability.

Proposed change
===============

Add two new filters to Manila - AffinityFilter and AntiAffinityFilter. These
two filters will look at the scheduler hint provided by end users and filter
back ends by checking the 'host' of old and new shares see if a back end meets
the requirement (being on the same back end as existing share or not being on
the same back end(s) as existing share(s)). Shares in different pools located
on the same host are still considered as based on the same host.

The scheduler_hints (same_host, different_host) have to be saved as a share
metadata as soon as new Share record is created in the database. The shares
referenced in those hints will receive the metadata update too, for instance:

1) There are three shares exist with: UUID1, UUID2, UUID3.
2) User provisions a new share with UUID4 and requests it with a hint to have
   different_host than UUID1, UUID2 and UUID3 shares.
3) Share UUID4 gets this hint stored with all other three shares UUIDs.
4) The metadata of each of three shares is also updated to reflect that they
   have anti-affinity to a new share with UUID4.

The respective share metadata is also automatically updated on share deletion
in the similar fashion.

This extra metadata should be marked as only "admin_modifiable" to prevent
non-admin users from accidential deletion or modification of the values.

Hint infomation is needed to allow user to understand how were the shares
scheduled and also to be able to respect the hints in case share is migrated
during its lifetime.

Alternatives
------------

There had been one proposal to allow admin user to directly specify the
back end for new shares.  It doesn't really provide similar functionality as
affinity filter because it was admin only and it itself has a few drawbacks
(security concern, for example).

Affinity is also possible with share groups. However, there are some
limitations of share groups to the use case proposed:

1) Affinity/Anti-affinity is only required at the time of provisioning the
resource, and not at any other point during its lifetime. Shares created
within share groups today are tied to the share group throughout their
lifetime.
2) Current feature limitations with share groups include the inability to
migrate or replicate shares within a share group.
3) While achieving affinity is possible with share groups,anti-affinity
is not.

There is an option to have a soft OR semantics for both filters instead of
hard AND. That means, the share creation should succeed if at least 1
condition has been met. The drawback is that there won't be a clear way
to present this information to the user since the share creation is async.
(When a user "fires and forgets" a creation request with hints where not
all conditions are met).

Alternative to storing scheduler_hints in share metadata can be extension
of Share DB model for saving the hints there. This approach causes more
modifications.

Data model impact
-----------------

None. The hints will be saved as additional share metadata.

REST API impact
---------------

Parameter 'scheduler_hints' added to share create, for example::

  `{
      "share": {
          "scheduler_hints": {
              "different_host": [
                  "share_uuid_1",
                  "share_uuid_2"
              ],
              "same_host": ["share_uuid_3"]
          }
      }
  }`

In this particular example, the share should be scheduled to the same host
where 'share_uuid_3' is located, yet only if both 'share_uuid_1' and
'share_uuid_2' are not located on that host.

All uuid-s given will be validated if exist and share creation
will fail if at least one share is not existing or uuid malformed.

The semantics for both filters is hard AND, meaning that the share won't
get created, unless all placement conditions are satisfied.

If the desired scheduling combination is not possible (e.g. affinity
with two shares located on two different hosts) - the share creation
should fail. The same applies for anti-affinity filter - if there are
only two hosts and each has a share with a uuid that was given in the
'different_host' hint for a new, third share - creation will fail.

The scheduler hints should also be respected if a share is migrated
between hosts, however there should be an option to override the hints
manually and force migration even if affinity/anti-affinity conditions
won't be met after the migration (e.g add 'force: true' option).

Microversion of the API is incremented.

Security impact
---------------

Although this change involves using or parsing user-provided - scheduler hints.
This doesn't put Manila in any more danger as it is now.


Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

New filters would query DB once per request, it only adds slightly latency
to the system and the latency has nothing to do with the size of the system.

The share metadata update might take extra time, especially when many shares
are specified in the hints. Yet share creation or deletion is an async process
and this impact should not be noticeable by the end user.

Other deployer impact
---------------------

None

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Dmitry Galkin (galkindmitrii in Gerrit)
  Kiran Pawar (kpdev in Gerrit)

Work Items
----------

1. Filter implementation
2. Add scheduler hints parameter
3. Add hints argument for python-manilaclient
4. Add hints support in manila-ui

Dependencies
============

None

Testing
=======

Test against AffinityFilter (Same host):
 * Create one share A;
 * Create another share B with uuid of A and 'same_host' as hint;
 * Checks if B is created on same back end as A;

Test against AntiAffinityFilter (Different host):
 * Create one share A;
 * Create another share C with uuid of A and 'different_host' as hint;
 * Checks if C is created on different back end as A;

Documentation Impact
====================

Need to document the usage of new filters.


References
==========

Nova has been offering similar feature called SameHostFilter and
DifferentHostFilter since *Diablo*.

https://github.com/openstack/nova/blob/master/nova/scheduler/filters/affinity_filter.py

Cinder has been offering similar feature called AffinityFilter and
AntiAffinityFilter since *Juno*.

https://specs.openstack.org/openstack/cinder-specs/specs/juno/affinity-antiaffinity-filter.html
