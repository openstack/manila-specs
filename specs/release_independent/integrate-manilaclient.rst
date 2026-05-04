..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================
Integrate manilaclient OSC commands into OSC
============================================

https://blueprints.launchpad.net/manila/+spec/integrate-manilaclient

Integrate the OSC commands currently found in manilaclient into OSC.

Problem description
===================

Like most of the non-core services, manila provides commands for OSC via a
plugin provided as part of its client package, ``python-manilaclient``.
Manila is mature enough now that it makes sense to start packaging these
commands into OSC proper, in the same way that Manila SDK bindings are provided
in SDK. Eventually this will allow us to significantly improve the performance
of OSC by getting rid of entrypoint scanning.

Use Cases
=========

* As a user, I would like to have access to manila-related commands without
  installing additional packages.
* As a user, I would like to have meta commands like ``availability zone list``
  behave consistently for all services.

Proposed change
===============

Add the ability to prioritize commands provided in-tree in OSC over those
provided by plugins, then migrate all commands from manilaclient to OSC before
removing the manilaclient variants.

We will **not** migrate the underlying library from manilaclient to SDK yet.
This will be tackled in a follow-up.

Alternatives
------------

None.

Data model impact
-----------------

N/A.

REST API impact
---------------

N/A.

Driver impact
-------------

N/A.

Security impact
---------------

N/A.

Notifications impact
--------------------

N/A.

Other end user impact
---------------------

Users will no longer need to install python-manilaclient to get manila commands
in their environment.

Performance Impact
------------------

None.

Other deployer impact
---------------------

None.

Developer impact
----------------

Changes to manila's OSC commands will now need to be created against OSC rather
than manilaclient.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  stephen.finucane

Work Items
----------

We need to do the initial bootstrapping of the client in OSC but once done, commands can be migrated to OSC incrementally using the following pattern:

* Copy the commands module and tests for same (e.g. ``manilaclient/osc/v2/services.py`` and ``manilaclient/tests/unit/osc/v2/test_services.py``) to their respective locations in OSC (e.g. ``openstackclient/share/v2/`` and ``openstackclient/tests/unit/share/v2/``)

* Update code and tests.

  * For code:

    * Replace the following imports:

      * ``from manilaclient.common._i18n import _`` with ``from openstackclient.i18n import _``
      * ``from manilaclient.osc import utils`` with ``from openstackclient.share import utils``

    * Reorder imports to reflect the fact that manilaclient is now a third
      party import (second grouping) and openstackclient is a first party
      import (third grouping)

  * For tests:

    * Replace the following imports:

      * ``from manilaclient.osc.v2 import foo as osc_foo`` with ``from openstackclient.share.v2 import foo``, updating remaining references to ``osc_foo`` with ``foo``

      * ``from manilaclient.tests.unit.osc import osc_utils`` with ``from openstackclient.tests.unit import utils as test_utils``, updating remaining references to ``osc_utils`` with ``test_utils``

      * ``from manilaclient.tests.unit.osc.v2 import fakes as manila_fakes`` with ``from openstackclient.tests.unit.share.v2 import fakes as share_fakes``, updating remaining references to ``manila_fakes`` with ``share_fakes``

    * Replace any calls to ``self.app.client_manager.share.api_version = api_versions.APIVersion(foo)`` in test ``setUp`` methods and test methods with calls to ``self.set_share_api_version(foo)`` (where ``foo`` is e.g. ``api_versions.MAX_VERSION`` or ``2.51``)

    * Reorder imports to reflect the fact that manilaclient is now a third
      party import (second grouping) and openstackclient is a first party
      import (third grouping)

Dependencies
============

None.

Testing
=======

Tests will be copied across as part of the migration.

Documentation Impact
====================

Any references to installing python-manilaclient in the docs can be removed.

References
==========

None.
