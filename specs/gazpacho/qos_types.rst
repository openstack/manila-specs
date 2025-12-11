..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================
Manila qos types
================

Blueprint: https://blueprints.launchpad.net/manila/+spec/qos-types

This blueprint proposes adding new object representing quality of service(QoS).
The QoS type object will be having specs representing attributes needed to
define quality of service.


Problem description
===================

Manila backend drivers support Quality Of Service (QoS) feature which allows
operator to define certain quality parameters on share. The parameters of QoS
are passed in share-type extra-specs as driver-specific parameters. Operators
must define extra-specs to describe QoS, which creates duplication, lack of
reuse, and poor visibility across share types. Backend drivers support
DHSS=True and DHSS=False. The way of defining QoS by using share-type
extra-specs works for deployments where policy and resource management occur
outside of the Openstack environment. However for Manila integrated management,
where resources and policies defined during run-time, share-type extra-specs
is not the efficient way to define QoS on Manila resources due to reasons
mentioned earlier.
This specification proposes an approach that enables Manila to create multiple
qos_types and optionally mention one of them in share type extra-spec. The qos_type
represents a QoS policy which will be applied on resource e.g. share.


Use Cases
=========

1. Operators want to define reusable QoS profiles (e.g. "Gold-QoS",
   "Silver-QoS") and convert them on QoS policies that will be applied on
   various Manila resources.

2. Tenants need visibility into the QoS characteristics of a share type before
   provisioning a share.

3. Drivers can implement enforcement of QoS policies by consuming QoS
   key/value pairs used to define QoS type.


Proposed change
===============

Introduce a new top-level object: qos_type.

1. qos_type contains an id, name, description, and a set of qos_specs
   (key/value pairs). The name must be unique.

2. A qos_type name can be specified in share type extra-spec named as
   ``default_qos_type``.

3. qos_type lifecycle management (create, update, delete) will be strictly
   allowed to admin by the virtue of default RBAC policies. All non-admin
   users can only list them.

* qos_type APIs

  qos_type create/delete/update/show/list APIs.

* qos_type_specs APIs

  qos_type_specs create/delete/update/show/list APIs.

* Modify share create API

  In order to allow users selecting the desired QoS policy by the virtue of
  qos_type, the qos_type name must be defined in share-type extra-spec
  ``default_qos_type``. When share is created from such share-type, the
  qos_type and its specs are picked and passed to backend driver. The
  scheduler must validate that backend driver reporting capability
  ``qos_type_support`` will be filtered.

* Changes in backend driver

  The QoS type and its specs are provided to storage backend driver during
  share and share-server creation. Backend driver will create the qos policy
  if its not present and apply to resources.

Alternatives
------------

1. Continue using extra_specs: leads to duplication and inconsistency. The
   user can check QoS characteristics of a share type before provisioning
   a share, but its limited to single QoS policy for resource like share.

2. Driver-only enforcement: lacks visibility at API and DB layers,
   inconsistent across backends.


Data model impact
-----------------


* New field in shares table

  +-----------------------+---------------+------+-----+---------+-------+
  | Field                 | Type          | Null | Key | Default | Extra |
  +=======================+===============+======+=====+=========+=======+
  | qos_type_id           | string(36)    | YES  |     | NULL    |       |
  +-----------------------+---------------+------+-----+---------+-------+


* New table qos_types

  +-----------------------+---------------+------+--------+---------+-------+
  | Field                 | Type          | Null | Key    | Default | Extra |
  +=======================+===============+======+========+=========+=======+
  | id                    | string(36)    |  No  | PRI    |         |       |
  +-----------------------+---------------+------+--------+---------+-------+
  | name                  | string(255)   |  No  | UNIQUE | NULL    |       |
  +-----------------------+---------------+------+--------+---------+-------+
  | description           | string(255)   | YES  |        | NULL    |       |
  +-----------------------+---------------+------+--------+---------+-------+
  | deleted               | string(36)    | YES  |        | NULL    |       |
  +-----------------------+---------------+------+--------+---------+-------+


* New table qos_specs

  +-----------------------+---------------+------+-----+---------+-------+
  | Field                 | Type          | Null | Key | Default | Extra |
  +=======================+===============+======+=====+=========+=======+
  | id                    | int(11)       |  No  | PRI |         |       |
  +-----------------------+---------------+------+-----+---------+-------+
  | qos_type_id           | string(36)    |  No  |     | NULL    |       |
  +-----------------------+---------------+------+-----+---------+-------+
  | key                   | string(255)   |  No  |     | NULL    |       |
  +-----------------------+---------------+------+-----+---------+-------+
  | value                 | string(1023)  |  No  |     | NULL    |       |
  +-----------------------+---------------+------+-----+---------+-------+
  | deleted               | int(11)       | YES  |     | NULL    |       |
  +-----------------------+---------------+------+-----+---------+-------+


CLI API impact
--------------

New OpenStackClient(OSC) commands:

.. code-block:: bash

    openstack share qos type create <name> [--description <desc>]
                                    [--spec <key=value>]

    openstack share qos type list

    openstack share qos type show <qos_type_id>

    openstack share qos type set <qos_type_id> [--description <desc>]
                                 [--spec <key=value>]

    openstack share qos type delete <qos_type_id>


REST API impact
---------------

** Create QoS Type**::

     POST /v2/qos-types

The create request should not be processed if its non-admin user and
the API will respond with ``403 Forbidden``

Request::

    {
        "qos_type": {
            "name": "Gold-QoS",
            "description": "High-performance storage QoS",
            "specs": {
                "expected_iops": "500",
                "peak_iops": "5000",
                "peak_iops_allocation": "used-space",
            }
        }
    }


Response(201)::

    {
        "qos_type": {
            "name": "Gold-QoS",
            "id": "b8b3d83f-96a2-45f2-9e39-ccf3ecf2ff33",
            "description": "High-performance storage QoS",
            "specs": {
                "expected_iops": "500",
                "peak_iops": "5000",
                "peak_iops_allocation": "used-space",
            },
            "created_at": "2025-09-04T10:00:00Z",
            "updated_at": None,
        }
    }


** Show QoS Type**::

    GET /v2/qos-types/{qos_type_id}


Response(200)::

    {
        "qos_type": {
            "name": "Gold-QoS",
            "id": "b8b3d83f-96a2-45f2-9e39-ccf3ecf2ff33",
            "description": "High-performance storage QoS",
            "specs": {
                "expected_iops": "500",
                "peak_iops": "5000",
                "peak_iops_allocation": "used-space",
            },
            "created_at": "2025-09-04T10:00:00Z",
            "updated_at": "2025-09-04T10:00:00Z",
        }
    }


** List QoS Types.**::

    GET /v2/qos-types?{queries}


In detail and index APIs, queries will allow filtering with exact and
inexact ("name", "name~" etc.) attributes.

Response(200)::

    {
        "qos_types": [
            {
                "name": "Gold-QoS",
                "id": "b8b3d83f-96a2-45f2-9e39-ccf3ecf2ff33",
                "description": "High-performance storage QoS",
                "specs": {
                     "expected_iops": "500",
                     "peak_iops": "5000",
                     "peak_iops_allocation": "used-space",
                },
                "created_at": "2025-09-04T10:00:00Z",
                "updated_at": "2025-09-04T10:00:00Z",
            },
            {
                "name": "Silver-QoS",
                "id": "c8b3d83f-96a2-45f2-9e39-ccf3ecf2ff33",
                "description": "Mid-performance storage QoS",
                "specs": {
                     "expected_iops": "300",
                     "peak_iops": "3000",
                     "peak_iops_allocation": "used-space",
                },
                "created_at": "2025-09-04T11:00:00Z",
                "updated_at": "2025-09-04T11:00:00Z",
            }
        ]
    }


** Update QoS Types.**::

    PUT /v2/qos-types/{qos_type_id}

Request::

    {
        "qos_type": {
            "description": "High-performance storage QoS - Update",
        }
    }


The update request should not be processed if its non-admin user and
the API will respond with ``403 Forbidden``

Response(200)::

    {
        "qos_type": {
            "name": "Gold-QoS",
            "id": "b8b3d83f-96a2-45f2-9e39-ccf3ecf2ff33",
            "description": "High-performance storage QoS - Update",
            "specs": {
                "expected_iops": "500",
                "peak_iops": "5000",
                "peak_iops_allocation": "used-space",
            },
            "created_at": "2025-09-04T10:00:00Z",
            "updated_at": "2025-09-04T11:00:00Z",
        }
    }


** Delete QoS Type.**::

    DELETE /v2/qos-types/{qos_type_id}


The delete request should not be processed if its non-admin user and
the API will respond with ``403 Forbidden``. Also if qos_type is in
use by shares, the API will respond with ``400 BadRequest``

Response(204)::

    None


** Create QoS Type Spec**::

     POST /v2/qos-types/{qos_type_id}/specs

The create request should not be processed if its non-admin user and
the API will respond with ``403 Forbidden``

Request::

    {
        "specs": {
            "expected_iops": "500",
            "peak_iops": "5000",
            "peak_iops_allocation": "used-space",
        }
    }


Response(202)::

    {
        "specs": {
            "expected_iops": "500",
            "peak_iops": "5000",
            "peak_iops_allocation": "used-space",
        }
    }


** List QoS Type Specs**::

    GET /v2/qos-types/{qos_type_id}/specs


Response(200)::

    {
        "specs": {
            "expected_iops": "500",
            "peak_iops": "5000",
            "peak_iops_allocation": "used-space",
        }
    }


** Show QoS Type Spec**::

    GET /v2/qos-types/{qos_type_id}/specs/{key}


Response(200)::

    {
        "expected_iops": "500",
    }


** Update QoS Type Spec.**::

    PUT /v2/qos-types/{qos_type_id}/specs

Request::

    {
        "expected_iops": "600",
    }

The update request should not be processed if its non-admin user and
the API will respond with ``403 Forbidden``

Response(200)::

    {
        "expected_iops": "600",
    }


** Delete QoS Type Spec.**::

    DELETE /v2/qos-types/{qos_type_id}/specs/{key}


The delete request should not be processed if its non-admin user and
the API will respond with ``403 Forbidden``.

Response(204)::

    None


Driver impact
-------------
The backend driver needs to implement:

* Driver support qos_type will report ``qos_type_support`` capability.
* The storage backed driver will receive qos_type along-with its specs
  during creation of resources like share or share-server. The driver then
  decide how to use those to create and apply QoS policies on resources.
* Modifying QoS type specifications after shares are created results
  in indeterminate behavior and must be avoided. Modifications are not
  enforced on the storage back end. This means that some drivers can choose
  to apply the modifications only for new shares and leave the existing shares
  as is, and some others may alter the behavior for existing shares as well
  as for new shares. However, this modification is triggered only when a new
  share is created.


Security impact
---------------

The create/update/delete operations for QoS types and specs are available only
for admin users by the virtue of default RBAC. All users can only list and use
QoS type and its specs.


Notifications impact
--------------------

"qos_type.create", "qos_type.update"  and "qos_type.delete" notification events
will be emitted for the respective actions.


Other end user impact
---------------------

None


Performance Impact
------------------

* Minimal overhead when listing QoSTypes.


Other deployer impact
---------------------

When cloud is upgraded to release that supports this feature and administrator
wants to enable this feature, he needs to follow below steps.

1. Create one or more qos_types

2. Create valid specs for qos_type

3. For share-type extra-spec, add extra-spec ``default_qos_type`` and mention
   the qos_type name as value. The share created from such share type will
   consider qos_type assuming driver supports capability ``qos_type_support``.


Developer impact
----------------

Implement feature for  Manila-core as well as reference driver.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
    * kpdev(kinpaa@gmail.com)


Work Items
----------

* Add DB migrations and ORM models.
* Implement DB API functions for CRUD and associations.
* Implement REST API controllers for qos_type and qos_type_specs.
* Add OSC plugin commands.
* Horizon UI updates.
* Tempest and unit test coverage.
* Documentation and release notes.


Future Work Items
-----------------

It is possible to support multiple qos_types under single share-type by
associating them with share-type. If such use-case arises, current spec can
be enhanced.


Dependencies
============

None


Testing
=======

* Unit tests
* Tempest tests


Documentation Impact
====================

- Docstrings
- Devref
- User guide
- Admin guide
- Release notes


References
==========

None
