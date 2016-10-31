..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================================
Fix and improve access rules and interactions with share drivers
================================================================

https://blueprints.launchpad.net/manila/+spec/fix-and-improve-access-rules

One of the major benefits that shared file systems offer is access control.
Manila's implementation of access control offers an easy-to-use interface to
allow and revoke client access to shared file systems. Access is controlled
on the basis of the respective type of access that makes sense to individual
storage protocols. For example, access to NFS Shares is controlled with IP
based rules, and to CEPH FS shares is controlled with CEPHX access rules, etc.
For many releases now, we have had to fix many issues with access control
APIs biggest of which were incorrect API behavior `[1]`_ and race conditions
`[2]`_.


Problem description
===================

Two major design changes in manila affected the access rules implementation:
Introduction of Share Instances `[3]`_ and bulk access rule updates on share
back ends `[4]`_. Prior to these changes the behavior was as follows:

* Tenant could request (or deny) access for a given share to a client
* Tenant could send x number of such requests in succession
* Tenant could list access rules on a given share and each rule had a
  distinct ``state`` that would inform the tenant if the rule was applied or
  denied successfully.

This API behavior was easy to write scripts around and even perform bulk
access rule operations on a given share. Since we had a per share per rule
``state`` and every access rule was individually applied or denied by a share
driver, there was better error reporting and lesser scope for a race
condition to occur.

The current behavior is as follows:

* Shares have instances. There is no ``state`` for an access rule for a given
  share instance, instead, the share instance has an ``access_rules_status``
  attribute that is a combined state across all the access rules of a given
  share instance.
* Tenants are not aware of share instances; access control is still
  performed at the share level. They can add or remove rules for a given
  share via the access control API. We do not support modifying multiple
  access rules at once, so API calls to ``allow_access`` or ``deny_access``
  only ever impact one access rule at a time.
* Tenants can send x number of such requests in succession, however, if new
  requests come in when rules are still being processed for the given share
  instance, the ``access_rules_status`` on the share instance will transition
  from 'active' to 'updating' and from 'updating' to 'updating_multiple'.
* If there was a request that the back end or the share driver could not honor,
  the exception is raised and the ``access_rules_status`` would be set to
  'error'. No further rule additions would be allowed, until all "bad" rules
  are removed.
* Identifying "bad rules" would require a careful amount of supervision on
  the tenant's part by noting each state transition; otherwise, the server
  would pretend that all rules are in 'error' state - owing to the fact that
  there is no tracking of the per share instance, per access rule state in
  the manila database.
* When a bunch of access requests are made on the same share, after a point,
  the share manager ignores the requests (i.e, share instance's
  ``access_rules_status`` transitions from 'active' to 'updating' on the first
  new rule, and from 'updating' to 'updating_multiple' when the next rule
  comes in and the status is still 'updating'. If any further requests come
  in when the status is 'updating_multiple', they are ignored.) `[5]`_
* Further, in an attempt to avoid race conditions in the share manager, the
  whole interaction to the share back end is synchronized with a lock. If the
  share driver or the back end takes a long time, the lock would prevent any
  further operations.
* If the share manager service crashes or is bounced when a share instance's
  ``access_rules_status`` is 'updating' or 'updating_multiple', users are
  stuck. No new rules can be added and because of poor error reporting. To
  recover from this state, users would have to perform another access rules
  API request (deny access to an existing client) to trigger an update of the
  access rules.
* When share instances were introduced as the underlying database
  representation for share replicas and migration instances, the design was
  conscious of the fact that "instances" would be visible to administrators
  and not end users (replicas are visible to tenants, however, they needn't
  know that replicas are share instances). However, error messages `[6]`_ seem
  to break the abstraction.
* When a share has multiple instances, the API sends an RPC request to the
  share manager host corresponding to each instance. Currently, the logic is
  as follows:

    * user requests access to a share for a client
    * manila-api performs basic validation on the state of the share
    * It then commits the rule to the database
    * It loops over the instances of the share, validating each one and
      shooting off the RPC request.
    * manila-api responds with a ``202 Accepted`` once all the RPC requests
      have been sent out.

  Here, if the validation fails for the second or subsequent share instance,
  the RPC call has already been made for the first share instance, resulting
  in the user receiving a ``400 Bad Request`` and the rule transitioning to
  'error', but the share might be accessible via export paths of the first
  instance.


Use Cases
=========

For a user of shared file systems, access control is arguably the most basic
requirement. The proposed change would allow preserving the good portions and
enhancing the user experience for access control in manila:

* Users can add any number of access rules to a given share in quick
  succession as far as the API is concerned.
* Users can identify which rule/s failed to apply.
* Users may continue to apply or deny rules while some rules are in
  'error' state.
* Users may be able to deny a rule that has not yet been applied.

Elimination of the currently existing race conditions will benefit both
users and cloud administrators.


Proposed change
===============

- Per share instance rule, per access rule ``state`` will be re-introduced.
- Four new access rules states will be introduced: 'queued_to_apply' (is
  currently 'new'), 'queued_to_deny', 'applying' and 'denying', the state
  transition for this:

  * Adding a new rule:

    **When the request is received by the API**

    share_instance.access_rules_status:

    - 'active'                ----> 'out_of_sync'
    - 'error'                 ----> 'error' (state will be preserved)
    - 'out_of_sync'           ----> 'out_of_sync' (state will be preserved)

    share_instance_access_map.state

    - 'queued_to_apply'       ----> 'queued_to_apply'
      (rule starts out in this state)

    **In the share manager, before the request is sent to the share driver**

    share_instance.access_rules_status:

    - 'error' / 'out_of_sync' ----> (state will be preserved)

    share_instance_access_map.state

    - 'queued_to_apply'       ----> 'applying'

    **When the share driver returns with the response**

    share_instance.access_rules_status:

    - 'error'                 ----> 'error' (state will be preserved)
    - 'out_of_sync'           ----> 'active'
      (if no other 'queued_to_apply' or 'queued_to_deny' rules)

    share_instance_access_map.state

    - 'applying'              ----> 'active' or 'error'

  * Deleting an existing rule:

    **When the request is received by the API**

    share_instance.access_rules_status:

    - 'active'                ----> 'out_of_sync'
    - 'error'                 ----> 'error' (state will be preserved)
    - 'out_of_sync'           ----> 'out_of_sync' (state will be preserved)

    share_instance_access_map.state

    - 'active'                ----> 'queued_to_deny'
    - 'applying'              ----> 'queued_to_deny'
    - 'error'                 ----> 'queued_to_deny'
    - 'queued_to_apply'       ----> 'queued_to_deny'

    **In the share manager, before the request is sent to the share driver**

    - 'queued_to_deny'        ----> 'denying'

    **When the share driver returns with the response**

    share_instance.access_rules_status:

    - 'error'                 ----> 'error' (state will be preserved)
    - 'out_of_sync'           ----> 'active'
      (if no other 'queued_to_apply' or 'queued_to_deny' rules)

    share_instance_access_map.state

    - 'denying'               ----> 'deleted' or 'error'


.. note::

    * When a share has multiple share instances, all instances of the share are
      expected to have the same access rules.

      In case of share migration, while existing access rules from the
      source share instance are eventually applied on the destination share
      instance, it begins out with no access rules.

      Also, when migrating a share, certain back ends may not be able to
      allow write operations on the source share during the migration,
      for a variety of reasons. Host based migration cannot handle new data
      being written into the share when existing data is being copied over.
      To ensure that all write operations are fenced off, manila casts
      existing rules on the source share to read-only prior to these
      kinds of migrations.

      In case of share replication, the expectation that clients have
      access to all replicas still holds with a clarification of semantics:
      For 'dr' replicas, we track all the access rules for the replica in
      the database, however, there are no export locations for the replica,
      hence, though the database contains these rules as being 'active' for
      the given replica, the replica is not accessible.

      In case of 'readable' replicas, any read/write rules are cast by the
      drivers to read-only for the secondary replicas; i.e, a rule's
      database representation does not change from 'r/w', the 'r/o'
      semantics on the replica are inherently ensured by the share drivers
      themselves `[7]`_ `[8]`_. This expectation off the drivers is an
      exception that exists only within this feature, i.e, the logic to
      ensure read-only semantics currently needs to live in each driver,
      though it is something that the share manager could handle uniformly,
      as we have in case of share migration.

      See further discussion within `Read-only access semantics`_.

.. important::

    * If all share instances are expected to have the same access rules,
      why would we still maintain per access rule per share instance states?

      While this may be clear with the purpose and design of share
      instances, let's clarify this once more for posterity: Each share
      instance is associated with a manila host. They can each be
      created at different times and acted upon independently, i.e, consider
      creating a share, allowing access and at a later time, creating a
      replica for it, or migrating it. Therefore, the ``state`` of access
      control should be tracked per rule, per share instance. This is the
      benefit of having an ``access_rules_status`` as an attribute of each
      share instance as well. The ``access_rules_status`` of an active replica
      can be 'active', while for a secondary replica where rules have not yet
      been synced, the attribute can be "out_of_sync". When the user lists
      the access rules however, the per share instance access rule states
      will be aggregated. See `Aggregate status fields`_ for more details.

- Rule status updates will be coordinated between the API service and the
  share manager service. The only state transition for an existing rule in
  the API service, as noted above, is performed while denying rules. The API
  service will acquire the same locks as the share manager service in order
  to make this transition.
- Manila-api and manila-share both have access to the database; the
  access rules payload is not necessary since these services
  can individually read the rules from the database and perform necessary state
  transitions. Thus, the RPCs: ``manila.share.rpcapi.ShareAPI.allow_access``
  and ``manila.share.rpcapi.ShareAPI.deny_access`` will be collapsed into
  ``manila.share.rpcapi.ShareAPI.update_access``.
- When the share manager receives the RPC request to update_access, it
  will react as follows:

  - Acquire a lock, look into the database for any rules in 'applying' or
    'denying' states for the given share instance. If there are any rules in
    these states, the driver is currently processing rule changes, any
    'queued_to_apply' or 'queued_to_deny' rules are `batched` to be applied or
    denied when the driver is done with its current task. Steps below are
    not executed right away. If there are no rules in 'applying' or
    'denying' states, set the state of any 'queued_to_apply' rules to
    'applying', and 'queued_to_deny' rules to 'denying'. Release the lock.

.. note::

    When the driver is processing an access rules update, any
    'queued_to_apply' or 'denying' rules are left alone until the driver
    returns from its current task. The ``update_access`` interface is
    designed for a bulk update. There is no order for processing rules. If
    access rules have to be processed in any particular order, it would be
    up to the share driver to do so.

  - Call the driver interface ``update_access`` passing existing rules and
    the "changed" rules ('applying' or 'denying' rules).
  - Accept ``state`` attribute to be set by driver, allowing for rules in
    transitional states to be updated by the driver. If the driver does not
    return a state for each rule in transitional state, transition 'applying'
    rules to 'active' and soft delete rules in 'denying' state. Perform
    these actions by acquiring a lock to read the current state, and
    releasing it at the end of the transaction.
  - Acquire a lock and read any 'queued_to_apply' rules that may have shown up,
    if any, repeat the last three steps, else continue to the next step.
  - Acquire a lock and transition the share instance's
    ``access_rules_status`` from ``out_of_sync`` to ``active``. ``error``
    state will be preserved if any access rules are
    in ``error`` state for the given share instance.

- The database API for retrieving a specific rule or all rules for a
  given share, or a given share instance will be refactored.
- The coarse lock around the ``update_access`` driver interface (or the
  fall back interfaces) will be removed. A reader-writer lock around
  database calls for these access rules will be introduced as pointed out
  above. This is because both manila-share and manila-api services read from
  and write to the database. For correct behavior, deployers should prefer a
  distributed lock `[9]`_ or a file lock living on a shared file system.
- On restarting the share-manager service, the 'recovery' step for access
  rules will be updated. All rules in 'applying' state will be reverted to
  'queued_to_apply' before requesting the driver to sync access rules for a
  given share instance.
- _`Read-only access semantics`:

  For share migration and share replication, access rules of a given
  instance, source instance for share migration or secondary replicas of a
  share, may require to be set to ``r/o`` (read-only).

  Currently, there is code to ensure the read-only semantics on the fly for
  the source share instance of a migration. This spec proposes adding a
  field to the database representation of share instances.

  The field will be called ``cast_rules_to_readonly``. It will be set and
  unset as necessary in migration and replication workflows where applicable.

  When we have the field, performing a check in the ``update_access`` workflow,
  before invoking the driver would be as simple as::

    if share_instance['cast_rules_to_readonly']:
        # The share manager will know to cast all rules to 'r/o'
        # before calling the share driver as long as this condition holds.

  The ``cast_rules_to_readonly`` field will be False by default for any
  share instance.

  The field will be set to True when creating a new replica on a share
  supporting ``readable`` style of replication. It will be unset on the
  replica when it gets promoted to being the primary (active replica) and
  set to True on the replica that gets demoted to being a secondary.

  The ``cast_rules_to_readonly`` field will be set to True at the beginning of
  each migration where necessary and unset always if any migration is canceled.

  Note that migration is not supported for replicated shares, replicas have
  to be deleted before migrating the share.

  The ``cast_rules_to_readonly`` field will not be exposed to tenants. It
  will be exposed to administrators in all APIs that return the share instance
  in "detail". It will not be present in the share model, not even as a
  @property because it solely belongs to a particular share instance.

  Setting and unsetting of the ``cast_rules_to_readonly`` will be synchronized
  by a lock.

  Supporting read-only rules is a minimum requirement within manila. If
  drivers do not support them, this and future evolution of manila features
  will expose differences and break the abstraction. There have been plans
  to act on such discrepancies, in terms of policy changes. Any action on
  these drivers is out of scope of this spec.

_`Aggregate status fields`
--------------------------

To preserve the abstraction at the API as far as users are concerned, we
need to aggregate status fields for both the share instance
``access_rules_status`` as well an per instance access rule ``state``
attributes:

     share_access_map.state
        non persistent, but reflects aggregate state from
        share_instance_access_map.state

        Aggregation: Read all share_instance_access_map.state, prefer in
        the following order: 'error', 'queued_to_apply', 'queued_to_deny',
        'applying', 'denying', 'active'

     share.access_rules_status:
        non persistent, but reflects aggregate state from
        share_instance.access_rules_status

        Aggregation: Read all share_instance.access_rules_status, prefer in
        the following order: 'error', 'out_of_sync', 'active'

Alternatives
------------

- Not having per share instance per access rule state: Live outside the
  reality that processes fail and things go wrong once in a while and even
  genuine user errors can upset well written driver or storage system code
  (There are drivers that error out entire requests on not understanding a
  rule type). If we don't separate the status field now, we would have to
  support annoyed users and document how to get out of messy situations.
- Not having transitional states: Allow coarse transitions as occurring
  today and live without the added benefit of concurrency control and poor
  user experience, i.e, allow all rules to go to 'error'.
- Keep the coarse lock in the share manager: Holding locks for long
  durations increases the chances of things going wrong when processes fail
  while holding locks and has a higher chance of running into deadlock
  situations `[10]`_.
- Don't refactor the API, allow the error messages to remain as they are: We
  need user documentation and awareness around what share instances are and
  how they work in manila. This breaks the abstraction and design, but maybe
  is not the worst thing in the complicated OpenStack ecosystem.
- Don't combine RPCs - reasonable, but no real benefit of separating them.
- Don't refactor the database API code - reasonable, it is a lot of code
  churn, but again, the number of database calls increases and we would be
  compromising on performance because we don't want to address this problem.
- Don't require a shared/distributed lock: The alternative would be to make
  state transitions only occur in one service (either manila-api or
  manila-share). However, the proposed design is superior in terms of
  correctness. We would need to compromise on the behavior changes: i.e,
  disallow denying a rule until it is 'active'; or the manila-data service
  must make requests via the manila-share or the manila-api to create new
  rules. The resulting code may be simpler, but the loss of functionality
  would be a backwards-incompatible API change. (This is perfect reasoning for
  a manila-conductor service, a la nova-conductor.)

  On the flip side, if a deployer does not deploy a DLM lock as suggested, and
  still distributes manila-api and manila-share services, the chances of
  running into race conditions is higher when the proposed implementation
  merges. However, this may not occur in some clouds where
  high volume of access requests on the same share are rarely performed, if
  ever. Note that any scripting of the access API may uncover these race
  conditions. Disclaimer: The developer driving this effort does not condone
  wrong deployment practices.
- Instead of the ``cast_rules_to_readonly``  field being added to the share
  instances, we can evolve a set of conditions within the code to ensure the
  read-only semantics that we desire for share instances.

  - For a host assisted migration, we can refer to the configuration
    parameter that specifies whether the host can support read-only access
    to a given share, and in the ``update_access`` workflow, we can change
    all access rules to read-only rules before invoking the driver.
  - Similarly, we can ascertain any ``readable`` replicas, within the
    ``update_access`` workflow and cast access rules to read-only before
    invoking the driver.
  - For ``writable`` and ``nondisruptive`` migrations, we can carefully ensure
    we never invoke ``update-access`` while migrating.

  While this seems straight-forward, it seems much complicated to maintain
  in the light of future feature functionality. It also overloads the use of
  multiple status fields. This creates a bottleneck, and a potential race
  condition when these status fields are updated by any process within
  manila. Also, we need to ensure a consistent behavior on restarting manila
  services. With this motivation, a field in the database seems necessary.
- Require that driver update the ``state`` attribute of rules. This is a
  reasonable ask. Allowing the drivers to update which rules were not acted
  upon successfully is more beneficial than the share manager setting all the
  transitional rule states to 'error' during a bulk update operation. We
  need to have code in the share manager that performs rule updates
  conditional to whether or not the driver has decided to return updates for
  each access rule. This is a similar pattern as seen with other existing
  driver entry-points. Therefore, while we will encourage drivers to revisit
  the ``update_access`` interface and return updates to the state field. It
  is not feasible to make these changes en-masse as part of this effort, not
  even for the first party drivers.

Data model impact
-----------------

The ``manila.db.sqlalchemy.models.ShareInstanceAccessMapping`` model will
have a new field called ``state``.

The database upgrade step will add this column by populating it with the
value of existing
``manila.db.sqlalchemy.models.ShareInstance.access_rules_status`` column.

Values of the ``manila.db.sqlalchemy.models.ShareInstance.access_rules_status``
column will be re-mapped to remove 'updating' and 'updating_multiple' as valid
access_rules_status values.

Values of the ``manila.db.sqlalchemy.models.ShareInstance.access_rules_status``
column will be re-mapped to remove 'updating' and 'updating_multiple' as valid
access_rules_status values.

The ``manila.db.sqlalchemy.models.ShareInstance`` model will have a new
field called ``cast_rules_to_readonly``.

The database upgrade step will add this column with a default value of
``True`` for all share instances that have their ``replication_type``
attribute set to ``readable`` and whose ``replica_state`` is not ``active``
(secondary replicas in a readable type of replication). For all other share
instances, the value will be set to False.

In the ORM layer, ``manila.db.sqlalchemy.models.ShareInstanceAccessMapping``
model will now contain a back-ref to the access rules model
(``manila.db.sqlalchemy.models.ShareAccessMapping``). That way, the database
APIs associated with reading a given share instance access mapping row will
have access to the related access rule data (such as ``access_type`` and
``access_level``).

The database downgrade step will drop the ``state`` column from
``manila.db.sqlalchemy.models.ShareInstanceAccessMapping``. It will not
alter the ``manila.db.sqlalchemy.models.ShareInstance.access_rules_status``
column.

The database downgrade step will also drop the ``cast_rules_to_readonly``
column from ``manila.db.sqlalchemy.models.ShareInstance``.

As always, this downgrade is not recommended in a production cloud.

REST API impact
---------------
No new APIs will be added. While we will bump the micro-version to expose the
'queued_to_deny' state, transitional 'applying' and 'denying' states,
some other changes will be made without bumping the micro-version, since the
behavior is currently broken in the API and it is hard to warrant requiring
backwards compatibility given the unpredictable/undesirable behavior:

**Adding an access rule**

    POST /v2/{tenant-id}/shares/{share-id}/action
    BODY::

        {
            'allow_access': {
                    "access_level": "rw",
                    "access_type": "ip",
                    "access_to": "0.0.0.0/0"
            }
        }

- Policy check will be completed first before any other validation
- If any share instance is invalid, the API will return without creating the
  rule in the database.
- Adding access rules while the share is being migrated will be possible, as
  long as all share instances have a valid host. This behavior will be
  exposed only from a new micro-version of the allow-access API.

**Denying an access rule**

    POST /v2/{tenant-id}/shares/{share-id}/action
    BODY::

        {
            'deny_access': {
                   "access_id": "a25b2df3-90bd-4add-afa6-5f0dbbd50452"
            }
        }

- Policy check will be completed first before any other validation
- If any share instance is invalid, the API will return without denying the
  rule for any other valid instance.
- Access rules cannot be denied when the share instance status is not
  'available'.

**Listing access rules**

    POST /v2/{tenant-id}/shares/{share-id}/action
    BODY::

        {
            'access_list': null
        }

- Policy check will be completed first before any other validation
- Transitional statuses introduced will be mapped to pre-existing states for
  API requests with a micro-version lesser than the micro-version where they
  were introduced. Mapping 'applying' and 'queued_to_apply' is easy, they will
  be set to 'new'; mapping 'queued_to_deny' and 'denying' is tough because the
  state could have been anything prior to transitioning to 'denying'. So, we
  will read the share's aggregate ``access_rules_status``, if it is 'error',
  we will map the state of this access rule to 'error', else if the
  ``access_rules_status`` is 'out_of_sync', the rule's state will be mapped
  as 'new', and if the ``access_rules_status`` is 'active', the rule's state
  will be mapped to 'active'. Note that this was the behavior that we are
  trying to change by re-introducing the per share instance per access rule
  state.

**Share Instance APIs**

- The ``cast_rules_to_readonly`` field will be exposed in the "detail" views of
  share instances (admin-only) as a micro-versioned change.

Security impact
---------------

Bulk access rule updates will no longer be ignored by manila. We will also
support denying access in case access was accidentally granted. Since
tracking of state changes would be improved, we expect a positive security
impact.

Notifications impact
--------------------

None until `[11]`_ merges and is fully supported in manila. The work to add
user messages will be proposed with a new blueprint and not part of this work.

Other end user impact
---------------------

End users will be prompted to prefer the new micro-version in case of
writing applications against a newer manila release, so as to gain full
benefit of the more predictable state transitions. When manila is upgraded
in their clouds, they will already benefit from faster API failures as
opposed to the previous versions, no matter what API version they use.

Performance impact
------------------

Changes to the ORM and the database API will reduce redundant calls to the
database, and is expected to have a positive impact on the performance.
Instead of reacting to every RPC request and triggering update_access calls
on the driver, the proposed implementation defers un-applied rule requests
if a driver is processing rule requests. This could lead to batching of
requests to take effect at once, thereby reducing the number of calls to the
storage back end, hopefully improving performance.

Other deployer impact
---------------------

Until tooz `[9]`_ support is added in manila, we expect deployers to support
manila with file locks created with oslo_concurrency. In multi-node
deployments, these file locks must be accessible to all nodes where all
manila services are run. Typically, this is achieved with storing these
locks on a shared file system.

Once tooz support is added, synchronization introduced in this patch will
automatically benefit from the locking abstraction introduced by tooz; and
deployers may choose to configure a distributed lock management system
underneath manila/tooz.

Since no state is saved in the services, this proposed change does not
introduce any regression to affect `active-active` deployment choices.

Developer impact
----------------

None

Driver impact
-------------

As always, when denying a rule, if the rule does not exist on the back end,
the drivers must not raise an exception, they may log that the access rule
never existed, and return None allowing for the rule to be deleted from the
database.

Raising exceptions will set all rules in transitional states, 'applying' or
'denying' to 'error'. So, drivers must carefully consider exception handling.

As part of this work, we will accept 'error' rule states from the driver in the
response, so, drivers can tell the share manager which exact rules failed to
be applied. This needs to be added to each driver considering how each back
end can handle error reporting.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  | gouthamr

Other contributors:
  | yogesh
  | rooneym

Work Items
----------

* Code - For ease of implementation and review, this work will be done in
  multiple related change sets, each will partially implement the blueprint
  until all of the proposed items are implemented.
* Functional tests in server and client


Dependencies
============

None


Testing
=======

- Negative/API tests will be added for the changes in the API including
  scheduling invalid shares or invalid replicas and testing the access
  rules interaction at the API.
- Existing access rules tests will be modified to validate the
  access_rules_status but wait on the access rule's state attribute in both
  manila and python-manilaclient.
- The scenario tests spec `[12]`_ proposes to add tests around the read-only
  semantics. This simplification should pass the tests proposed.

Documentation Impact
====================

The following OpenStack documentation will be updated to reflect this change:

* OpenStack User Guide: will document the changes in state transitions.
* OpenStack API Reference: All API changes will be documented
* Manila Developer Reference: A state transition diagram will be added for
  access rules' ``state`` and share instances' ``access_rules_status``.

References
==========

_`[1]`: https://bugs.launchpad.net/manila/+bug/1550258

_`[2]`: https://bugs.launchpad.net/manila/+bug/1626249

_`[3]`: https://blueprints.launchpad.net/manila/+spec/share-instances

_`[4]`: https://blueprints.launchpad.net/manila/+spec/new-share-access-driver-interface

_`[5]`: https://bugs.launchpad.net/manila/+bug/1626249

_`[6]`: https://github.com/openstack/manila/blob/4c6ce2f/manila/share/api.py#L1417

_`[7]`: http://docs.openstack.org/developer/manila/devref/share_replication.html#access-rules

_`[8]`: http://docs.openstack.org/admin-guide/shared-file-systems-share-replication.html#access-rules

_`[9]`: https://review.openstack.org/#/c/318336/

_`[10]`: https://en.wikipedia.org/wiki/Dining_philosophers_problem

_`[11]`: https://blueprints.launchpad.net/manila/+spec/user-messages

_`[12]`: https://review.openstack.org/#/c/374731/
