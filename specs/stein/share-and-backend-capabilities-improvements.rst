..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================================
Improve share and back end capabilities in Stein
================================================

https://blueprints.launchpad.net/manila/+spec/storage-proto-enhancement

https://blueprints.launchpad.net/manila/+spec/make-share-shrinking-support-a-capability

Manila's back-end storage drivers offer a wide range of capabilities. The
variation in these capabilities allows cloud administrators to provide a
storage service catalog to their end users.

Sometimes back-end capabilities are very specific to the storage system, and
are opaque to manila or the end users - they provide programmatic
directives to the concerned storage driver to do something during share
creation or share manipulation. Cloud administrators know of the ``opaque
capabilities`` through driver documentation and they configure these
capabilities within share types as ``scoped extra-specs`` (e.g.:
`hpe3par:nfs_options`). The manila scheduler ignores scoped extra-specs
during its quest to find the right back end to provision shares.

There are some back-end capabilities in manila that `do` matter to the
scheduler. For our understanding, lets call these ``non-opaque capabilities``.
The documentation for these capabilities is here [1]_ and the share
back end feature support classification is provided here [2]_. All
non-opaque capabilities can be used within share types as ``non-scoped
extra-specs``. The non-opaque capabilities can be of three types:

- **Capabilities pertaining to a specific back end storage system driver**: For
  example, `huawei_smartcache`, these are considered by the scheduler's
  capabilities filter (and any custom filter defined by deployers). These
  capabilities are reported via the scheduler-stats API, however, no other
  manila API relies on them.
- **Common capabilities that are not tenant visible**: The manila community
  has standardized some cross-platform capabilities like
  `thin_provisioning`, `dedupe`, `compression`, `qos`, `ipv6_support`
  and `ipv4_support`. Values of these options do not matter to any manila
  APIs, however, they can signify something to the manila services
  themselves. For example when a back end supports `thin_provisioning`,
  the scheduler service performs over-provisioning, and if a back end
  does not report `ipv6_support` as `True`, the share-manager service drops
  IPv6 access rules before invoking the storage driver to allow/deny access
  rules.

  Although these capabilities are not tenant visible, administrators may use
  them as extra-specs in share types and the scheduler will match these
  capabilities in the various filters, including the Capabilities filter
  included with manila.
- **Common capabilities that are tenant visible**: Some capabilities affect
  functionality exposed via the manila API. For example, not all back ends
  support snapshots, and even if they do, they may not support `all` of the
  snapshot operations. For example, `snapshot_support`,
  `create_share_from_snapshot_support`, `revert_to_snapshot_support`,
  `mount_snapshot_support` and `replication_type`, etc. The support for
  these extra-specs determines whether users would be able to perform certain
  control-plane operations with manila. For example, the CEPHFS back end may
  report `snapshot_support` as `True` allowing end users to invoke the snapshot
  API, however, CEPHFS snapshots cannot be used by the driver to create a new
  share, so it reports `create_share_from_snapshot_support` as `False`. This
  reporting allows cloud administrators to create a share type that supports
  snapshots but not creating shares from snapshots. When a user uses such a
  share type and their share is created on a CEPH cluster, they cannot
  invoke the snapshot API successfully because the share type specifies that
  the feature be disabled for that specific share.

Tenant-visible capabilities aid manila in validating requests and
failing fast on requests it cannot accommodate. They also help level set the
user expectations on some failures. For example, if `snapshot_support` is
set to `False` on the share type, since users can see this, they will not
invoke the create snapshot API, and even if they do, they will understand
the HTTP 400 (and error message) in better context.

Synchronous failures can be acted upon by automation as well as end users
interacting manually with manila. Therefore, to the extent possible, all
validation that can be performed in the API must be performed because
failing early and fast is better than failing asynchronously.

Asynchronous failures may be a bit more painful to react to. With user
messages [3]_ tenants can examine the cause of asynchronous failures.
However not all user messages are actionable by end users.


Problem description
===================

This specification focuses on two problem areas: shrinking of shares and
storage protocol:

**Shrinking shares**

Shared File Systems are meant to be elastic. However, not all storage
systems can support reducing the size of the allocated filesystems.
Consider the case where manila is deployed with a back end that does not
support shrinking of shares. When a user tries to shrink a share provisioned
on that back end, the call reaches the share-manager process and fails at
the driver with a ``NotImplementedError``. The result of the failure causes
a state change on the share to ``shrinking_error``. There is no user message
either [4]_. Even though the share itself is not manipulated in any way,
the status on the share makes it impossible to do anything else with it.
Typically, users would have to contact the cloud administrator to recover
from this status change.

**Storage Protocols supported**

Currently there is a configuration option ``enabled_share_protocols`` that is
used by the manila API service to validate if a user's request can be
accepted for scheduling. If this option is mis-configured, we expect a
scheduling failure to occur (See Bug 1783736 [5]_). Users have no way of
knowing what storage protocols are have been configured and what protocols
are supported in a given deployment.  Drivers themselves may
support one or more storage protocols that they report to the scheduler.
We introduced capability lists in manila [6]_ several releases ago. However,
drivers that support multiple protocols are reporting a fused string to the
scheduler (ex: ``NFS_CIFS``).


Use Cases
=========

- Users must be able to determine beforehand if shrinking a share is possible
- Users must receive a synchronous API failure if shrinking is not supported
  for a given share, but they attempt it anyway.
- Cloud administrators must be able to configure storage protocols supported
  on a share type as extra-specs.
- Users must be able to determine what storage protocols are supported on a
  manila deployment through share types.
- The UI should be able to filter share protocols supported and enabled
  through the share types and vice versa.


Proposed change
===============

- A new common capability called ``shrink_support`` will be introduced. The
  base driver will determine if the ``shrink_share()`` interface is
  implemented and report the capability ``shrink_support`` appropriately.
- Cloud administrators will be able to configure ``shrink_support``
  as an extra-spec in share types. This extra-spec will be tenant visible, and
  can have one of three values: ``True``, ``False`` or ``legacy`` (reserved
  value).
- The scheduler will match only back ends that report
  ``shrink_support=True`` if the extra-spec's value is set to `True`. If the
  extra-spec's value is set to ``False`` or ``legacy``, the share could be
  scheduled to a back end that may or may not support shrinking shares.
- If the extra-spec's value is set to `False`, manila will set a boolean flag
  (also called ``shrink_support``) on the share object to `False`. This
  flag is inspected by the shrink API and if the value is `False`, shrinking
  will be disallowed right off the bat.

- If the extra-spec's value is set to ``legacy``, the scheduler
  disregards the capability and allows the share to be provisioned on any
  back end (as filtered and weighed by all other factors). On successful
  scheduling and share creation, the real value of ``shrink_support`` will
  be determined and set on the share by the share-manager process.

.. note::

   The extra-spec value ``legacy`` preserves backwards compatibility to
   schedule shares with pre-existing share types. See the `Upgrade impact`_
   section for more discussion on this proposal.

- Administrators can only set the value of ``shrink_support`` to a boolean
  expression. In consequence, they will be prevented from setting the value to
  the reserved value ``legacy``.

- Cloud administrators already have the ability to configure
  ``storage_protocol`` as an extra-spec. This extra spec will become tenant
  visible. If administrators do not configure this extra-spec, it will have
  a default value of ``'*'`` which indicates that shares of all storage
  protocols are allowed to be created. The value of this extra-spec can be a
  single protocol or list form, ex: ``<in> NFS <in> CIFS``.
- If the extra-spec ``storage_protocol`` is present and not set to ``'*'``,
  the API will validate if the requested protocol for the share being
  created is supported by the share type.
- The UI will be modified to parse the share type chosen and populate the
  allowed protocols in the drop down on the create share form.
- Drivers reporting multiple protocols will be updated to report them as a
  capability list instead of a string.
- The base driver will evaluate ``[DEFAULT]/enabled_share_protocols`` and
  intersect this list with the capability reported by the driver code to
  report the correct protocol/s supported in the deployment to the scheduler.

Alternatives
============

An alternative to not implementing ``shrink_support`` as a capability is
to disable the share shrinking API with the help of API policy
(``share:shrink``). However, this mechanism cannot be used if the cloud has
multiple back ends, one or more of which support shrinking shares while
others don't.

Users today cannot discover storage protocols they can use. We can live with
this situation by expecting cloud administrators to communicate what is
supported out-of-band of manila. This strategy may work for small
deployments where there is a tight communication between end users and cloud
administrators. For many others this inconvenience may be
annoying, for instance, consider a cross-cloud application that prefers to
discover capabilities and run on any OpenStack cloud.

Data model impact
=================

The ``manila.db.sqlalchemy.models.Share`` model will be modified to include
a nullable field called ``shrink_support``.

Upgrade impact
==============

There will be no data migration to populate the ``shrink_support`` capability
field on pre-existing shares. The initial value of this field will be
``null``. When users try to shrink a share that has its ``shrink_support``
attribute set to NULL, the API will continue to work as it does today, with
no change in behavior, whether or not the share can really be shrunk by the
back end.

In the share-manager service if ``NotImplementedError`` is raised by the
driver, the field is set to `False` and a user message is generated. The
status of the share will be set to ``shrinking_error``. After resetting the
share's status, if users attempt to shrink the same share again, the API
will be able to prevent it because the ``shrink_support`` has been
appropriately determined. If the share-manager service does not detect a
``NotImplementedError``, the ``shrink_support`` field will be set to `True`.

All existing share types will be updated to include the extra-spec
``shrink_support`` with its value set to ``legacy``. This upgrade impact will
be clearly called out in administrator documentation and release notes. By
virtue of setting this extra-spec, we preserve backwards compatibility. The
manila scheduler will ignore the ``shrink_support`` capability if its value is
``legacy``. If a user uses such a share type to create a share, a warning
message will be logged in the scheduler service suggesting that the
administrator must override the ``legacy`` value to enhance user experience.


The share's ``shrink_support`` attribute is determined at the driver after
share creation by the virtue of the capability known by the driver. If cloud
administrators do not change the value from ``legacy``, their deployments will
continue to provide a bad user experience. Users will not know until the
share is created whether they can shrink the share or not. The experience
further degrades if the same legacy share type matches two back ends: one
that supports shrinking shares, and one that doesn't. Users will see that
some shares are shrinkable while some others are not even when using the same
legacy share type.


REST API impact
===============

Please note the current state of these APIs in our
`API reference <http://developer.openstack.org/api-ref-share-v2.html>`_.

**Create and Manage share APIs**::

    POST /v2/{tenant_id}/shares
    POST /v2/{tenant_id}/shares/manage

Changes to request schema / headers / authorization / endpoint: None

Changes to API behavior / response schema / headers / authorization:

Response schema::

    {
        "share": {
            ...
            "shrink_support": true,
            ...
        }
    }

The API will set the share attribute of ``shrink_support`` to `True` if the
tenant visible extra-spec ``shrink_support`` has been set to a value that
evaluates to `True`. If ``shrink_support`` extra-spec has not been
configured, or if its value has been set to `False`, the ``shrink_support``
attribute on the share is set to `False`.

.. note::

   Please see `Upgrade impact`_ to understand how pre-existing share types
   will be dealt with.

The API will evaluate if the requested protocol is present in the
``storage_protocol`` extra-spec of the share type used. If not present, HTTP
400 is sent back to the requester suggesting that they are using a protocol
that is not supported.


**Shrink share API**::

    POST /v2/{tenant_id}/shares/{share_id}/action

Changes to request schema / headers / authorization / endpoint: None

Changes to API behavior / response schema / headers / authorization:

In a new API version, the ``shrink_support`` attribute of the share object
will be evaluated, and HTTP 400 will be returned to the requester if
shrinking is not allowed. Since this action is bound to fail if allowed, API
backwards compatibility will not be preserved to provide for a better user
experience.

**Other APIs**::

    GET /v2/{tenant_id}/shares
    GET /v2/{tenant_id}/shares/detail
    GET /v2/{tenant_id}/shares/{share_id}
    PUT /v2/{tenant_id}/shares/{share_id}

All of the above APIs will include a response schema change to include the
``shrink_support`` in a new API version.

The share types APIs::

    POST /v2/{tenant_id}/types
    POST /v2/{tenant_id}/types/{share_type_id}/extra_specs

already disallow non-boolean values for a list of tenant visible common
extra-specs. ``shrink_support`` will be added to this list.


Security impact
===============

None


Notifications impact
====================

A fix for LP 1783736 [4]_ will add a notification for a share shrink
operation that can fail asynchronously.


Other end user impact
=====================

None


Performance impact
==================

None


Other deployer impact
=====================

See `Upgrade impact`_ for the deployer's action when this feature lands.


Developer impact
================

None


Driver impact
=============

The driver capability ``storage_protocol`` will be changed as part of this
effort to cleanup support for multiple protocols. From an underscore
separated string it will be converted into a capability list.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  | gouthamr


Work Items
----------

- Introduce ``shrink_support`` capability. Modify create/manage share APIs
  and other share API view builders. Add database migration to introduce share
  capability attribute. Add base driver implementation to detect support for
  shrinking shares.
- Introduce ``storage_protocol`` tenant visibe extra-spec, modify driver
  reporting
- Add support for ``shrink_support`` in python-manilaclient and manila-ui
- Add support for ``storage_protocol`` in python-manilaclient and manila-ui
- Add tempest tests for both changes

Dependencies
============

None


Testing
=======

Unit test coverage will be added/maintained as per community standards.
Tempest tests will be modified/added to cover new API changes. Share shrink
tests will now create a new share type with ``shrink_support`` set to `True`
to test shrinking shares.


Documentation Impact
====================

The following OpenStack documentation will be updated to reflect this change:

* OpenStack User Guide: Capabilities documentation will be improved
* OpenStack Admin Guide: Share types changes will be documented and upgrade
  impact will be re-emphasized (Release notes will contain the impact and the
  actions in detail).
* OpenStack API Reference: New APIs will be documented
* Manila Developer Reference: No new documentation expected
* OpenStack Security Guide: No new documentation expected

References
==========

.. [1] Capabilities and Extra-Specs in the Rocky release https://docs.openstack.org/manila/rocky/admin/capabilities_and_extra_specs.html
.. [2] Manila share features support mapping in the Rocky release https://docs.openstack.org/manila/rocky/admin/share_back_ends_feature_support_mapping.html
.. [3] User Messages feature specification http://specs.openstack.org/openstack/manila-specs/specs/pike/user-messages.html
.. [4] No user message shrinking errors in the share manager https://bugs.launchpad.net/manila/+bug/1802424
.. [5] Scheduler does not filter storage protocol https://bugs.launchpad.net/manila/+bug/1783736
.. [6] Capability lists in Manila scheduler https://review.openstack.org/#/c/260054/
