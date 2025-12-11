..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.
 http://creativecommons.org/licenses/by/3.0/legalcode

====================================
Allow force deletion of used subnets
====================================

Allow deletion of a subnet of a share network used by one or multiple shares.
This action is possible only if subnet is not the last subnet of the share
network, as it is forbidden to have a network without any subnet.

Problem description
===================
When a share network is attached to a share, manila API allows adding a new
subnet to this network, but not deleting it. This means Manila users can add
as many network interfaces as they want to their share servers, but these
interfaces could never be deleted.

Use Cases
=========
Requirements of clients using a share might change that requires a completely
different network configuration of the share server. For example, it might be
used from a public network, and then from a private network. In this case, as
public IP addresses are limited resources, we want them to be released.
Keeping unused network interfaces on a share server may also lead to security
issues. Therefore, Manila should provide a way to delete such interfaces or
subnets.

Proposed change
===============
* Don't delete share server on share network subnet deletion.
  A share network might have multiple subnets. So when deleting one of its
  subnets, the share will still need a share server.

* Allow removing network interface on share server.
  Currently in Manila, there is an API to add network interfaces, but no API
  allows deleting network interfaces.

* Add a `delete_share_network_subnet` method to share API, RPC API and manager.
  In the manager, this method relies on
  `update_share_server_network_allocations` with `remove=True`. After the
  allocation update, if no allocation remains, the share server is deleted.

* Add `delete` parameter to method
  `update_share_server_network_allocations` of share manager and driver (that
  defaults to False). When set to True, allocations are removed from the share
  server instead of added.

REST API impact
---------------
1. Add a new action endpoint to force subnet deletion that would be
   admin-only by virtue of default RBAC rules.

  * URL:

    /v2/share-networks/{share_network_id}/subnets/{sns_id}/action

  * Method: POST

  * JSON body:

    .. code:: python

      {
        'force_delete': None
      }

  * This API is restricted to administrators only by virtue of default
    RBAC rules.

  * When deleting a subnet through this action, check for existence of
    attached shares is not performed.

  * Possible response codes:
    - 202 - Request accepted.
    - 403 - Forbidden (non-admin access)
    - 404 - Unrecognized share network ID
    - 404 - Unrecognized share network subnet ID
    - 404 - API does not exist in requested microversion
    - 409 - Conflict (e.g., trying to delete the last subnet)

2. Introduce a "check-only" mode when attempting to delete a subnet from
   a share network. This mode would allow users to evaluate whether
   a subnet can be safely removed from all associated share servers,
   without actually performing the deletion. This endpoint is also admin-only
   by virtue of default RBAC rules.

   To keep consistency with subnet creation check, this endpoint accept a
   `reset_operation` parameter. If set to false API returns cached result.
   If set to true, it triggers a new asynchronous check operation.

  * URL:

    /v2/share-networks/{share_network_id}/subnets/{sns_id}/action

  * Method: POST

  * JSON body:

    .. code:: python

      {
        'share_network_subnet_delete_check': {
          'reset_operation': false
        }
      }

  * This API is restricted to administrators only by virtue of default
    RBAC rules.

  * Possible response codes:
    - 202 - Request accepted.
    - 403 - Forbidden (non-admin access)
    - 404 - Unrecognized share network ID
    - 404 - Unrecognized share network subnet ID
    - 404 - API does not exist in requested microversion
    - 409 - Conflict (e.g., trying to delete the last subnet)

End-user impact
---------------
None. Default behaviour remains the same. By specifying the `force` parameter,
users indicate that they are aware of consequences and take responsibility for
eventual disruption. This API is only available to administrators.

Implementation
==============
Assignee(s)
-----------
Primary assignee:
    * sylvanld (sylvan.le-deunff@ovhcloud.com)

Work Items
----------
* Implement the force-delete share subnet API.
* Implement the corresponding functionality in SDK.
* Implement tempest tests.
* Update the documentation.

Testing
=======
* Unit tests
* Tempest tests

Documentation Impact
====================
- User guide
- Admin guide
- API reference
