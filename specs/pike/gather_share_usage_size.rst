..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================
Share usage size tracking
=========================

https://blueprints.launchpad.net/manila/+spec/share-usage-size

We need to have a way to gather information about actual share storage usages,
so cloud operators could use this information for billing, health checks
and/or other purposes.

Problem description
===================

Currently, it is impossible to get actual storage usage of some specific
share, hence, it is impossible to bill for storage usage. It is only possible
to bill for quota limits.

Use cases
=========

* Manila could report storage usages to metering services like ceilometer [1].
* Some storage backends do not reserve whole requested share size right away
  after share creation and just set quota/limit for shares, for such backends
  it may be more suitable to bill for usage, not limit/quota.

Proposed change
===============

Add the periodic task in share service for gathering the shares usage size.
The periodic task will be invoked after the share manager starts up. We
can set the interval time that determines how often the share manager will
poll the driver to perform the next step of get share usage size.
Administrators and driver developers can "opt-in" to enable this feature
in their drivers and in their clouds.

Add notifiers for shares usage size we want to measure. Right now we intend
to add notifiers to resources (such as: shares, share groups, snapshots and
etc) and publish the resources we expect [1].

The update_share_usage_size() driver method is a new driver method which will
be called after driver initialization and in the periodic task, that returns a
dictionary about shares id and shares usage size. The shares usage size value
info will be notified to ceilometer or other project, otherwise the share info
doesn't need to be notified.

Alternatives
------------

* Why we don't gather the space usage when we need to use it (such as: list,
  show APIs)?

  Because we could publish such real time information to ceilometer.

* Why we don't store the space usage?

  Because we can't implement this inside manila without duplicating
  functionality already in other projects like ceilometer.
  It's a number that changes over time. If the number is too old, it might
  be wildly inaccurate. So we're going to push the storage/retrieval parts
  of the problem outside of manila to keep manila's scope small and focused.

Data model impact
-----------------

None

REST API impact
---------------

None

CLI impact
----------

None

Driver impact
-------------

* Get real share provisioned capacity from driver::

    def update_share_usage_size(self, context, shares):
        """Invoked to get the usage size of given shares.

        Driver can use this method to update the share usage size of
        the shares. To do that, a dictionary of shares should be
        returned.
        :shares: None or a list of all shares for updates.
        :return None or a dictionary of updates in the format::

            {
                '09960614-8574-4e03-89cf-7cf267b0bd08': {
                    'used_size': '200',
                },
            }

        """
        raise NotImplementedError()

If an administrator has configured a backend for monitoring and the driver
cannot support the update_share_usage_size method, then the share manager will
catch the raise exception (NotImplementedError) from the driver, and we will
later not invoke the periodic task to update share usage size for shares of
this backend. We will LOG that it is unsupported instead.

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance impact
------------------

Collecting share usage have a negative service performance impact.
When the size of a share is known at the moment of creation and does not
change without an explicit extend or shrink, we don't have to collect usage
at all. Now we have to collect usage. It could be costly.

Administrators may set the periodic interval configuration option value to
a large interval or disable it if necessary. It could be less costly or not
costly.

Other deployer impact
---------------------

We will need to add an option for enabling and disabling sending notifications.
This flag will affect manila deployed with any backend.
By default, this option will be set to false and manila won't be sending
notifications, keeping the behavior it currently has.

If operators want to start retrieving metrics on the manila shares usage size,
they will need to enable sending notifications on the configuration file after
upgrade to or installation of Pike. They also need to make sure they have a
specified version of ceilometer or other software deployed. After this, the
feature here specified will be available for use.

Developer impact
----------------

This will require a change on the driver interface. To support this feature,
drivers will need to implement the ``update_share_usage_size`` feature.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

* zhongjun(jun.zhongjun2@gmail.com)

Work items
----------

* Implement the core feature with functional tempest test
  coverage
* Add documentation of this feature

Dependencies
============

None

Testing
=======

* Unit tests
* Functional tempest tests

Documentation impact
====================

- Devref
- API reference
- User guide and Admin guide

References
==========

[1] https://specs.openstack.org/openstack/manila-specs/specs/pike/ceilometer-integration.html
