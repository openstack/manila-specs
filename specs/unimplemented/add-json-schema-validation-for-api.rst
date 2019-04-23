..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==============
API Validation
==============

https://blueprints.launchpad.net/manila/+spec/json-schema-validation

Currently, Manila has different implementations for validating request bodies.
The purpose of this blueprint is to track the progress of validating the
request bodies sent to the Manila server, accepting requests that fit the
resource schema and rejecting requests that do not fit the schema. Depending
on the content of the request body, the request should be accepted or rejected
consistently.


Problem description
===================

Currently Manila doesn't have a consistent request validation layer. Some
resources validate input at the resource controller and some fail out in the
backend. Ideally, Manila would have some validation in place to catch
disallowed parameters and return a validation error to the user.

The end user will benefit from having consistent and helpful feedback,
regardless of which resource they are interacting with.


Use Cases
=========

As a user or developer, I want to observe consistent API validation and values
passed to the Manila API server.


Proposed change
===============

One possible way to validate the Manila API is to use jsonschema similar to
Nova, Cinder, Keystone and Glance (https://pypi.python.org/project/jsonschema).
A jsonschema validator object can be used to check each resource against an
appropriate schema for that resource. If the validation passes, the request
can follow the existing flow of control through the resource manager to the
backend. If the request body parameters fails the validation specified by the
resource schema, a validation error wrapped in HTTPBadRequest will be returned
from the server.

Example:
"Invalid input for field 'name'. The value is 'some invalid name value'.

Each API definition should be added with the following ways:

* Create definition files under ./manila/api/schemas/.
* Each definition should be described with JSON Schema.
* Each parameter of definitions(type, minLength, etc.) can be defined from
  current validation code, DB schema, unit tests, Tempest code, or so on.

Some notes on doing this implementation:

* Common parameter types can be leveraged across all Manila resources. An
  example of this would be as follows::

    # share create schema

    manila/api/schema/shares.py:

    from manila.api.validation import parameter_types
    create_v231 = {
        'type': 'object',
        'properties': {
            'share': {
                'type': 'object',
                'properties': {
                    'description': parameter_types.description,
                    'share_type': {
                        'format': 'uuid'
                    },
                    'share_proto': parameter_types.proto
                    'share_network_id': {
                        'format': 'uuid'
                    },
                    'share_group_id': {
                        'format': 'uuid'
                    },
                    'name': parameter_types.name,
                    'snapshot_id': {
                        'format': 'uuid'
                    },
                    'size': parameter_types.positive_integer,
                    'metadata': {
                        'type': 'object'
                    },
                },
                'required': ['size'],
                'additionalProperties': False,
            }
            'required': ['share'],
            'additionalProperties': False,
        }
    }

    manila/api/validation/parameter_types.py:

    description = {
        'type': 'string', 'minLength': 0, 'maxLength': 255,
    }

    name = description

    # This registers a FormatChecker on the jsonschema module.
    # It might appear that nothing is using the decorated method but it gets
    # used in JSON schema validations to check uuid formatted strings.
    from oslo_utils import uuidutils

    @jsonschema.FormatChecker.cls_checks('uuid')
    def _validate_uuid_format(instance):
        return uuidutils.is_uuid_like(instance)

    positive_integer = {
        'type': ['integer', 'string'],
        'pattern': '^[0-9]*$', 'minimum': 1, 'minLength': 1
    }

    proto = {
        'type': ['string'],
        'enum': ['NFS', 'CIFS', 'GlusterFS', 'HDFS', 'CephFS']
    }

* The validation can take place at the controller layer using below decorator::

    manila/api/v2/shares.py:

    from manila.api.schemas import share

    @wsgi.Controller.api_version("2.31")
    @validation.schema(shares.create_v231)
    def create(self, req, body):
        """creates a share."""

* Initial work will include capturing the Shared File API Spec for existing
  resources in a schema. This should be a one time operation for each
  major version of the API. This will be applied to the Shared File V2 API.

* When adding a new extension to Manila, the new extension must be proposed
  with its appropriate schema.


Alternatives
------------

Before the API validation framework, we need to add the validation code into
each API method in ad-hoc. These changes would make the API method code messy
and we needed to create multiple patches due to incomplete validation.

If using JSON Schema definitions instead, acceptable request formats are clear
and we don’t need to do ad-hoc works in the future.

Pecan Framework:
`Pecan <http://pecan.readthedocs.org/en/latest/>`_
Some projects(Ironic, Ceilometer, etc.) are implemented with Pecan/WSME
frameworks and we can get API documents automatically from the frameworks.
In WSME implementation, the developers should define API parameters for
each API. Pecan would make the implementations of API routes(URL, METHOD) easy.


Data model impact
-----------------

None


REST API impact
---------------

In Manila, since Liberty, we strive to maintain complete API
backwards-compatibility. So, if we have a client that's talking to different
manila releases, they should experience the same behavior if using the same
API version. We break this rule rarely, but have consistently maintained
that if we were correcting buggy behavior, we will consider breaking
backwards-compatibility.

API Response code changes:

There are some occurrences where API response code will change while adding
schema layer for them. For example, On current master 'services' table has
'host' and 'binary' of maximum 255 characters in database table. While updating
service user can pass 'host' and 'binary' of more than 255 characters which
obviously fails with 404 ServiceNotFound wasting a database call. For this we
can restrict the 'host' and 'binary' of maximum 255 characters only in schema
definition of 'services'. If user passes more than 255 characters, he/she will
get 400 BadRequest in response.

API Response error messages:

There will be change in the error message returned to user. For example,
On current master if user passes more than 255 characters for share name
then below error message is returned to user from manila-api:

Invalid input received: name has <actual no of characters user passed>
characters, more than 255.

With schema validation below error message will be returned to user for this
case:

Invalid input for field/attribute name. Value: <value passed by user>.
'<value passed by user>' is too long.


Security impact
---------------

The output from the request validation layer should not compromise data or
expose private data to an external user. Request validation should not
return information upon successful validation. In the event a request
body is not valid, the validation layer should return the invalid values
and/or the values required by the request, of which the end user should know.
The parameters of the resources being validated are public information,
described in the Shared File API spec, with the exception of private data.
In the event the user's private data fails validation, a check can be built
into the error handling of the validator to not return the actual value of the
private data.

jsonschema documentation notes security considerations for both schemas and
instances:
http://json-schema.org/latest/json-schema-core.html#anchor21

Better up front input validation will reduce the ability for malicious user
input to exploit security bugs.


Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

Manila will pay some performance cost for this comprehensive request
parameters validation, because the checks will be increased for API parameters
which are not validated now.


Other deployer impact
---------------------

None


Developer impact
----------------

This will require developers contributing new extensions to Manila to have
a proper schema representing the extension's API.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
dongdongpei (Dongdong Pei <peidongdong121@163.com>)

Other contributors:
wosunoozzy (Yang Zhang <wosunoozzy@gmail.com>)

Work Items
----------

1. Initial validator implementation, which will contain common validator code
   designed to be shared across all resource controllers validating request
   bodies.
2. Introduce validation schemas for existing core API resources.
3. Introduce validation schemas for existing API extensions.
4. Enforce validation on proposed core API additions and extensions.
5. Remove duplicated ad-hoc validation code.
6. Add unit and end-to-end tests of related API's.
7. Add/Update Manila documentation.

Dependencies
============

The code under manila/api/v1 are getting called by v2. So if we add schema
validation for v2 then we will have to remove the existing validation of
parameters which is there inside of controller methods which will again
break the v2 apis.

Solution:

1. Do the schema validation for v2 apis using the @validation.schema decorator
   similar to Nova and also keep the validation code which is there inside of
   method to keep v1 working.

2. Once the decision is made to remove the support to v1 we should remove the
   validation code from inside of method.


Testing
=======

Tempest tests can be added as each resource is validated against its schema.
These tests should walk through invalid request types.

We can follow some of the validation work already done in the Nova V3 API:

* `Validation Testing <https://opendev.org/openstack/tempest/src/commit/24eb89cd3efd9e9873c78aacde804870962ddcbb/etc/schemas/compute/flavors/flavors_list.json>`_

* `Negative Validation Testing <https://opendev.org/openstack/tempest/src/commit/b2978da5ab52e461b06a650e038df52e6ceb5cd6/tempest/api/compute/flavors/test_flavors_negative.py>`_

Negative validation tests should use tempest.test.NegativeAutoTest


Documentation Impact
====================

1. The Manila API documentation will need to be updated to reflect the
   REST API changes.
2. The Manila developer documentation will need to be updated to explain
   how the schema validation will work and how to add json schema for
   new API's.


References
==========

* Understanding JSON Schema:

  https://json-schema.org/understanding-json-schema/index.html

* Nova Validation Examples:

  https://opendev.org/openstack/nova/src/branch/master/nova/api/validation

* JSON Schema on PyPI:

  https://pypi.python.org/project/jsonschema

* JSON Schema core definitions and terminology:

  https://tools.ietf.org/html/draft-zyp-json-schema-04

* JSON Schema Documentation:

  https://json-schema.org/specification.html
