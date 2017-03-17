..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================
Support IPv6 access in manila
=============================

https://blueprints.launchpad.net/manila/+spec/support-ipv6-access

Problem description
===================

Currently ip access rule type only supports IPv4. A valid format is
xxx.xxx.xxx.xxx or xxx.xxx.xxx.xxx/xx. This is not enough when users want to
enable IPv6 in manila. The most affected areas are API (access rules) and
the drivers (access rules & export locations).

Use cases
=========

End users with IPv6 interfaces also want to access their shares, and manila
currently only supports IPv4 protocol when access is IP-based.

Proposed change
===============

* Support IPv6 in access rule

As agreed by the manila community in IRC meeting (topic IPv6) [1], manila
can address this requirement by adding the IPv6 format validation for ip
access rule type. Change the validation approach from our own function
(_validate_ip_range) to a standard module 'ipaddress' (both IPv4 and IPv6
are supported).

With the ipaddress module, the IPv6 formats below are all considered
acceptable::

      2001:0DB8:02de:0000:0000:0000:0000:0e13   #uncompressed
      2001:DB8:2de:0:0:0:0:0e13                 #leading zeros removed
      2001:DB8:2de::0e13                #consecutive sections of zeroes omitted
      2001:dB8:2De::0E13                #mixed case
      2001:dB8::/32                     #network range

and manila will convert all of these into lower and compressed format, which
means the result of first example will be like (it's also a canonical format
which is recommended in RFC5952 [2])::

      2001:db8:2de::e13

* Support IPv6 in network plugins

The standalone and neutron network plugins (nova network plugins are ignored
as they are being deprecated) will have to be modified to support IPv6.

1. Network plugins will add new boolean configuration options 'network_plugin
   _ipv4_enabled' and 'network_plugin_ipv6_enabled' to indicate its enabled
   ip versions. The default values are 'True' and 'False' respectively. We
   will ultimately support both IPv4 and IPv6 enabled for single network
   plugin, but for this phase, restricted to the current manila network plugin
   architecture, only one option should be configured True (either IPv4 or
   IPv6) in Ocata. We will also deprecate the 'standalone_network_plugin_ip
   _version' which will be equivalent to 'network_plugin_ipv6_enabled=True'
   and 'network_plugin_ipv4_enabled=False', or on the contrary.

2. Network plugins should be updated to allocate network resource based on
   their ip version's configuration (both for user network and admin network).
   For example, the neutron network plugins will only allocate IPv6 ports
   if 'network_plugin_ipv6_enabled = True'. If no suitable resource is found
   with the configured or specified network resource the allocation process
   would fail with exception.

* Support IPv6 in drivers

As the export location support's status varies in different drivers, manila
will introduce two extra_specs (and driver capabilities) 'ipv4_support',
'ipv6_support' to make backends and users easy to support and configure.

1. Add new optional boolean extra_specs 'ipv4_support', 'ipv6_support' for
   share type that indicates the required ip version (Omitting either
   extra_spec means "don't care" about a match for that capability).

2. Drivers have to report the capabilities with the ip version which they can
   support, the capability is reported in the same format with extra_specs.
   There are two cases that driver should consider:
   - **DHSS=false**, the driver should report IPv4 is True only if it is
   actually configured with IPv4 export locations, and IPv6 is True only
   if it is actually configured to support IPv6 export locations.
   - **DHSS=true**, the driver should report IPv4 is True only if both driver
   and the configured network plugins (both user and admin) have the IPv4
   capability, so does the IPv6.
3. The manila scheduler will choose the correct backend with capability
   filter when ip version extra_specs are involved.
4. Driver must return export locations according to its IP version
   capabilities, and only host addresses are allowed in export locations,
   not hostnames.


Alternatives
------------

None

Data model impact
-----------------

None

REST API impact
---------------

The API microversion will have to be bumped for this.

* The IPv6 address and IPv6 network are supported in allow_access action.

Driver impact
-------------

Drivers will be updated so that existing drivers that use IPv4 addresses
will report ipv4_support=True. When drivers support both IPv4 and IPv6,
they need report their support versions based on their capabilities
and configuration of the storage backend.
Also the following functions might need to be changed:

* _update_share_stats
* create_share
* delete_share
* create_share_from_snapshot
* update_access
* setup_server
* manage_existing
* create_replica
* promote_replica
* migration_start
* migration_complete
* create_consistency_group
* create_consistency_group_from_cgsnapshot

Also, drivers must export locations as addresses rather than as host names
since addresses are specifically IPv4 or IPv6 whereas host names could resolve
to either an IPv4 or an IPv6 address at any particular time.

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

* python-manilaclient will be changed to reflect the validation approach
  changes for ip access type.

  The access-allow command with IPv6 supported will be like::

      manila access-allow test_share ip "AD80:0:0:0:ABAA:0:C2:2"

* manila-ui will have corresponding UI changes to deal with the valid value
  for ip access rule type after python-manilaclient implementation.


Performance impact
------------------

None

Other deployer impact
---------------------

Deployers must ensure their IP version preferences are reflected on the
storage backends, such that data interfaces have IPv4 and IPv6 addresses
as desired.

Developer impact
----------------

Changes to the driver interface are noted above. Third party backends
will implement these changes and report their IPv6's ability if they
want to enable IPv6 in their drivers (the support matrix can be found
here [3]).


Implementation
==============

Assignee(s)
-----------

Primary assignee:

* zhongjun(jun.zhongjun2@gmail.com)
* TommyLike(tommylikehu@gmail.com)
* zengyingzhe(zengyingzhe@huawei.com)

Work items
----------

* Implement core feature
* Report capability ipv4_support=True for existing IPv4 drivers
* Implement IPv6 in network plugins.
* Implement IPv6 ACL in generic, huawei and dummy drivers
* Implement tempest tests
* Implement Scenario tests
* Implement manila-ui support
* Implement python-manilaclient support
* Update manila API documentation
* Update manila guide documentation on IPv6

Dependencies
============

None

Testing
=======

* Unit test
* Functional test
* Tempest test

Documentation impact
====================

* Update user guide
* Update manila API documentation
* Update manila developer documentation on IPv6

References
==========

[1] http://eavesdrop.openstack.org/meetings/manila/2016/manila.2016-08-25-15.00.log.html
[2] https://tools.ietf.org/html/rfc5952#section-4
[3] http://docs.openstack.org/developer/manila/devref/share_back_ends_feature_support_mapping.html