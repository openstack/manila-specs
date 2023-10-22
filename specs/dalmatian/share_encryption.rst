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
"back-end" (at rest) encryption of share data. This spec proposes an encryption
solution based on the existing one from cinder `[1]`_.


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

In order to support share encrytion, administrator needs to create a manila
share type and associate encryption information with it. The information
contains fields such as size of encryption key, encryption provider class,
control location and encryption algorithm e.g. aes-xts-plain64. If a share is
created using such share types, Manila will generate an encryption key with
the help of key-store e.g. Barbican.

The user can also encrypt the share using their own keys. Such keys are stored
in key store supported by driver e.g. Barbican. In this case, user need to use
share type without encryption information i.e. if share type with encryption
information is used to create share along-with user provided encryption key
ref, manila will throw error.

In either scenario, after key is being fetched it will be given to storage
back end driver. The storage back end driver then talks with key-store using
KMIP (Key Management Interoperability Protocol) and then retrieves the key
data. The key data is then used to encrypt the share's data within the storage
back end.

The actual encryption of the data at-rest is performed by the back end storage
system. The scope of Manila's involvement ends with coordinating the user's
secret with the Key Store.

This specification does not target encryption at the share server level. If a
share server has any sort of encryption settings, the expectation on the back
end storage system and its driver is that the per-share encryption settings
from Manila will override the encryption settings of the share server for the
given share. The driver can also decide whether the default encryption
satisfies the ask to encrypt or if it needs to do something, e.g. re-encrypt
with a new key.


Proposed change
===============

* New API collection for creating encryption specs

  In order to support encryption with share type, we will introduce operations
  create/update/delete/list/show for encryption specs associated with a share
  type.

* New database resource encryption

  When creating a share type, user can specify encryption information such as
  cipher, key_size, provider, control location.

* Manila API service changes

  Manila api service will allow configuration of a key manager. We will
  introduce an interface for the Manila API service to communicate with an
  external key manager (e.g. Castellan), which internally works with a key
  store (e.g. Barbican). The key manager will be configured via conf file.
  If encryption key ref is provided, API service will pass it to back end
  driver. However, if share-type with encryption information is used (i.e.
  encryption key is not provided), API service will generate key in key-store
  and the pass key ref to back end driver.

* Changes in backend driver

  A backend driver will get encryption key ref which it will pass to backend
  hardware to perform the encryption. The back end driver will fetch key data
  from key-store using key ref. The fetched key data will be used to encrypt
  the share' data.

* A generic encryption capability called "encryption_support" will be
  introduced, defaulting to False. Admins then would have to either set
  "encryption_support" to True explicitly, or specify these encryption
  options so that Manila will set it. This will help to filter the storage
  back ends that support such style of encryption.

* Things to consider for this spec

  If share is created from encryption based share-type, Manila will not permit
  the following actions on the share:

  - unmanage
  - transfer
  - backup

  In future, some of these operations might be allowed for encrypted share.


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

* Add table for 'encyption'

  +-----------------------+--------------+------+---------+------------+-------+
  | Field                 | Type         | Null | Key     | Default    | Extra |
  +=======================+==============+======+=========+============+=======+
  | encryption_id         | varchar(255) | No   | Primary | NULL       |       |
  +-----------------------+--------------+------+---------+------------+-------+
  | created_at            | datetime     | YES  |         | NULL       |       |
  +-----------------------+--------------+------+---------+------------+-------+
  | updated_at            | datetime     | YES  |         | NULL       |       |
  +-----------------------+--------------+------+---------+------------+-------+
  | deleted_at            | datetime     | YES  |         | NULL       |       |
  +-----------------------+--------------+------+---------+------------+-------+
  | deleted               | tinyint(1)   | YES  |         | NULL       |       |
  +-----------------------+--------------+------+---------+------------+-------+
  | share_type_id         | varchar(255) | NO   |         | NULL       |       |
  +-----------------------+--------------+------+---------+------------+-------+
  | key_size              | int(11)      | YES  |         | NULL       |       |
  +-----------------------+--------------+------+---------+------------+-------+
  | cipher                | varchar(255) | YES  |         | NULL       |       |
  +-----------------------+--------------+------+---------+------------+-------+
  | provider              | varchar(255) | YES  |         | NULL       |       |
  +-----------------------+--------------+------+---------+------------+-------+
  | control_location      | varchar(255) | YES  |         | 'back-end' |       |
  +-----------------------+--------------+------+---------+------------+-------+


* New field in share_instances table

  +-----------------------+---------------+------+-----+---------+-------+
  | Field                 | Type          | Null | Key | Default | Extra |
  +=======================+===============+======+=====+=========+=======+
  | encryption_key_id     | varchar(255)  | YES  |     | NULL    |       |
  +-----------------------+---------------+------+-----+---------+-------+


* New field in share_snapshot_instances table

  +-----------------------+---------------+------+-----+---------+-------+
  | Field                 | Type          | Null | Key | Default | Extra |
  +=======================+===============+======+=====+=========+=======+
  | encryption_key_id     | varchar(255)  | YES  |     | NULL    |       |
  +-----------------------+---------------+------+-----+---------+-------+


CLI API impact
--------------

Add new parameters to commands in openstackclient(OSC):

.. code-block:: bash

    openstack share type create [--encryption-provider <provider>]
                                [--encryption-cipher <cipher>]
                                [--encryption-key-size <key-size>]
                                <name>

* encryption-provider: Set the class that provides encryption support for this
  share type (e.g “LuksEncryptor”)
* encryption-cipher: Set the encryption algorithm or mode for this share type
* encryption-key-size: Set the size of the encryption key of this share type
* name: Name of share type.

.. code-block:: bash

    openstack share type show [--encryption-type] <share-type>

* encryption-type: Display encryption information of this share type.
* share-type: Share type to display (name or ID)

.. code-block:: bash

    openstack share type set [--encryption-provider <provider>]
                             [--encryption-cipher <cipher>]
                             [--encryption-key-size <key-size>]
                             <share-type>

* encryption-provider: Set the class that provides encryption support for this
  share type (e.g “LuksEncryptor”)
* encryption-cipher: Set the encryption algorithm or mode for this share type
* encryption-key-size: Set the size of the encryption key of this share type
* share-type: Share type to display (name or ID)

.. code-block:: bash

    openstack share type unset [--encryption-type] <share-type>

* encryption-type: Remove the encryption type for this share type.
* share-type: Share type to unset encryption info (name or ID)

.. code-block:: bash

    openstack share type list [--encryption-type]

* encryption-type: Display encryption information for each share type in list



REST API impact
---------------

**To create an encryption type for an existing share type**::

    POST /v2/types/{share_type_id}/encryption

Request::

    {
        "encryption": {
            "key_size": 256,
            "provider": "luks",
            "control_location":"back-end",
            "cipher": "aes-xts-plain64"
        }
    }

All fields in the ``encryption`` request are needed.

If the share-type is not known to manila, the API will respond with
``404 Not Found``. If any share is already created from share-type,
manila will not allow to create encryption type.

Response(202 Accepted)::

    {
        "encryption": {
            "share_type_id": "77eb3421-4549-4789-ac39-0d5185d68c20",
            "control_location": "back-end",
            "encryption_id": "81e069c6-7394-4856-8df7-3b237ca61f74",
            "key_size": 256,
            "provider": "luks",
            "cipher": "aes-xts-plain64"
        }
    }


**To show an encryption type for an existing share type.**::

    GET /v2/types/{share_type_id}/encryption

Response(200 OK)::

    {
        "encryption": {
            "share_type_id": "77eb3421-4549-4789-ac39-0d5185d68c20",
            "control_location": "back-end",
            "encryption_id": "81e069c6-7394-4856-8df7-3b237ca61f74",
            "key_size": 256,
            "provider": "luks",
            "cipher": "aes-xts-plain64"
        }
    }


**To update an encryption type for an existing share type.**::

    PUT /v2/types/{share_type_id}/encryption/{encryption_id}

Request::

    {
        "encryption":{
            "key_size": 64,
            "provider": "luks",
            "control_location":"back-end",
            "cipher": "aes-xts-plain64"
    }


If the share-type is not known to manila, the API will respond with
``404 Not Found``.  If any share is already created from share-type,
manila will not allow to update encryption type.


Response(200 OK)::

    {
        "encryption": {
            "share_type_id": "77eb3421-4549-4789-ac39-0d5185d68c20",
            "control_location": "back-end",
            "encryption_id": "81e069c6-7394-4856-8df7-3b237ca61f74",
            "key_size": 256,
            "provider": "luks",
            "cipher": "aes-xts-plain64"
        }
    }


**To delete an encryption type for an existing share type.**::

     DELETE /v2/types/{share_type_id}/encryption/{encryption_id}

If the share-type is not known to manila, the API will respond with
``404 Not Found``. If any share is already created from share-type,
manila will not allow deletion of the corresponding encryption type.
User must delete all shares from share-type and then can delete
encryption type.


Driver impact
-------------
The backend driver needs to implement:

* Instruct the back end storage system to Encrypt the share with key
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

None

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

* Implement encryption support in share-type APIs.
* Update share-type commands in python-manilaclient.
* Implement tempest support.

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

_`[1]`: https://review.opendev.org/q/topic:bp%252Fencrypt-cinder-volumes
