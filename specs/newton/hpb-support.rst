..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

====================================================================
manila networking support for hierarchical port bindings for neutron
====================================================================

https://blueprints.launchpad.net/manila/+spec/manila-hpb-support

Many manila drivers are capable of supporting VLAN networking but this
technology limits the number of actual networks in the cloud to 4096. Other
overlay technologies are often not supported by vendor drivers. With HPB
(Hierarchical port binding) this barrier can be reduced by using multiple (in
the hierarchy of the physical network) port bindings. For example this allows
the usage of VXLAN on top of VLAN. In general this is transparent for the
underlying storage since this hierarchical binding is all done by neutron and
in the end it's just a VLAN that will be visible to the backend storage.

Problem description
===================

Manila with the current design (manila/network) uses neutron only to reserve
an IP and to retrieve the dedicated segmentation id (VLAN ID) out of it. The
port itself stays in status `inactive`.

This feature will cover three aspects of implementation:

 * The support of neutron port binding in manila
 * The support of `baremetal` provisioning
 * The support of multi-segment network / HPB support


Use Cases
=========

As admin I want to create a share-network with share-server that will be
automatically provisioned to the underlaying network infrastructure. This
covers neutron ML2 private and provider networks and multi-segmented networks
(like HPB networks).

This feature is only useful for drivers with DHSS support (DHSS=True).
Otherwise networking setup is a manual work an admin needs to do.

Proposed change
===============

manila should be able to benefit from the neutron port-binding extension [1].
This API extension allows ports to do a real binding also with physical port
configuration on a switch.

Enabling port binding support
-----------------------------

To enable port binding it's necessary to specify one additional field
within the neutron port create request: ``binding:host_id``

The host_id is often used in neutron ML2 drivers to identify the agent
that can do the binding. An agent is not necessarily needed for manila case,
so the host_id should be set to a value that is not managed by an
OVS-agent. It can be set to the name (or IP address) of the storage box.

While binding the port, neutron will iterate through all ML2 mech drivers.
It's important that one of the drivers signals that the binding can be
fulfilled. Available mech drivers from Cisco [2] and upcoming Arista mech
driver already support binding for such cases.

Part of the feature is also a small neutron manila mech driver. Which can
fulfill the binding generically. This also enables drivers that do a partial
binding and would work in neutron also in the gate.

Baremetal vNIC type / Ironic ML2 integration
--------------------------------------------

Ironic has a very similar problem when connecting physical devices/servers
to a neutron managed network. This feature can reuse the ML2 Ironic
interface described here: [3]

To reuse the feature, the vnic_type must be set to ``baremetal`` during
port creation. Furthermore it's needed to add some network information to the
port create, like::

    "binding:profile": {
        "local_link_information": [{
            "switch_id": 00-12-ff-e1-0d,
            "port_id": dd013:12:33:4,
            "switch_info": {"switch_ip": 10.0.0.1}
        }]
    }

This can be done with static configuration in ``manila.conf`` per backend::

    [manila_storage_drv1]
    port_binding_profiles=phy_conn1,phy_conn2

    [phy_conn1]
    switch_id = 00-12-ff-e1-0d
    port_id = dd013:12:33:4
    switch_info = switch_ip=10.0.0.1

Multi-segment binding
---------------------

A multi-segment binding behaves differently than binding a single segment
network. API-wise a single segment looks like::


    $ neutron net-show <>

    provider:network_type: vlan
    provider:physical_network: mynet1
    provider:segmentation_id: 1029

A multi-segment network looks like::

    $ neutron net-show <>

    segments: [
        provider:network_type: vxlan, provider:segmentation_id: 123, ..
        provider:network_type: vlan, provider:segmentation_id: 544, ..
        ]

It's also possible for mech drivers to dynamically allocate segments during
binding.

For manila, this means, the ports must be created before identifying the
used segment. This can be done with a dedicated neutron-manila-mech driver
that adds needed information in ``binding:vif_details`` or by using the
``physical_network`` field in manila configuration.

Later, neutron should support an API interface to retrieve the binding
information in a better way. This work will be tracked here: [5]

Alternatives
------------

Introduce a neutron ML2 agent that does the actual binding following the
concept that neutron is doing all network related actions. This would
mean all the agent needs to have a driver concept to support multiple
vendors and APIs.

Data model impact
-----------------

None.

REST API impact
---------------

None.

Security impact
---------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

None.

Performance impact
------------------

The share server creation will take longer since manila needs to wait for
the neutron port to become active.
This can be enhanced later, e.g. by introducing multi-processing and proceeding
with share server creation like Nova is doing.

Other deployer impact
---------------------

Configuration files need to be enhanced to activate the feature.
Old functionality / old configuration will work as before.


Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  m-koderer

Other contributors:

  * tpatzig
  * dgonzalez

Work Items
----------

 * The support of neutron port binding in manila
 * The support of ``baremetal`` provisioning
 * The support of multi-segment network / HPB support
 * Adding a manila mech driver in contrib

Dependencies
============

None.


Testing
=======

Code will be tested by unit tests.
Functional testing must be done in a separate CI job:

 * Binding can be tested using a manila mech driver
 * Multi-segment (only static) can potentially be tested using a neutron
   network

A potential test candidate could be the container driver [6], since it needs
a binding done by neutron. A full end-to-end test with dynamic multi-segments
would need a 3rd-party CI. Currently in discussion is to add those tests in
Netapp-CI system.


Documentation Impact
====================

Documentation for new configuration switches and possible deployment types.

References
==========

* [1]: http://developer.openstack.org/api-ref-networking-v2-ext.html#createPort

* [2]: https://github.com/openstack/networking-cisco

* [3]: https://specs.openstack.org/openstack/ironic-specs/specs/not-implemented/ironic-ml2-integration.html

* [4]: https://github.com/openstack/networking-cisco

* [5]: https://bugs.launchpad.net/neutron/+bug/1573197

* [6]: https://review.openstack.org/#/c/308930/
