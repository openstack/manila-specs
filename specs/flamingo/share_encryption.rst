..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================
Manila share encryption
=======================

Blueprint: https://blueprints.launchpad.net/manila/+spec/share-encryption

Encrypting OpenStack Manila shares is crucial for ensuring the security and
confidentiality of users' data. There are broadly two levels of encryption:
"front-end" (data in-transit) and "back-end" (data at-rest). Currently,
users can request back-end data encryption via share types that have custom
extra-specs. These custom-extra specs direct the back end driver to encrypt
the share data at rest, however, there is no mechanism for the user to control
much else regarding the encryption process. Ideally, users must be allowed to
create and manage their own encryption keys. This specification proposes an
approach that enables Manila to coordinate user defined encryption keys for
"back-end" (at rest) encryption of share data.


Problem description
===================

While manila users can create encrypted shares with some storage back ends,
they cannot create or control their encryption keys via OpenStack Manila.
Encryption keys are made up by the storage back end or the Manila driver, and
any one with access to the keys could access the data if they gain access to
the back end storage. So this spec is addressing is user control of encryption
keys.

Here are some reasons why you should consider encrypting Manila shares:

1. **Data Confidentiality:** Encryption protects the confidentiality of your
   data by converting it into unreadable ciphertext. If unauthorized users
   gain access to the storage, they won't be able to make sense of the
   encrypted data without the appropriate decryption key.

2. **Compliance Requirements:** Many industries and regulatory standards
   require the encryption of sensitive data. Encrypting OpenStack Manila share
   helps you comply with data protection regulations and industry standards,
   ensuring that your organization meets legal requirements.

3. **Protection Against Unauthorized Access:** Encrypting shares adds an extra
   layer of security against unauthorized access. Even if someone gains access
   to the underlying storage, they won't be able to access the data without the
   encryption key.

4. **Mitigation of Insider Threats:** Encryption can help mitigate the risk of
   insider threats. Even if an authorized user with access to the storage
   attempts to misuse the data, encryption prevents them from reading or
   tampering with sensitive information without the proper decryption key.

5. **Protection Against Data Breaches:** In the event of a security breach or
   data leak, encrypted data is much more difficult for attackers to exploit.
   This can significantly reduce the impact of a data breach, as the stolen
   information remains unreadable without the encryption key.

6. **Risk Management:** Encryption is a fundamental component of a
   comprehensive risk management strategy. By implementing encryption for
   Manila shares, you enhance your overall security posture and reduce the
   potential impact of security incidents.


Use Cases
=========

Users would like to protect the data within shares. This proposal of share
encryption provides a manila level support to trigger the data encryption on
backend and thus protect the data from attacker.

Users can provide Manila with a reference to an encryption key. This encryption
key can be applied by share drivers to share servers that they create, or
individually to a specific share. Such keys are stored in through the OpenStack
key management service, Barbican. The encryption key ref will be provided in
share create request. After key ref is being validated by manila, it will be
given to storage back end driver alongside information on how to access the
key manager. The key data will be used to encrypt the share's data within the
storage back end.

The actual encryption of the data at-rest is performed by the back end storage
system. The scope of Manila's involvement ends with coordinating the user's
secret with the Key-store.


Proposed change
===============

* Modify share create API

  In order to support 'bring your own key' use case, manila share create API
  will accept optional parameters i.e. ``encryption-key-ref``.

* Share type extra-spec changes

  A tenant-visible extra-spec on share types called "encryption_support" will
  be introduced. It can have a string value. Value can be "share-server" or
  "share". When set to "share", it means that the user can expect encryption
  keys to apply per-share. Value "share_server" would mean that the user can
  expect encryption keys to apply at the share server level. Default will be
  None.

* Manila API and Share service changes

  Manila api service will allow configuration of a key manager. We will
  introduce an interface for the Manila API service to communicate with an
  external key manager interface (OpenStack Castellan), which internally works
  with a key store (Openstack Barbican). The key manager will be configured
  via conf file.
  When encryption key ref is provided, and share type has "encryption_support"
  set to either "share" or "share_server", Manila will validate the encryption
  key ref with barbican and store the encryption key ref in the share instance
  along with the type of encryption supported. Encryption key ref will be a
  valid barbican secret_ref. Storage back end devices would need to obtain the
  key out-of-band in order to perform the encryption. Manila will collate the
  necessary information that allows a storage back end to identify and retrieve
  the secret from the key manager. Manila will interact with Barbican as a
  service. It will use service credentials to retrieve secrets from Barbican
  on behalf of OpenStack users. Back end storage devices would also need
  OpenStack credentials to work with Barbican. We will improve Manila's share
  manager service to create OpenStack Identity Service (Keystone) application
  credentials to facilitate this interaction, and hand these to the storage
  back end device via storage back end drivers. The application credentials
  will be restricted and so allowed to only fetch key-manager secret for
  service user. The credentials can not list secrets, or retrieve unrelated
  secrets even if they belong to the same tenant.
  To restrict share server creation with different encryption key refs new
  project level quota called 'server_encryption_keys' will be introduced.
  API service will check the quota limit in case share type extra-spec is
  'share-server'. Once quota limit is reached, API will throw error.

* Changes in Manila Scheduler

  CapabilitiesFilter will be updated to consider share backend drivers
  capability `encryption_support`.

* Changes in backend driver

  A backend driver will report `encryption_support` as a capability. It can
  have any of these values: ["share"], ["share_server"],
  ["share", "share_server"] or None. When a share creation request arrives,
  if DHSS=True, drivers will be asked to provide a compatible share server
  and the call will include the share's encryption-key-ref along with all the
  other network details. If a share server exists with the appropriate key ref
  and satisfying the network parameters, the driver can present that server.
  If a compatible share server doesn't exist, the driver returns [] and the
  share manager will ask the driver to create a new server. The key ref will
  be stored in share servers db model. The encryption key ref along-with
  application credentails will be provided to backend hardware to perform the
  encryption. The back end storage system will fetch the key directly, out of
  band of manila using the ref that Manila shares via the back end driver. The
  fetched key data will be used to encrypt the share data.


Alternatives
------------

If OpenStack Manila doesn't provide a way for users to manage their own
encryption keys, the cloud may need an out-of-band solution, such as:

1. External or third party key management services that support integration
   with OpenStack Manila.

2. Client-Side Encryption: forego data encryption at-rest. Users must encrypt
   their data locally on their clients before storing it in Manila shares.

3. File-Level Encryption: encrypting individual files or directories within
   the clients using tools or libraries instead of encrypting the share data
   as a whole.

4. Custom Scripts or Tools: Deployment-local scripts that enable users to
   manage their encryption keys outside of OpenStack Manila. This may involve
   creating a user interface or command-line tool that interacts with OpenStack
   Manila and external key management systems.

5. OpenStack Manila Extensions: unofficial API extensions that can enhance the
   functionality of Manila to deal with encryption metadata.

In all, these alternatives are inferior to the convenience that we would
provide by implementing the proposal in this specification.


Data model impact
-----------------

* New field in share_servers table

  +-----------------------+---------------+------+-----+---------+-------+
  | Field                 | Type          | Null | Key | Default | Extra |
  +=======================+===============+======+=====+=========+=======+
  | encryption_key_ref    | string(36)    | YES  |     | NULL    |       |
  +-----------------------+---------------+------+-----+---------+-------+
  | app_cred_id           | string(36)    | YES  |     | NULL    |       |
  +-----------------------+---------------+------+-----+---------+-------+

* New field in share_instances table

  +-----------------------+---------------+------+-----+---------+-------+
  | Field                 | Type          | Null | Key | Default | Extra |
  +=======================+===============+======+=====+=========+=======+
  | encryption_key_ref    | string(36)    | YES  |     | NULL    |       |
  +-----------------------+---------------+------+-----+---------+-------+


CLI API impact
--------------

Add new parameters to commands in openstackclient(OSC):

.. code-block:: bash

    openstack share create [--encryption-key-ref <key-ref>]
                           <share_protocol> <size>

* encryption-key-ref: Valid Barbican (i.e. key-manager) secret ref (UUID)
  represents share or share server encryption key reference


REST API impact
---------------

** To Create share with user defined share encryption key.**::

     POST /v2/shares

Request::

    {
        "share": {
            "description": "My custom share London",
            "share_type": null,
            "share_proto": "nfs",
            "share_network_id": "713df749-aac0-4a54-af52-10f6c991e80c",
            "share_group_id": null,
            "name": "share_London",
            "snapshot_id": null,
            "is_public": true,
            "size": 1,
            "metadata": {
            },
            "scheduler_hints": {
            },
            "encryption_key_ref": "b7460a86-30ea-4c20-901f-6cee1e945286",
        }
    }

The ``encryption_key_ref`` should be valid Barbican secret href or UUID,
otherwise the API will respond with ``400 Not Found``

Response(202 Accepted)::

   {
        "share": {
            "id": "011d21e2-fbc3-4e4a-9993-9ea223f73264",
            "size": 1,
            "description": "My custom share London",
            "status": "creating",
            "progress": null,
            "share_server_id": null,
            "project_id": "16e1ab15c35a457e9c2b2aa189f544e1",
            "name": "share_London",
            "share_type": "25747776-08e5-494f-ab40-a64b9d20d8f7",
            "share_type_name": "default",
            "availability_zone": null,
            "created_at": "2025-02-18T10:25:24.533287",
            "export_location": null,
            "links": [
                {
                    "href": "http://172.18.198.54:8786/v1/16e1ab15c35a457e9c2b2aa189f544e1/shares/011d21e2-fbc3-4e4a-9993-9ea223f73264",
                    "rel": "self"
                },
            ],
            "share_network_id": "713df749-aac0-4a54-af52-10f6c991e80c",
            "share_group_id": null,
            "export_locations": [],
            "share_proto": "NFS",
            "host": null,
            "access_rules_status": "active",
            "has_replicas": false,
            "replication_type": null,
            "task_state": null,
            "snapshot_support": true,
            "volume_type": "default",
            "snapshot_id": null,
            "is_public": true,
            "metadata": {
            },
            "encryption_key_ref": "b7460a86-30ea-4c20-901f-6cee1e945286"
        }
   }


Driver impact
-------------
The backend driver needs to implement:

* The driver needs to report "encryption_support" as a capability.
  If encryption is supported, the value of this capability can be reported
  as a list of capabilities with valid keys among "share" and "share_server".

* Return valid list of share servers to Manila-share Manager based on
  requested parameters. This is supported today, however function needs to
  be updated to consider additional requirement i.e. encryption key ref. When
  an encryption key reference is provided by Manila's share manager, care must
  be taken to specifically eliminate share servers that don't match that key.

* Instruct the back end storage system to encrypt the share with key
  data sent from key-store e.g. Barbican

Security impact
---------------

After encryption, the shares will be more secure and data is protected from
attacker.


Notifications impact
--------------------

None

Other end user impact
---------------------

User need to store encryption key/payload with Barbican key-manager and get
encryption key ref.

Performance Impact
------------------

The share can be encrypted at front-end or back-end. But for Manila we intend
to support only back-end encryption and so very less performance penalty in
manila services.

Other deployer impact
---------------------

None.

Developer impact
----------------



Implementation
==============

Assignee(s)
-----------

Primary assignee:
    * kpdev(kinpaa@gmail.com)

Work Items
----------

* Implement 'bring you own key' in share create APIs.
* Update 'openstack share create' command in python-manilaclient.
* Implement tempest support.
* Document about create share with encryption

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

- Docstrings
- Devref
- User guide
- Admin guide
- Release notes

References
==========

None
