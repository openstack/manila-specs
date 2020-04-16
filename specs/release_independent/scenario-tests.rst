..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================
Scenario tests design
=====================

https://blueprints.launchpad.net/manila/+spec/scenario-tests

Manila has good functional test coverage of its features,
but narrow coverage of scenarios.
And with addition of new features this coverage becomes even less.

Problem description
===================

Drivers often lack features or implement them incorrectly and
it's very difficult for humans to notice this in code review. Tests ensure
that drivers which lack features must explicitly skip tests to get a passing
result, and the non-skipped tests ensure that the drivers implement features
correctly.
Documentation for how features should work is often sparse and driver authors
don't always know what is the correct behavior when implementing a driver
feature. Tests allow driver authors to code the feature to pass the test,
which simplifies the driver author's job.

Use Cases
=========

Consider the case when new manila deployment becomes ready. And one wants
to verify that main user use cases work in general. Having automatic scenario
tests one could test his deployment much faster than manually, as it is now.
Also, it could be used in CI systems to test share drivers continuously.

Proposed change
===============

It is proposed to get agreement on scenario tests design (this spec) and
implement them in manila plugin for tempest. After that these tests could be
used in CI systems as well as on customer deployments. This spec covers only
existing features in manila as of last available (Newton) release. All newly
added features should be covered separately.

Prerequisites for scenarios:

* Depending on share driver mode, it can be required to create share-network
  with or without security-services.
* Depending on protocol, its versions and access type, "mount" operations
  should be defined explicitly. Scenario tests assume it as predefined
  and known. Hence, scenarios do not include difference between
  access type (IP, User, Cert). Due to this, scenario tests should be
  data driven by "access_type", "access_proto", "access_level" and
  "mount command with all expected options".
* **Bold texts** depend on specific implementations and
  can be unsupported by share backends.
* Text in brackets depend on share driver configuration and may be optional.
* All user machines are separate VMs from share-nodes.
  They should have network connectivity with share host.
  User VMs should be built with image that has shared file systems clients.
* Share host should have open ports for SSH protocol and
  protocols of shared file systems.

Here is list of scenarios for implementation:

* `1) Share access and file operations`_ `(Partially implemented)`
* `2) Share access with multiple guests`_ `(Partially implemented)`
* `3) Relationships between source shares and child shares`_ `(Implemented)`
* `4) Create/extend share and write data`_ `(Implemented)`
* `5) Create/shrink share and write data`_ `(Implemented)`
* `6) Create/manage share and write data`_ `(Implemented)`
* `7) Create/manage share and snapshot and write data`_ `(Partially implemented)`
* `8) Replicate ‘writable’ share and write data`_ `(Not yet implemented)`
* `9) Replicate and promote ‘readable’ share and write data`_ `(Not yet implemented)`
* `10) Replicate and promote ‘dr’ share and write data`_ `(Not yet implemented)`
* `11) Get a snapshot of a replicated share`_ `(Not yet implemented)`
* `12) Migrate share and write data`_ `(Implemented)`

_`1) Share access and file operations`

`Driver mode: any`

`Involved APIs:`

* [create share network]
* [create share type]
* create share
* allow share access
* deny share access
* delete share

.. note::

    **Implementation Status**

    This test case extends `manila_tempest_tests.tests.scenario
    .test_share_basic_ops.ShareBasicOpsBase#test_write_with_ro_access
    <https://opendev
    .org/openstack/manila-tempest-plugin/src/commit
    /eff4f9b87f0d36e0cfa4b1d861125f456f341af9/manila_tempest_tests/tests
    /scenario/test_share_basic_ops.py#L117>`_

.. list-table:: **Scenario 1 steps**
   :class: table-striped
   :stub-columns: 1
   :widths: 3 25 25
   :header-rows: 1

   * - Step
     - Action
     - Result
   * - 1
     - Create user VM (UVM)
     - ok, created
   * -  2
     - Create share (S)
     - ok, created
   * - 3
     - SSH to UVM
     - ok, connected
   * - 4
     - Try mount S to UVM
     - fail, access denied
   * - 5
     - Provide RO access to S
     - ok, provided
   * - 6
     - Try mount S to UVM
     - ok, mounted
   * - 7
     - Try create files on S
     - fail, access denied
   * - 8
     - Unmount S from UVM
     - ok, unmounted
   * - 9
     - Remove RO access from S
     - ok, removed
   * - 10
     - Try mount S to UVM
     - fail, access denied
   * - 11
     - Provide RW access to S
     - ok, provided
   * - 12
     - Try mount S to UVM
     - ok, mounted
   * - 13
     - Try write files to S
     - ok, written
   * - 14
     - Try read files from S
     - ok, read
   * - 15
     - Try delete files on S
     - ok, deleted
   * - 16
     - Unmount S from UVM
     - ok, unmounted
   * - 17
     - Delete S
     - ok, deleted
   * - 18
     - Try mount S
     - fail, not found
   * - 19
     - Delete UVM
     - ok, deleted

_`2) Share access with multiple guests`

`Driver mode: any`

`Involved APIs:`

* [create share network]
* [create share type]
* create share
* allow share access
* delete share

.. note::

    **Implementation Status**

    This test case extends `manila_tempest_tests.tests.scenario
    .test_share_basic_ops.ShareBasicOpsBase#test_read_write_two_vms
    <https://opendev
    .org/openstack/manila-tempest-plugin/src/commit
    /eff4f9b87f0d36e0cfa4b1d861125f456f341af9/manila_tempest_tests/tests
    /scenario/test_share_basic_ops.py#L148>`_


.. list-table:: **Scenario 2 steps**
   :stub-columns: 1
   :widths: 3 25 25
   :header-rows: 1

   * - Step
     - Action
     - Result
   * - 1
     - Create UVM1
     - ok, created
   * - 2
     - Create UVM2
     - ok, created
   * - 3
     - Create share S
     - ok, created
   * - 4
     - Add RW access to UVM1
     - ok, added
   * - 5
     - SSH to UVM1
     - ok, connected
   * - 6
     - Try mount S from UVM1
     - ok, mounted
   * - 7
     - SSH to UVM2
     - ok, connected
   * - 8
     - Try mount S from UVM2
     - fail, access denied
   * - 9
     - Add RW access for UVM2
     - ok, added
   * - 10
     - Try mount S from UVM2
     - ok, two VMs have it mounted at once.
   * - 11
     - Create test file in mounted share from UVM1
     - ok, created. Available from UVM2
   * - 12
     - Write data to test file from UVM2
     - ok, written. Available from UVM1 too
   * - 13
     - Unmount S on UVM1
     - ok, unmounted
   * - 14
     - Unmount S on UVM2
     - ok, unmounted
   * - 15
     - Delete UVM1
     - ok, deleted
   * - 16
     - Delete UVM2
     - ok, deleted
   * - 17
     - Delete S
     - ok, deleted

_`3) Relationships between source shares and child shares`

`Driver mode: any`

`Involved APIs:`

* [create share network]
* [create share type]
* create share
* allow share access
* create share snapshot
* delete share snapshot
* delete share

.. note::

    **Implementation Status**

    This test case has been implemented as `manila_tempest_tests.tests
    .scenario.test_share_basic_ops
    .ShareBasicOpsBase#test_write_data_to_share_created_from_snapshot
    <https://opendev
    .org/openstack/manila-tempest-plugin/src/commit
    /eff4f9b87f0d36e0cfa4b1d861125f456f341af9/manila_tempest_tests/tests
    /scenario/test_share_basic_ops.py#L291>`_


.. list-table:: **Scenario 3 steps**
   :stub-columns: 1
   :widths: 3 25 25
   :header-rows: 1

   * - Step
     - Action
     - Result
   * - 1
     - Create UVM
     - ok, created
   * - 2
     - Create share S1
     - ok, created
   * - 3
     - Provide RW access to S1
     - ok, provided
   * - 4
     - SSH to UVM
     - ok, connected
   * - 5
     - Try mount S1 to UVM
     - ok, mounted
   * - 6
     - Create "file1"
     - ok, created
   * - 7
     - Create snapshot SS1 from S1
     - ok, created
   * - 8
     - Create "file2" in share S1
     - ok, created. We expect that snapshot will not contain any data created
       after snapshot creation.
   * - 9
     - Create share S2 from SS1
     - ok, created
   * - 10
     - Try mount S2
     - fail, access denied. We test that child share did not get access rules
       from parent share.
   * - 11
     - Provide RW access to S2
     - ok, provided
   * - 12
     - Try mount S2
     - ok, mounted
   * - 13
     - List files on S2
     - only "file1" exists
   * - 14
     - Create file3 on S2
     - ok, file created
   * - 15
     - List files on S1
     - two files exist - "file1" and "file2"
   * - 16
     - List files on S2
     - two files exist - "file1" and "file3"
   * - 17
     - Unmount S1 and S2
     - ok, unmounted
   * - 18
     - Delete S2, then SS1, then S1, then UVM
     - ok, all deleted

_`4) Create/extend share and write data`

`Driver mode: any`

`Involved APIs:`

* [create share network]
* [create share type]
* create share
* allow share access
* extend share
* delete share

.. note::

    **Implementation Status**

    This test case has been implemented as `manila_tempest_tests.tests
    .scenario.test_share_extend.ShareExtendBase#test_create_extend_and_write
    <https://opendev
    .org/openstack/manila-tempest-plugin/src/commit
    /eff4f9b87f0d36e0cfa4b1d861125f456f341af9/manila_tempest_tests/tests
    /scenario/test_share_extend.py#L49>`_

.. list-table:: **Scenario 4 steps**
   :class: table-striped
   :stub-columns: 1
   :widths: 3 20 25
   :header-rows: 1

   * - Step
     - Action
     - Result

   * - 1
     - Create UVM
     - ok, created
   * - 2
     - Create share S1 of 1Gb size
     - ok, created
   * - 3
     - Provide RW access to S1
     - ok, provided
   * - 4
     - SSH to UVM
     - ok, connected
   * - 5
     - Try mount S1 to UVM
     - ok, mounted
   * - 6
     - Create "file1"
     - ok, created
   * - 7
     - Fill file1 with data as possible
     - size of a file does not exceed share size quota
   * - 8
     - Extend share S1 to 2Gb
     - ok, extended
   * - 9
     - Write additional data to file1
     - data written, size of a file does not exceed new share size quota and
       it is more than old one
   * - 10
     - Unmount S1
     - ok, unmounted
   * - 11
     - Delete share S1
     - ok, deleted
   * - 12
     - Delete UVM
     - ok, deleted

_`5) Create/shrink share and write data`

`Driver mode: any`

`Involved APIs:`

* [create share network]
* [create share type]
* create share
* allow share access
* **shrink share**
* delete share

.. note::

    **Implementation Status**

    This test case has been implemented as `manila_tempest_tests.tests
    .scenario.test_share_shrink.ShareShrinkBase#test_create_shrink_and_write
    <https://opendev
    .org/openstack/manila-tempest-plugin/src/commit
    /eff4f9b87f0d36e0cfa4b1d861125f456f341af9/manila_tempest_tests/tests
    /scenario/test_share_shrink.py#L52>`_

.. list-table:: **Scenario 5 steps**
   :class: table-striped
   :stub-columns: 1
   :widths: 3 23 25
   :header-rows: 1

   * - Step
     - Action
     - Result
   * - 1
     - Create UVM
     - ok, created
   * - 2
     - Create share S1 of 2Gb size
     - ok, created
   * - 3
     - Provide RW access to S1
     - ok, provided
   * - 4
     - SSH to UVM
     - ok, connected
   * - 5
     - Try mount S1 to UVM
     - ok, mounted
   * - 6
     - Write some data for 2 Gb
     - ok, created
   * - 7
     - Fill file1 with data as possible
     - size of a file does not exceed share size quota
   * - 8
     - Try shrink share S1 to 1Gb
     - fail, possible data loss exception
   * - 9
     - Delete data for amount of 1 Gb
     - data deleted
   * - 10
     - Shrink share S1 to 1Gb
     - ok, shrinked
   * - 11
     - Try write data more than new size of 1 Gb
     - fail, cannot write
   * - 12
     - Unmount S1
     - ok, unmounted
   * - 13
     - Delete share S1
     - ok, deleted
   * - 14
     - Delete UVM
     - ok, deleted

_`6) Create/manage share and write data`

`Driver mode: any`

`Involved APIs:`

* [create share network]
* [create share type]
* create share
* allow share access
* **manage share**
* **unmanage share**
* **manage share again**
* delete share

.. note::

    **Implementation Status**

    This test case has been partially implemented as `manila_tempest_tests
    .tests.scenario.test_share_manage_unmanage
    .ShareManageUnmanageBase#test_create_manage_and_write
    <https://opendev
    .org/openstack/manila-tempest-plugin/src/commit
    /eff4f9b87f0d36e0cfa4b1d861125f456f341af9/manila_tempest_tests/tests
    /scenario/test_share_manage_unmanage.py#L60>`_ . It currently tests only
    ``DHSS=False`` back end share drivers. To complete the implementation,
    this test case needs to support ``DHSS=True`` mode of share drivers.
    Support for managing shares with DHSS=True was added to Manila via API
    version 2.49. So this test must create a share network if
    ``[share]/multitenancy_enabled=True`` and the API version being tested
    is >= 2.49, and manage the share into the specific share network.

.. list-table:: **Scenario 6 steps**
   :class: table-striped
   :stub-columns: 1
   :widths: 3 15 25
   :header-rows: 1

   * - Step
     - Action
     - Result
   * - 1
     - Create UVM
     - ok, created
   * - 2
     - Create share S1 of 1Gb size
     - ok, created
   * - 3
     - Provide RW access to S1
     - ok, provided
   * - 4
     - SSH to UVM
     - ok, connected
   * - 5
     - Try mount S1 to UVM
     - ok, mounted
   * - 6
     - Write some data
     - ok, written
   * - 7
     - Unmount S1
     - ok, unmounted
   * - 8
     - Unmanage share
     - ok, unmanaged
   * - 9
     - Try get share S1
     - fail, 404 code in response
   * - 10
     - Manage share S1
     - ok, managed.
   * - 11
     - Provide RW access to S1 again
     - ok, provided. We make sure that even if rule has existed on backend,
       we do not fail if explicitly try add it again after ‘manage’ operation.
   * - 12
     - Try mount S1 to UVM
     - ok, mounted. Previously created data still there.
   * - 13
     - Unmount S1
     - ok, unmounted
   * - 14
     - Delete share
     - ok, deleted
   * - 15
     - Try manage share again
     - fail, resource not found
   * - 16
     - Delete UVM
     - ok, deleted

_`7) Create/manage share and snapshot and write data`

`Driver mode: any`

`Involved APIs:`

* [create share network]
* [create share type]
* create share
* allow share access
* **create snapshot**
* **manage share**
* **unmanage share**
* **manage snapshot**
* **unmanage snapshot**
* **delete snapshot**
* delete share

.. note::

    **Implementation Status**

    This test case is yet to be implemented.

.. list-table:: **Scenario 7 steps**
   :class: table-striped
   :stub-columns: 1
   :widths: 3 15 25
   :header-rows: 1

   * - Step
     - Action
     - Result
   * - 1
     - Create UVM
     - ok, created
   * - 2
     - Create share S1 of 1Gb size
     - ok, created
   * - 3
     - Provide RW access to S1
     - ok, provided
   * - 4
     - SSH to UVM
     - ok, connected
   * - 5
     - Try mount S1 to UVM
     - ok, mounted
   * - 6
     - Write some data
     - ok, written
   * - 7
     - Create snapshot SS1
     - ok, created
   * - 8
     - Unmanage snapshot SS1
     - ok, unmanaged
   * - 9
     - Unmanage share S1
     - ok, unmanaged
   * - 10
     - Try get share S1
     - fail, 404 code in response
   * - 11
     - Manage share S1
     - ok, managed.
   * - 12
     - Provide RW access to S1 again
     - ok, provided. We make sure that even if rule has existed on backend,
       we do not fail if explicitly try add it again after ‘manage’ operation.
   * - 13
     - Try mount S1 to UVM
     - ok, mounted. Previously created data still there.
   * - 14
     - Manage snapshot SS1
     - ok, managed
   * - 15
     - Delete snapshot SS1
     - ok, deleted
   * - 16
     - Unmount S1
     - ok, unmounted
   * - 17
     - Delete share S1
     - ok, deleted
   * - 18
     - Try manage share S1 again as S2, this should fail asynchronously
       since the resource is gone
     - S2 has a status set to 'error'
   * - 19
     - Delete S2
     - ok, deleted
   * - 20
     - Delete UVM
     - ok, deleted

_`8) Replicate ‘writable’ share and write data`

`Driver mode: any`

`Involved APIs:`

* [create share network]
* [create share network subnets in different availability zones]
* [create share type]
* create share
* allow share access
* **create replica**
* **delete replica**
* delete share

.. note::

    **Implementation Status**

    This test case is yet to be implemented.

.. list-table:: **Scenario 8 steps**
   :class: table-striped
   :stub-columns: 1
   :widths: 3 25 25
   :header-rows: 1

   * - Step
     - Action
     - Result
   * - 1
     - Create UVM1
     - ok, created
   * - 2
     - Create share S1-R1 of 1Gb size
     - ok, created
   * - 3
     - Provide RW access to S1-R1
     - ok, provided
   * - 4
     - SSH to UVM1
     - ok, connected
   * - 5
     - Try mount S1-R1 to UVM1
     - ok, mounted
   * - 6
     - Create file1
     - ok, created
   * - 7
     - Create share replica S1-R2
     - ok, created
   * - 8
     - Create UVM2
     - ok, created
   * - 9
     - SSH to UVM2
     - ok, connected
   * - 10
     - Try mount S1-R2 to UVM2
     - fail, access denied
   * - 11
     - Try mount S1-R2 to UVM1
     - ok,mounted. Same files exist.
   * - 12
     - Provide RW access to S1-R2
     - ok, provided
   * - 13
     - Try mount S1-R2 to UVM2
     - ok, mounted
   * - 14
     - Create file2 in S1-R2
     - ok, created. S1-R1 has both files too.
   * - 15
     - Create file3 in S1-R1
     - ok, created. Both replicas have three created files.
   * - 16
     - Unmount both replicas
     - ok, unmounted
   * - 17
     - Delete original replica S1-R1
     - ok, deleted. second and the only replica now still exists and
       has all files that were created.
   * - 18
     - Delete share S1
     - ok, deleted

_`9) Replicate and promote ‘readable’ share and write data`

`Driver mode: any`

`Involved APIs:`

* [create share network]
* [create share network subnets in different availability zones]
* [create share type]
* create share
* allow share access
* **create replica**
* **promote replica**
* **delete replica**
* delete share

.. note::

    **Implementation Status**

    This test case is yet to be implemented.

.. list-table:: **Scenario 9 steps**
   :class: table-striped
   :stub-columns: 1
   :widths: 3 20 25
   :header-rows: 1

   * - Step
     - Action
     - Result
   * - 1
     - Create UVM1
     - ok, created
   * - 2
     - Create share S1-R1 of 1Gb size
     - ok, created
   * - 3
     - Provide RW access to S1-R1
     - ok, provided
   * - 4
     - SSH to UVM1
     - ok, connected
   * - 5
     - Try mount S1-R1 to UVM1
     - ok, mounted
   * - 6
     - Create file1
     - ok, created
   * - 7
     - Create share replica S1-R2
     - ok, created
   * - 8
     - Create UVM2
     - ok, created
   * - 9
     - SSH to UVM2
     - ok, connected
   * - 10
     - Try mount S1-R2 to UVM2
     - fail, access denied
   * - 11
     - Try mount S1-R2 to UVM1
     - ok, mounted. Same files exist.
   * - 12
     - Provide RW access to S1-R2
     - ok, provided
   * - 13
     - Try mount S1-R2 to UVM2
     - ok, mounted
   * - 14
     - Try create some file in S1-R2
     - fail, filesystem is RO only.
   * - 15
     - Create file2 in S1-R1
     - ok, created. Both replicas have two created files.
   * - 16
     - Promote S1-R2 to active
     - ok, promoted. S1-R1 became RO.
   * - 17
     - Create file3 in S1-R2
     - ok, created. S1-R1 has all files too.
   * - 18
     - Try create some file in S1-R1
     - fail, filesystem is RO
   * - 19
     - Unmount both replicas
     - ok, unmounted
   * - 20
     - Delete original (now RO) replica S1-R1
     - ok, deleted. Second and the only replica (active) now still exists and
       has all files that were created.
   * - 21
     - Delete share S1
     - ok, deleted

_`10) Replicate and promote ‘dr’ share and write data`

`Driver mode: any`

`Involved APIs:`

* [create share network]
* [create share network subnets in different availability zones]
* [create share type]
* create share
* allow share access
* **create replica**
* **promote replica**
* **delete replica**
* delete share

.. note::

    **Implementation Status**

    This test case is yet to be implemented.

.. list-table:: **Scenario 10 steps**
   :class: table-striped
   :stub-columns: 1
   :widths: 3 25 25
   :header-rows: 1

   * - Step
     - Action
     - Result
   * - 1
     - Create UVM
     - ok, created
   * - 2
     - Create share S1-R1
     - ok, created
   * - 3
     - Provide RW access to S1-R1
     - ok, provided
   * - 4
     - SSH to UVM
     - ok, connected
   * - 5
     - Try mount S1-R1 to UVM
     - ok, mounted
   * - 6
     - Create file1
     - ok, created
   * - 7
     - Create share replica S1-R2
     - ok, created
   * - 8
     - Unmount S1-R1
     - ok, unmounted
   * - 9
     - Promote S1-R2
     - ok, promoted. S1-R1 became ‘dr’-only
   * - 10
     - Try mount S1-R2 to UVM
     - ok, mounted. ‘file1’ exists
   * - 11
     - Create file2
     - ok, created
   * - 12
     - Unmount S1-R2
     - ok, unmounted
   * - 13
     - Promote S1-R1
     - ok, promoted. S1-R2 became ‘dr’-only.
   * - 14
     - Try mount S1-R1 to UVM
     - ok, mounted. Files ‘file1’ and ‘file2’ exist.
   * - 15
     - Unmount S1-R1
     - ok, unmounted
   * - 16
     - Delete S1-R2 (current secondary)
     - ok, deleted.
   * - 17
     - Delete share
     - ok, deleted

_`11) Get a snapshot of a replicated share`

`Driver mode: any`

`Involved APIs:`

* [create share network]
* [create share network subnets in different availability zones]
* [create share type]
* create share
* allow share access
* **create snapshot**
* **create share from snapshot**
* **create replica**
* **promote replica**
* **delete replica**
* **delete snapshot**
* delete share

.. note::

    **Implementation Status**

    This test case is yet to be implemented.

.. list-table:: **Scenario 11 steps**
   :class: table-striped
   :stub-columns: 1
   :widths: 3 30 20
   :header-rows: 1

   * - Step
     - Action
     - Result
   * - 1
     - Create UVM
     - ok, created
   * - 2
     - Create share S1-R1
     - ok, created
   * - 3
     - Provide RW access to S1-R1
     - ok, provided
   * - 4
     - SSH to UVM
     - ok, connected
   * - 5
     - Try mount S1-R1 to UVM
     - ok, mounted
   * - 6
     - Create ‘file1’
     - ok, created
   * - 7
     - Create snapshot SS1
     - ok, created
   * - 8
     - Create replica S1-R2
     - ok, created
   * - 9
     - Create ‘file2’
     - ok, created
   * - 10
     - Create snapshot SS2
     - ok, created
   * - 11
     - Unmount S1-R1
     - ok, unmounted
   * - 12
     - Promote S1-R2 (For non-’writable’ replication types)
     - ok, promoted
   * - 13
     - Try mount S1-R2 to UVM
     - ok, mounted
   * - 15
     - Delete S1-R1
     - ok, deleted
   * - 16
     - Create share S2 from SS2
     - ok, created
   * - 17
     - Provide RW access to S2
     - ok, provided
   * - 18
     - SSH to UVM
     - ok, connected
   * - 19
     - Try mount S2 to UVM
     - ok, mounted. All created files exist
   * - 20
     - Unmount S2
     - ok, unmounted
   * - 21
     - Delete S2, SS2, SS1, S1
     - ok, deleted

_`12) Migrate share and write data`

`Driver mode: any`

`Involved APIs:`

* [create share network]
* [create share type]
* create share
* allow share access
* **migration-start share**
* **migration-complete share**
* delete share

.. note::

    **Implementation Status**

    This test case has been implemented as `manila_tempest_tests.tests
    .scenario.test_share_basic_ops.ShareBasicOpsBase#test_migration_files
    <https://opendev
    .org/openstack/manila-tempest-plugin/src/commit
    /eff4f9b87f0d36e0cfa4b1d861125f456f341af9/manila_tempest_tests/tests
    /scenario/test_share_basic_ops.py#L186>`_

.. list-table:: **Scenario 12 steps**
   :class: table-striped
   :stub-columns: 1
   :widths: 3 25 25
   :header-rows: 1

   * - Step
     - Action
     - Result
   * - 1
     - Create UVM
     - ok, created
   * - 2
     - Create share S1 of 1Gb size
     - ok, created
   * - 3
     - Provide RW access to S1
     - ok, provided
   * - 4
     - SSH to UVM
     - ok, connected
   * - 5
     - Try mount S1 to UVM
     - ok, mounted
   * - 6
     - Create file1
     - ok, created
   * - 7
     - Unmount share S1
     - ok, unmounted
   * - 8
     - Do "migration-start"
     - ok, finished. 1 phase is completed.
   * - 9
     - Do "migration-complete"
     - ok, share instance only one - new one. it has previously created file1.
   * - 10
     - Try mount S1 to UVM
     - ok, mounted. Created file1 exists
   * - 11
     - Unmount share S1
     - ok, unmounted
   * - 12
     - Delete share S1
     - ok, deleted
   * - 13
     - Delete UVM
     - ok, deleted

Alternatives
------------

Alternative is what we have now. It is requirement to test each share driver
manually and dependency on presence of detailed docs for each feature share
drivers implement.

Data model impact
-----------------

None

REST API impact
---------------

None

Driver impact
-------------

None

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

End users will be able to run scenario tests against their manila deployment
to test workability of various features.

Performance Impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Original assignee:

* vponomaryov

Other contributors:

* We're inviting more contributors to continue to improve scenario tests.
  Adding new scenario test cases to manila-tempest-plugin does *not* require
  adding the test case description to this spec, but it is encouraged if you
  like feedback for your new test cases.

Work Items
----------

* Implement designed scenario tests in manila plugin for tempest.


Dependencies
============

None

Testing
=======

It is expected that all first-party drivers as well as third-party drivers
will be covered in CI systems with designed here scenario tests.
Due to big amount of optional features that are covered by scenario tests,
only appropriate scenario tests for specific back-end should run in CI systems.
Scenarios that include only required features (1-4) are a must for running in
CI systems for each share driver.

Documentation Impact
====================

Doc describing usage of manila plugin for tempest [1] should be extended with
configuration and usage details of scenario tests.

References
==========

* [1] http://docs.openstack.org/developer/manila/devref/tempest_tests.html
