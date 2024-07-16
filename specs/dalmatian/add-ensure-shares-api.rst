..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================
Add new ensure shares API
=========================

Ensure shares is a functionality in the manila share service that goes over all
of the shares within that backend and checks and updates them in case something
has changed since the last time it was invoked.

Currently, ensure shares is only called while the manila-share binaries are
being restarted, and in case the OpenStack Operator has identified that
something has changed in the storage backend.

Ensure shares can not be invoked via the Manila API, so in case a
configuration changes, the corresponding manila-share binaries must be
restarted to pick up the new configuration.


Problem description
===================

In cloud environments, the share backend configuration can change, and such
changes might mean that the share will have outdated mount information until
Manila applies the most recent configuration.

Currently, we are only applying such changes when the manila-share binary
starts up and invokes the ``ensure_shares`` routine for a share driver.
In a multi-backend environment, a single share manager service instance spawns
multiple processes, each interacting with a single storage backend through a
driver. In such an environment, it is possible that "ensure_shares" needs to be
run for a single/specific storage backend driver. Today, there is no such
provision. So, even if an operator wants to ensure shares belonging to a single
backend, they would need to perform a disruptive restart of all backend drivers
in such environments.

Ensure shares being called only when the service is restarted provides little
flexibility to OpenStack operators to make configuration changes and apply them
quickly to all of the shares within that backend, as every time they would
need to restart the manila-share service. This operation is also disruptive,
and will cause a temporary outage to all of the binaries involved.


Use Cases
=========

OpenStack Operators would like to update the backend configuration and having
it applied quickly to the shares without having the cloud facing the downtime
of a ``manila-share`` service restart, making this a good way to have export
locations and other important share information updated easily.


Proposed change
===============

* New API for ensure shares

  We will introduce an API that allows OpenStack Operators to call the manila
  share's ensure shares routine, making this configuration update smooth and
  requiring no service downtime, so that export locations can be re-calculated
  and shares updated with the newer config.

* Changes to manila shares

  Shares will have a different status while the configuration updates are being
  applied: ``ensuring``.
  Ensure shares can take a while to complete, as we will be updating all shares
  in the given backend. After all shares had the configuration applied, we will
  transition them back to the ``available`` status in case there was a status
  change.

* Changes to the manila-share service:

  While running ensure shares, the ``ensuring`` field for the service will
  transition to True until the operation is complete. All services which the
  binary are not ``manila-share`` will remain untouched.

* Things to consider for this spec

  The manila-share service should be up and enabled for being able to run the
  ensure shares through the API.

  We will always allow the ensure shares API to be triggered. The manila
  share-manager service will decide if a share needs its data to be changed or
  not based on the status of the share, or if the share is busy or not.

  A new configuration option named ``update_shares_status_on_ensure`` will be
  added, and it will default to ``True``. Through this configuration option,
  OpenStack operators can decide whether they want the shares status to be
  changed when ensure shares was invoked through the API or not.


Alternatives
------------

Implementing periodic configuration checks, with configurable time for when
they would run, and as a result they would pick up the newest configuration and
apply them to the shares. However, routine would be run periodically, meaning
that we would check for updates even if there are not.

Data model impact
-----------------

* A new status (``ensuring``) will be added to the shares' possible
  status.

* New field will be added to the services table
  +-----------------------+---------------+------+-----+---------+-------+
  | Field                 | Type          | Null | Key | Default | Extra |
  +=======================+===============+======+=====+=========+=======+
  | ensuring              | tinyint(1)    | YES  |     | NULL    |       |
  +-----------------------+---------------+------+-----+---------+-------+


CLI API impact
--------------

A new command will be added to the Manila's OpenStack CLI:

.. code-block:: bash

    openstack share service ensure <host@backend>


* host@backend: the host and the backend that should update its shares
  configuration.


REST API impact
---------------

**To trigger the ensure shares routine**::

    POST /v2/services/ensure

Request::

    {
      "host": "openstackhost@generic",
    }

Normal http response code(s):

- 202 - The share service will start the ensure shares procedure.

Expected http error code(s):

- 401 - Unauthorized; user has not authenticated
- 403 - Forbidden; user is forbidden by policy
- 404 - Not found; API does not exist in micro version
- 404 - Not found; There is no host and backend matching the provided
- 409 - Conflict; Not all shares are currently available.
- 409 - Conflict; Service is not up.


Driver impact
-------------

No driver impact, as drivers are currently free to implement what is necessary
during the ensure shares routine, we will just allow OpenStack Operators to
access it easily.

Security impact
---------------

We will implement RBAC rules for this API. Only OpenStack Operators will be
able to use it by the virtue of the default RBAC policy.


Notifications impact
--------------------

We will emit a notification when the "ensure_shares" operation starts and when
the operation ends.

Other end user impact
---------------------

When state changes are enabled, and an OpenStack operator invokes the ensure
shares API, end users will notice that their shares have transitioned to
"ensuring" status. They will be unable to perform any actions on the share via
Manila. Unless the backend driver updates export locations as part of this
operation, there is no impact on the data path.

Performance Impact
------------------

Shares will be updated by the manila-share service and this can take some
time to complete, specially in busy clouds.

Other deployer impact
---------------------

None.

Developer impact
----------------

None.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
    * carloss (ces.eduardo98@gmail.com)

Work Items
----------

* Implement the ensure shares API.
* Implement the corresponding OSC commands.
* Implement the corresponding functionality in SDK.
* Implement tempest tests.
* Update the documentation.
* Add the feature to manila-ui.

Future Work Items
-----------------

None

Dependencies
============

None

Testing
=======

* Unit tests
* Tempest tests

Documentation Impact
====================

- User guide
- Admin guide

References
==========

_`[1]`: https://bugs.launchpad.net/manila/+bug/1996793
_`[2]`: https://etherpad.opendev.org/p/caracal-ptg-manila-cephfs
