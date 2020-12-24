..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Metadata for all user facing resources
======================================

Launchpad blueprint:

https://blueprints.launchpad.net/manila/+spec/metadata-for-share-resources

Manila stores information regarding user-owned tangible technical assets in
its database as resources. These assets include file shares, snapshots,
access control rules, export locations, share networks and security services
among others. Resources are designed to accommodate distinguishable identifiers
geared at machines and humans such as ID, Name and Description. However
these identifiers may be insufficient or inflexible to end users' needs such
as being able to tag resources to a purpose, or attach labels that drive some
automation. This specification proposes a design allowing for user
modifiable metadata on all user facing resources.


Problem description
===================

End users interact with manila API to manage their shared file system storage
resources. These resources include:

- Shares
- Snapshots
- Export Locations
- Share Replicas
- Share Groups
- Share Group Snapshots
- Security Services
- Share Networks
- Share Network Subnets

All of these resources are identifiable by ID. Some of them even allow setting
free form text as Name and/or Description. Free form text is usually hard to
construct, breakdown or query, so these fields are mostly useful to humans
rather than machines. Many times, users want to be able to store much
more auxiliary contextual information.

One way they do that today with shares and access rules is with the help of
"metadata". Metadata is a term used to describe auxiliary information
associated with some data. For shares and access rules, users today use
metadata as flexible key=value pairs to attach useful optional/additional
information. This sort of interaction isn't possible with the other
resources enumerated above.

Metadata isn't just helpful for user interactions. It could also provide
additional representational space for manila itself. For example, the service
currently uses database objects to represent export location metadata to
relay helpful hints regarding the nature of specific exports to end users,
such as the preference and accessibility. This metadata is owned by the
service, and end users cannot modify this metadata.

Use Cases
=========

Users may want to use metadata to store contextual information such as:

- the purpose of the resource, e.g.: "usedfor=fileserver"
- details regarding the provisioner, e.g.: "createdby=manila-csi"
- grouping information, e.g.: "department=physics"
- additional chronological information, e.g.: "last_audited=2020-12-24T00:22:29"
- mount point tag for clients, e.g.: "fstag=website"

Metadata interactions are expected to be uniform and consistent across
resources, which makes it easier for building automation on top of these APIs.

Proposed change
===============

A new metadata controller mixin will be created in manila's API modules that
can be inherited by all the user facing API controllers. This mixin will
implement API handler code to:

- get resource metadata (retrieve either all of it, or specific item)
- set/unset all resource metadata (create, update or delete metadata items as
  key=value pairs)
- set a single resource metadata item (update a specific key=value pair)
- unset resource metadata item (delete a specific key=value pair)

To distinguish and protect service owned metadata, all metadata tables will
include an attribute named ``user_modifiable``. This attribute will
determine if users have the ability to manipulate (or delete)
the specific metadatum.

Alternatives
------------

Alternatives to key-value metadata include Name and Description fields.
Resources that don't currently support name and/or description can be
updated to include these fields. This alternative suffers from the
inflexibility problems described above and doesn't cover all the use cases.

We could also just add metadata controllers one at a time, based on user
feedback instead of adding metadata controllers for each user facing
resource. This also allows us to test each metadata controller in isolation
and build incrementally. This is pretty non-controversial, however, each
resource API update will have to be made in a new microversion, and doing
things one at a time instead of using the inheritence pattern to subsume
common code will be costlier. Testing effort is significant, and unavoidable.

Instead of distinguishing "user" owned and "service" owned metadata, we
could allow all metadata to be modifiable by end users - even data that's
created by the service. Doing so would simplify user interactions, however,
it may harm the service and cause misbehavior.

Another option to including a ``user_modifiable`` boolean field to metadata
is to introduce granular RBAC. However, this may produce too many RBAC rules
that may be unnecessary. If this turns out to be a use case, it is possible
to easily build on top of the proposed design.

Data model impact
-----------------

To store resource metadata, new tables and corresponding ORM models are
necessary. These tables will be created during a database upgrade, and
nullified and destroyed during a database downgrade.

Existing metadata tables for Shares ("share_metadata"), Export Locations
("share_instance_export_locations_metadata") and Share Access
Rules ("share_access_rules_metadata") will be modified to include a boolean
attribute called ``user_modifiable``. The default value for this attribute is
set to True. All existing export location metadata will have this attribute
set to False during the database upgrade step. This is because we expect no
end user created metadata on export locations yet.

Existing metadata tables will also be modified to replace the datatype of
the "id" field to uuids. This will be done to avoid integer overflows and
provide scalability.

A general ORM schema for a metadata table will be as follows::

  class ResourceMetadata(BASE, ManilaBase):
    """Represents a metadata key/value pair for a resource."""
    __tablename__ = 'resource_metadata'
    id = Column(String(36), primary_key=True)
    key = Column(String(255), nullable=False)
    value = Column(String(1023), nullable=False)
    resource_id = Column(String(36), ForeignKey('resources.id'), nullable=False)
    user_modifiable = Column(Boolean, default=True)
    resource = orm.relationship(Resource, backref="resource_metadata",
                                foreign_keys=resource_id,
                                primaryjoin='and_('
                                'ResourceMetadata.resource_id == Resource.id,'
                                'ResourceMetadata.deleted == 0)')


Metadata items are not soft deleted when they are unset by the service or by
end users. The metadata table is not loaded alongside the resource unless
the resource has been queried with metadata, or a detailed view of the
resource has been requested.

REST API impact
---------------

New API endpoints will be created to get metadata, set metadata, unset a
metadata item, delete all metadata for each resource. The general structure of
these APIs is as follows:

Get all Metadata
^^^^^^^^^^^^^^^^

Retrieve all metadata key=value pairs as JSON::

  GET /v2/{resource}/metadata

- *Sample request body*: null
- *Success Codes*: 200
- *Default API policy role*: Project Reader
- *Error Codes*: 401 (Unauthorized), 403 (Policy Not Authorized), 404
  (Invalid resource)
- *Sample response body*::

    {
       "metadata": {
           "project": "my_app",
           "aim": "doc"
       }
    }

Set/Unset all metadata
^^^^^^^^^^^^^^^^^^^^^^

Replace all metadata with the updated set, can also be used to delete all
metadata::

  PUT /v2/{resource}/metadata

- *Sample request body*::

   {
       "metadata": {
          "aim": "changed_doc",
          "project": "my_app",
          "new_metadata_key": "new_information"
       }
   }

- *Success Codes*: 200
- *Default API policy role*: Project Member
- *Error Codes*: 401 (Unauthorized), 403 (Policy Not Authorized), 404 (Invalid resource), 400 (Malformed request)
- *Sample response body*::

    {
       "metadata": {
          "aim": "changed_doc",
          "project": "my_app",
          "new_metadata_key": "new_information"
       }
    }

Set specific metadata item/s
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Update one or more specific metadata items, leaving the rest unmodified::

  POST /v2/{resource}/metadata/{key}

- *Sample request body*::

    {
       "metadata": {
          "aim": "updated_doc",
       }
    }

- *Success Codes*: 200
- *Default API policy role*: Project Member
- *Error Codes*: 401 (Unauthorized), 403 (Policy Not Authorized), 404 (Invalid resource), 400 (Malformed request)
- *Sample response body*::

    {
       "metadata": {
          "aim": "updated_doc",
          "project": "my_app",
          "new_metadata_key": "new_information"
       }
    }


.. important::

   Currently, the ``POST /v2/{share}/metadata`` API currently expects a
   ``meta`` object. However, the other metadata APIs expect a ``metadata``
   object. For the sake of consistency, this error will be fixed in a new
   API microversion. However, use of ``meta`` will be honored for this API even
   after the new microversion.

Delete specific metadata item
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Hard delete a single metadata key=value pair::

  DELETE /v2/{resource}/metadata/{key}

- *Sample request body*: null
- *Success Codes*: 200
- *Default API policy role*: Project Member
- *Error Codes*: 401 (Unauthorized), 403 (Policy Not Authorized), 404
  (Invalid resource)
- *Sample response body*: null

Query resources by metadata items
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

URL encoding can be performed by the client::

  GET /v2/{resource}?metadata=%7B%27foo%27%3A%27bar%27%2C%27clemson%27%3A%27tigers%27%7D

or the request can be made in a decoded format as well::

  GET /v2/{resource}?metadata={'foo':'bar','clemson':'tigers'}

Driver impact
-------------

None. Metadata manipulation is directly performed on the manila database and
shared file system back end drivers are not invoked during the creation,
modification or deletion of resource metadata.

Security impact
---------------

It is advised that metadata operations are rate limited to prevent bad
actors or automation from adding a large number of metadata items to a
resource.

Notifications impact
--------------------

None

Other end user impact
---------------------

Python-manilaclient SDK will include support for the new APIs and we'll
ensure that there are corresponding CLI commands in manilaclient shell as
well as the new OSC plugin shell. Manila UI's support for share, export
location and access rule metadata is limited. This specification doesn't
seek to address all the UI gaps; but all effort will be made to close the
feature parity between the CLI utilities and the UI. Eventually users will
be able to perform all metadata interactions via the UI as well.


Performance Impact
------------------

API performance is bound to suffer when resource queries include metadata
items. Since we'll be providing metadata along with ``detail`` retrievals of
resources, the performance of those APIs will also be affected negatively
because of the new database joins that will be necessary. Over time, as the
number of shares and the metadata tables grow, performance degradation can
be severe. This impact will be documented; as best practice, it is
recommended that a resource have no more than a handful of metadata items.

Other deployer impact
---------------------

New APIs introduced may require tweaking of policy files if the default RBAC
policy is not acceptable.

Developer impact
----------------

When adding any new user facing metadata, the metadata mixin controller can
be inherited and extended by developers.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  gouthamr

Other contributors:
  None

Work Items
----------

- Add database migration to convert the ``id`` field of  share, export
  location and access rule metadata to a string from integer and populate
  the field with UUIDs
- Add database migrations to introduce the "user_modifiable" field to share,
  export location and access rule metadata tables.
- Add database migrations to create new metadata tables for all other resources
- Add ``MetadataControllerMixin``, inherit and extend in all resources and
  bump up the API microversion.
- Add unit and integration tests
- Add support for metadata APIs in manilaclient SDK, manilaclient shell and OSC
- Add support for metadata interactions in the UI
- Add documentation

Dependencies
============

None


Testing
=======

Extensive unit tests will be written to test the API and database methods
being added. A database "walk migrations" test will be added for all the
database changes. Tempest tests will be added to cover the new metadata
operations across resources.


Documentation Impact
====================

- API reference
- End user guide
- Release notes


References
==========

* `Manila API Reference <https://docs.openstack.org/api-ref/shared-file-system>`_
* `API SIG Guideline on Metadata <https://specs.openstack.org/openstack/api-sig/guidelines/metadata.html>`_
* `Wallaby PTG Discussion Etherpad <https://etherpad.opendev.org/p/wallaby-ptg-manila-metadata-controller>`_
