..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================================================
Prevent Access rules from being viewed or manipulated by non-owners
===================================================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/manila/+spec/protect-access-rules

In the OpenStack Shared File Systems service (Manila), by virtue of
default RBAC, share access rules (a.k.a "Access Control Lists") can be created,
viewed and deleted by all users of the project that owns the share. An
access rule can contain sensitive information. This specification proposes
an approach to prevent sharing of this sensitive information amongst all
users of the project.

Problem description
===================

When creating an access rule for a share, a user can provide type of access
requested (``access_type``), a client identifier (``access_to``),
the level of access (``access_level``) and metadata. All of this information
is namespaced to the share's project. So, all project users can view the
access rules associated with a share within the project. Sometimes this may
not be desirable. When a user creates a CephX access rule, they are able to
retrieve the CephX access secret key via Manila, this key can be private
information. Sharing this key among other users of the project would mean
that a CephX user key is visible and usable by multiple OpenStack project
users. If the goal was to allow only a subset of the OpenStack users to
access the share, the purpose is defeated with this level of visibility. The
same applies if individual CephX users had to have different access levels
("read-only" vs "read-write"); since the CephX users (``access_to``) and
CephX access secret keys (``access_key``) are easily available, nothing
prevents unauthorized access amongst project users.

In case of NFS with IP access control, users may not want to "leak" the
IP address of the client for security reasons, even amongst other users of
the same project.

Use Cases
=========

When creating and using CephFS shares, different OpenStack users would like
to protect their own CephX access keys from being visible to other users in
the project.

The OpenStack Compute service (Nova) creates an access rule to mount the share
on a compute host and make it available to a virtual machine via VirtIOFS.
It expects that the access rule data (``access_to`` and ``access_key``) are
not visible to other users. `[1]`_ It's important to clarify here that Nova
would be performing the API interactions with the user's own token. However,
Manila would rely on nova also supplying the ``X-Service-Token`` header.
Doing so would allow manila to create any visibility restrictions in favor of
the nova service user, instead of the project user.

Nova would also like to prevent users from deleting an access rule that was
created since it would hard mount the share to the compute host.

Proposed change
===============

When creating an access rule, users will be able to restrict the visibility
of the ``access_to`` and ``access_key`` fields to themselves. A restricted
access rule is still visible to all project users, however, these sensitive
fields are only visible to the user that imposed the restriction. A
restricted access rule cannot be deleted by other users in a project. Only
the user, or a more privileged user ("admin") can remove the restrictions.
Once a restriction is removed, other users in the project can view all
details regarding the previously restricted access rule.

These restrictions will be facilitated through `resource locks
<allow-locking-shares-against-deletion.html>`_. Through the rest of this
specification, for the sake of brevity, we'll refer to a resource lock that
imposes restrictions as a "access rule lock". The access rules API will be
enhanced to create such an access rule lock automatically, and drop it based on
new API input parameters.

An OpenStack service (like nova) can use the `X-Service-Token` and proxy the
user's own token. The restrictions created by a service user will be imposed
in favor of the service user in this case. This means that sensitive access
rule fields can only be viewed by the service user, and the rule can only be
deleted by the service user. Internally, manila will enforce this by recording
a "lock user context". In consequence, even when using a user's token, if a
service token is provided, the lock created cannot be updated or removed by
the user. Only a service user, or a user with "admin" role can remove or
update the lock. Manila will validate that the `X-Service-Token` corresponds
to a user with a ``service`` role.

This ability to restrict an access rule will have an important caveat. While
the deletion of the access rule is restricted, the corresponding share can
still be deleted. To prevent deletion of shares, a `dedicated resource
deletion lock <allow-locking-shares-against-deletion.html>`_ can be used.

Alternatives
------------

This feature can be limited to "admin" and "service" users only.
It'd be easier to implement the visibility and manipulation restrictions
directly via RBAC in this case. However, such an approach does not lend
itself to the self-service nature of the cloud. Regular users would have to
work through the administrator user to gain the ability to impose these
restrictions.

Data model impact
-----------------

None.

REST API impact
---------------

**Create an access rule with restrictions**::

  POST /shares/{share_id}/action

Normal http response code(s):

- 202 - Access rule creation accepted

Expected http error code(s):

- 401 - Unauthorized; user has not authenticated
- 400 - Bad Request
- 403 - Forbidden; user is forbidden by policy
- 404 - API does not exist in micro version

Request example::

    {
        "allow_access": {
            "access_type": "ip",
            "access_to": "203.0.113.10",
            "restrict": "True"
        }
    }


Response example::

    {
        "access": {
            "share_id": "406ea93b-32e9-4907-a117-148b3945749f",
            "created_at": "2023-04-30T09:14:48.000000",
            "updated_at": null,
            "access_type": "ip",
            "access_to": "203.0.113.10",
            "access_level": "rw",
            "access_key": null,
            "id": "a25b2df3-90bd-4add-afa6-5f0dbbd50452",
            "metadata": null
        }
    }

When an access rule has  restrictions applied, a different user
without the "admin" or "service" roles will see the ``access_to`` and
``access_secret`` fields set to ``******``.

**Delete an access rule with restrictions**::


  POST /shares/{share_id}/action

Normal http response code(s):

- 202 - Access rule deletion accepted

Expected http error code(s):

- 401 - Unauthorized; user has not authenticated
- 400 - Bad Request
- 403 - Forbidden; user is forbidden by policy
- 404 - API does not exist in micro version

Request example::

    {
        "deny_access": {
            "access_id": "a25b2df3-90bd-4add-afa6-5f0dbbd50452",
            "unrestrict": "True",
        }
    }


The API provides no response body. If the "unrestrict" field is not
specified, a restricted access rule cannot be deleted. The API will respond
with HTTP 400 in that case. If the "unrestrict" field is specified, but, the
user doing so isn't the one that restricted the rule; or isn't a "service"
user if the lock user context is "service", the API will respond with
error code HTTP 403.

**Create a restriction on an existing access rule**

    POST /v2/resource-locks

Normal http response code(s):

- 200 - Lock created successfully

Expected http error code(s):

- 401 - Unauthorized; user has not authenticated
- 400 - Bad Request
- 400 - Unrecognized action on access rule
- 400 - Unrecognized access rule (no such rule in project namespace)
- 403 - Forbidden; user is forbidden by policy
- 404 - API does not exist in micro version


Request example::

    {
        'resource_lock': {
            'resource_action': 'view,delete',
            'resource_type': 'access_rule',
            'resource_id': '222e1229-3de7-4678-9cb9-20cfd4ca9776',
            'lock_reason': 'infra host rule managed by fancyuser1'
        }
    }


Response example::

    {
        'resource_lock': {
            'id': '413080b6-1a20-48e1-9516-b2c509d034ec',
            'user_id': 'cec1dd3e297b45348228f4fc3f5dba38',
            'project_id': '2e47ac4e2cf04a5b8b8509de8177d65d',
            'resource_action': 'view,delete',
            'resource_type': 'access_rule',
            'resource_id': 'a448e0d2-7501-4b99-a447-1b89e3961e39',
            'lock_reason': 'infra host rule managed by fancyuser1',
            'created_at': '2023-04-28T09:49:58-05:00',
            'updated_at': None
        }
    }


**Update access rule restrictions**

To remove or update an access rule restriction without deleting the access
rule, see `<allow-locking-shares-against-deletion
.html#update-resource-locks>`__.

**Other API impact**

When access rule restrictions are present on a share, a transfer request to
move it across OpenStack projects cannot be accepted with keeping the rules
intact. The API will respond with 409 Conflict.

The REST API micro version will be increased indicating these new features.
Access rule restrictions can only be applied when using an API micro version
that is equal to or later than the micro version at which these API changes
appear. However, when access rule restrictions exist, they cannot be
circumvented by using a lower API micro version.

Driver impact
-------------

None. This change affects API behavior and is back end driver agnostic.

Security impact
---------------

The proposed change enhances the security model of the access control lists.
Restrictions can be created with new and existing access rules. The
enhancement is opt-in.

Notifications impact
--------------------

None

Other end user impact
---------------------

This feature will be supported by the OpenStack CLI plugin and
``manilaclient`` SDK in ``python-manilaclient``. The CLI interactions will
look like the following:

- Create an access rule with restrictions::

    openstack share access create <share> <access_type> <access_to> \
       [--access-level <access_level> \
       [--restrict] \
       [--properties [<key=value> ...]] \
       [--wait]


- Set the restrictions on an access rule::

    openstack share lock create <access_id> \
    --resource-action view-or-delete \
    --resource-type access \
    [--reason <lock_reason>}]

- Unset the restrictions on an access rule::

    openstack share lock delete <lock_id>

- Delete an access rule with restrictions::

    openstack share access delete <access_rule_id> --unrestrict


Performance Impact
------------------

Access rule restrictions introduce pre-conditions in API methods to list and
delete access rules. Evaluating these pre-conditions will introduce a minor
performance degradation. This should be made better with appropriate indices
in the `resource_locks` database table.

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
  gouthamr

Other assignee:
  carloss

Work Items
----------

- Testing with manila-tempest-tests
- Support in python-manilaclient, openstacksdk, manila-ui
- Documentation


Dependencies
============

This feature is required by the Nova/VirtioFS feature `[1]`_.


Testing
=======

Code changes will be covered with unit and tempest tests.


Documentation Impact
====================

- API Reference
- User Guide documentation


References
==========

_`[1]` `VirtIOFS Specification <https://specs.openstack
.org/openstack/nova-specs/specs/2023.2/approved/libvirt-virtiofs-attach-manila-shares.html>`_

[2] `Manila Specs: Allow locking shares against deletion
<allow-locking-shares-against-deletion.html>`_

[3] `2023.2 Bobcat PTG Discussion <https://etherpad.opendev
.org/p/nova-bobcat-ptg#72>`_
