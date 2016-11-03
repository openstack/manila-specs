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
+-------------------------------------------+-----------------------+

.. _Valeriy Ponomaryov: https://launchpad.net/~vponomaryov

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
