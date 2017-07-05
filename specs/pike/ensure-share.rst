..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============
Ensure share
============

https://blueprints.launchpad.net/manila/+spec/ensure-share

It's not reasonable to update all shares on every share manager when we
restart manila share service. This driver interface is currently being used
wrong and needs a rewrite. This spec adds the ability to solve the potential
problem of slow start up, and deal with non-user-initiated state changes to
shares.

Problem description
===================

Manila-share service has share driver interface called "ensure_share",
that, depending on share driver, performs some actions on existing shares.
This method is called on each manila-share service start and none of RPC
requests are handled while this method does not finish processing existing
shares. It makes life of cloud administrator painful, because each
manila-share service restart takes more and more time with growth of shares
amount. Also, in most cases, this processing is redundant.

Use cases
=========

Consider the following reasonable use cases:

* Manila-share node going to be shut down for maintenance and then enabled
  back when we want to upgrade software or something else. In this case
  admin expects fast start of service, no matter how many shares do exist
  and managed by manila, and no matter they change any config options or not.

* Allows admins to update shares automatically when the admin restarts the
  service and they changed some config options or updates storage somehow
  (such as: we have an IPv4-only controller and we want to add an IPv6 address
  (any export location)).

In the future we will solve following use cases:

* Allows admins to automatically tag some shares or shares belonging to
  particular backends as needing 'ensure'.

* Allows admins to update other resouces(such as: snapshot, share group,
  etc) for share services when the admin restarts the service.

* Allows admins to update shares and don't need to restart manila share
  services when the admin changes something on array (such as: export
  location IP). (depends on other spec)

Proposed change
===============

To accomplish this change, we will do the following:

When the share manager starts up, it will call the get_backend_info() driver
method to obtain a dictionary of values which could affect shares. The share
manager will compute a hash of the returned dictionary and compare that hash
to the previously computed value. If the value hasn't changed
then the ensure_shares logic will be skipped, otherwise the ensure_shares logic
will execute as normal but the new hash value will be stored in the DB.

The get_backend_info() driver method is a new driver method which will be
called after driver initialization but before ensure_shares, that returns a
dictionary. Drivers will have complete control of the contents of the
dictionary to be hashed. Driver can include any of the following:

1. The ID of the most recent DB migration. Including this value guarantees
   that ensure_shares will be called after DB schema changes.
2. Any of the keys and values in the oslo config object. Including these
   values guarantees that ensure_shares is called whenever the config file
   changes in a way that impacts the driver.
3. Hardcoded values inside the driver, such as version numbers. Including
   these values allows the driver to force ensure_shares to be called when
   a code changes occurs.
4. Values collected from the storage controller itself. In case the
   administrator upgrades or reconfigures the storage controller, this allows
   Manila to detect the change and to run ensure_shares, allowing Manila to
   learn any new important details or perform any needed maintenance.

The reason for hashing the values is because Manila doesn't care what the old
values were, only that something changed. Storing a hash of the values is
simpler from a DB schema perspective. In order to avoid accidental hash
collisions MD5 will be used to generate 128 bits worth of data. Dictionaries
will be sorted lexicographically by key, and all values will be converted to
strings, then UTF-8 encoded into bytes, and fed into the hash algorithm.

The ensure_share() method will be replaced by a method called ensure_shares()
which takes a list of shares in and returns a dictionary of model updates.
This is because there are no cases where it makes sense to call ensure_share()
on less than all of the shares on a particular backend (or share_server, if
DHSS=true). Combining all the calls into one makes it easier for driver
implementers to reuse data that will be common to all shares.

Alternatives
------------

None

Data model impact
-----------------

We will add a new table and model to store the hash information per-backend.

The model will be called manila.db.sqlalchemy.models.BackendInfo and will
contain 2 columns: host and info_hash. The host field will be just the
"hostname" of the backend with no pool suffix.

REST API impact
---------------

None

Driver impact
-------------

Add driver interfaces::

    def get_backend_info(self, context):
        """Get driver and array configuration parameters and return for
           assessment.
        :return A dictionary containing driver-specific info::
             {
              'version': '2.23'
              'port': '80',
              'logicalportip': '1.1.1.1',
               ...
             }
        """

Replace ensure_share() with ensure_shares()

    def ensure_shares(self, context, shares, share_server=None):
        """Invoked to ensure that shares are exported.

        Driver can use this method to update the list of export locations of
        the shares if it changes. To do that, a dictionary of shares should be
        returned.
        :shares: None or a list of all shares for updates.
        :return None or a dictionary of updates in the format::

            {
                '09960614-8574-4e03-89cf-7cf267b0bd08': {
                    'export_locations': [{...}, {...}],
                    'status': 'error',
                },

                '28f6eabb-4342-486a-a7f4-45688f0c0295': {
                    'export_locations': [{...}, {...}],
                    'status': 'available',
                },

            }

        """

Note that drivers that don't override the parent class's implementation of
get_backend_info() would get the parent class implementation which would return
an empty dictionary and thus prevent ensure_shares() from being called, a
change from current behavior. We can optionally add an override implementation
of that method to drivers that need ensure_shares() to be called more often.

As part of this change we will determine which drivers have required logic in
ensure_shares() and implement get_backend_info() for them in such a way that no
functionality is lost.

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

The effect of this change would be to make the manila-share service start up
faster in cases where nothing important has changed (the common case). The
effect will be small when ensure_shares doesn't do anything or when the number
of shares is small, but could be very large for backends with a large number
of shares and expensive ensure_shares operations, reducing a O(n) startup time
to O(1).

Other deployer impact
---------------------

None

Developer impact
----------------

Drivers will be strongly encouraged to implement get_backend_info(), but
won't be required. Any drivers implementing ensure_shares() will need to update
their logic to not assume that ensure_shares is called every time the driver
starts.

Most importantly, drivers will be able to implement potentially more expensive
operations in ensure_shares() without creating large scalability problems.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

* zhongjun(jun.zhongjun2@gmail.com)

Work items
----------

* Implement the core feature with functional tempest and scenario test
  coverage
* Convert all ensure_share() methods to ensure_shares()
* Implement get_backend_info() in first party drivers.
* Add documentation of this feature

Dependencies
============

None

Testing
=======

Due to the difficulty of restarting services during functional tests, it's
only practical to test this change with unit tests.

Documentation impact
====================

Documentation of this feature will be added to the developer reference.

References
==========

None
