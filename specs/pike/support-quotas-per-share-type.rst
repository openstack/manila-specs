..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
Support quotas per share type
=============================

Blueprint: https://blueprints.launchpad.net/manila/+spec/support-quotas-per-share-type

Currently, quotas in Manila are configurable on a per tenant and per user
basis. They allow admins to set the values for the number of shares,
the number of snapshots, or the amount of gigabytes for a project and
allow a project member to use resources within these given limits.
This spec describes the extension of the current quota scheme: in addition to
an overall quota for, say, the number of shares, admins will have
the additonal option to define the quota for the number of shares
for a given share type. This functionality is similar
to the one offered by Cinder for quotas per volume type.

Problem description
===================

Quotas are meant to help administrators to control the provisioning of
available resources, such as the number of shares or the space shares
take up on the configured backends. While Manila supports quotas,
the current quota scheme does not take share types into account.
Considering setups with multiple backends (accessible via different
share types), administrators have hence no means to control
the resource consumption on the various backends.

Use Cases
=========

In setups with multiple share-types, per share type quota will allow
resource provider to have better control over the provisioned resources.

Proposed change
===============

The proposal is to add per share type quotas in addition to project and
user quotas. When a resource is requested, the share type specific quota is
checked first. If this check passes, the request is validated against
the project/user quota for that resource. It will be possible
to set share type quotas only per project, but not per user in scope of
single project. It is intended restriction to avoid creation of
one more quota layer (third), that will be useful only in case when we need to
set different share type quotas for users that are related to
one single project.

Alternatives
------------

An alternative would be to make all share types private, require users
to create a new tenant for each share type they'd like to access, and then
grant access to the corresponding tenants. This will become impracticable
for the users when the number of share types goes up, or when quota schemes
for other projects and resources would follow the same approach.

.. _Cinder's implementation: https://review.openstack.org/#/c/25059/

Other possible solution is to port `Cinder's implementation`_. Where
instead of additional quota layer Cinder creates additional quota
resources for each created volume type and names them using following
template::

  %RESOURCE_NAME%_%VOLUME_TYPE_NAME%

But this approach provides two inconveniences:

  * "quota-show" command's response grows with creation of
    each new share/volume type
  * names of new share-type-based quota resources can be confusing
    depending on names of share types. For example, share type 'default' and
    'shares' quota resource will produce additional quota resource
    named 'shares_default' in parallel to existing 'shares' project quota
    resource that is not really intuitive for understanding.

Data model impact
-----------------

New share type quotas will have its own DB model similar to existing DB model
that is used for "user" quotas with the same relation to "project" quotas.
So, it will behave completely the same way as "user quotas". We will have
following structure:

.. graphviz::

   digraph quotas {
       center=true;

       node_pr [label = "Project Quotas", shape = ellipse, width = 2.2];
       node_prs[label = "<f0> PROJECT_1|<f1> PROJECT_2|<f2> \.\.|<f3> PROJECT_N", shape = record, fontsize = 10];
       "node_prs":f0 -> "node_pr" [dir=back];
       "node_prs":f1 -> "node_pr" [dir=back];
       "node_prs":f3 -> "node_pr" [dir=back];

       node_u [label = "Users  Quotas", shape = ellipse, width = 2.2];
       node_us[label = "<f0> USER_1|<f1> USER_2|<f2> \.\.|<f3> USER_N", shape = record, fontsize = 10];
       "node_u" -> "node_us":f0;
       "node_u" -> "node_us":f1;
       "node_u" -> "node_us":f3;

       node_st [label = "Share Type Quotas", shape = ellipse, width = 2.2];
       node_sts[label = "<f0> ST_1|<f1> ST_2|<f2> \.\.|<f3> ST_N", shape = record, fontsize = 10];
       "node_st" -> "node_sts":f0;
       "node_st" -> "node_sts":f1;
       "node_st" -> "node_sts":f3;

       "node_pr" -> "node_u";
       "node_pr" -> "node_st";
   }


REST API impact
---------------

Three (3) quota APIs from four (4) will be changed - 'update', 'delete' and
'get' API. 'defaults' will stay unchanged and will return project default
quotas as they will be default for each share type.
All three APIs will start handling additional 'share_type' GET argument,
which will be mutually exclusive with existing 'user_id' argument.
Response structure will stay the same, with the only exception -
quota 'share_networks' will be absent in 'quota-show' response body, because
this kind of quota has no value in case of share types. Also, it will not be
possible to set/update this quota per share type.
New 'share_type' argument should be able to accept both - name and UUID of
a share type. API changes will require microversion bump.
Old microversions will behave as did before.

* Get quotas

  * URL: /quota-sets?share_type=%name_or_id%
  * Method: GET
  * Response Body::

      {
          'quota_set': {
              'shares': -1,
              'gigabytes': -1,
              'snapshots': -1,
              'snapshot_gigabytes': -1,
          }
      }

  .. note::

      'share_networks' quota is absent in response,
      because it is user/project quota only.


* Delete quotas

  * URL: /quota-sets?share_type=%name_or_id%
  * Method: DELETE


* Update quotas

  * URL: /quota-sets?share_type=%name_or_id%
  * Method: PUT
  * JSON body:

    .. code-block:: json

        {
            'quotas': {
                'shares': -1,
                'gigabytes': -1,
                'snapshots': -1,
                'snapshot_gigabytes': -1,
            }
        }

Driver impact
-------------

None.

Security impact
---------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

The Manila client will get a new optional '--share-type' argument for the
'quota-update', 'quota-show' and 'quota-delete' commands,
so the modified version of the client will be::

  $ quota-update ... [--share-type <name_or_id>] ... <tenant_id>
  $ quota-show ... [--share-type <name_or_id>] ... --tenant <tenant_id>
  $ quota-delete ... [--share-type <name_or_id>] ... --tenant <tenant_id>

When the '--share-type' argument is omitted, the project quota will be updated,
just as now. If the '--share-type' argument is passed, the quota of the
corresponding resource(s) for the specified share type will be set.

Performance Impact
------------------

Additional layer of quota usages calculation will slow down things at
some level.

Other deployer impact
---------------------

When creating new share types, the initial quotas for the new share type will
be the project quota.
Hence, for new tenants, the initial quota must be set for the share types
if they shall differ from the project quota.

Developer impact
----------------

Developers of new features that consume quota will need to provide additional
"share_type_id" argument to 'reserve', 'commit' and 'rollback' quota commands.
Underlying DB layer will handle both quota layers automatically.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

* vponomaryov ( vponomaryov@mirantis.com )

Work Items
----------

* Server side API changes
* Manila Client side changes
* Manila UI side changes

Dependencies
============

None.

Testing
=======

* unit tests in 'manila', 'python-manilaclient' and 'manila-ui' projects
* functional tests in 'manila' and 'python-manilaclient' projects

Documentation Impact
====================

* admin guide: add new '--share-type' parameter to 'update-quota',
  'quota-show' and 'quota-delete' commands.
* devref: reflect the proposed change
