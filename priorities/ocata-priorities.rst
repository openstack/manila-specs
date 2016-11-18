.. _ocata-priorities:

========================
Ocata Project Priorities
========================

List of themes (in the form of use cases) the manila development team is
prioritizing in Ocata (in no particular order).

+-------------------------------------------+-----------------------+
| Priority                                  | Primary Contacts      |
+===========================================+=======================+
| `Scenario Tests`_                         | `Valeriy Ponomaryov`_ |
| `Eliminate Race Conditions`_              | `Ben Swartzlander`_   |
| `Fix and improve Access Rules`_           | `Goutham Ravi`_       |
| `Enable IPv6`_                            | `Jun Zhongjun`_       |
+-------------------------------------------+-----------------------+

.. _Valeriy Ponomaryov: https://launchpad.net/~vponomaryov
.. _Ben Swartzlander: https://launchpad.net/~bswartz
.. _Goutham Ravi: https://launchpad.net/~gouthamr
.. _Jun Zhongjun: https://launchpad.net/~jun-zhongjun

Scenario Tests
--------------

History has shown that CI without scenario tests is inadequate because some
CI systems are able to pass the existing functional tests while having
serious bugs.

Also lack of clarity about how certain driver operations should work or
which operations are required has led to driver that work incorrectly
despite good testing work by the original author. Scenario tests should
provide a way for driver authors to run a "conformance test" to validate
that their drivers do indeed meet community standards.

To address these two issues we plan to implement new scenario tests to
address coverage gaps and also to request that 3rd party CI systems start
running the scenario tests suite in addition to the functional test suite.

Eliminate Race Conditions
-------------------------

Race conditions are the cause of a lot of Manila's test instability. Fixing
them in a way that works in a distributed active/active HA model is
surprisingly hard, so we plan to implement some fixes that will be the model
for solving race conditions in the future.

Fix and improve Access Rules
----------------------------

Changes made to access rule management in the Mitaka release introduced some
bugs caused by a combination of bad design choices and lack of a good way
to prevent race conditions between state updates in the manager and the API.
We plan to fix these problems once and for all in Ocata.

Enable IPv6
-----------

IPv6 support has been added to Manila in some places but not consistently.
In Ocata we plan to add complete support for IPv6 to the REST API, data model
and and driver interfaces, so that deployers and users in IPv6-only
environments can use Manila.
