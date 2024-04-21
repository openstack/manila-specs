..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

Spec Lite: Pass resource metadata updates to backend drivers
------------------------------------------------------------

:problem: Currently, Manila allows a user to perform add/update/delete
          operations on resource metadata. This includes resources such as
          share, snapshot and share network subnet etc. However these
          operations only handle changes in db assuming resource metadata can
          only be used for tagging purposes. So if a customer wants to perform
          any backend driver operation on a resource based on resource
          metadata, Manila needs to introduce a new API.

:solution: This effort of adding new APIs for resources can be eliminated by
           extending existing metadata add/update/delete operations to handle
           db operations as well as pass metadata updates to backend driver.
           For resources like snapshot and share network subnet, the updated
           metadata will be passed to backend driver.
           For resource like share, in addition to passing updated metadata to
           backend driver, few changes will be introduced to decide which
           metadata it can pass to backend driver.

           Flow -
           1. New driver interface `update_share_from_metadata` will be added.
           New config option `driver_updatable_metadata` will be introduced in
           default section of manila.conf. This contains comma separated list
           where each element is of format <driver>:<driver_updatateble_key>.
           Please note, this contains both admin and non-admin metadata keys.
           2. The share type extra-specs will determine the initial
           configuration of a share. It will contain values in the format of
           <driver>:<driver_updatateble_key>. During share create API, Manila
           will copy the share type extra-specs to share metadata. Only keys
           present in config option `driver_updatable_metadata` are copied
           along-with their values in share metadata.
           3. If metadata is provided in share create API, possible actions
           are as below -

             - If provided metadata is present in `admin_only_metadata`,
               Manila will check if policy allows the user to add/update that
               metadata and if yes, it will check if its present in share
               metadata (i.e. copied from share type extra-spec). If present
               in share metadata, db operations are performed and Manila will
               pass metadata to backend driver. If not present, only db
               operations are performed.
             - If provided metadata is not admin-only, the Manila will check
               if its present in share metadata (i.e. copied from share type
               extra-spec). If present in share metadata, db operations are
               performed and Manila will pass metadata to backend driver.

           4. During metadata update API as well, first validations and then
           db operations are performed. After this updated metadata will be
           passed to backend driver.
           5. Finally backend driver will peform some action based on passed
           metadata. If action results in success or failure, the user will be
           notified through the user message API. It might be possible that a
           db change gets applied successfully, but respective backend driver
           operation results in failure. Thus notifying the end user using
           message API becomes important.

           Use cases -
            1. For example, NetApp ONTAP driver needs to set snapshot policy
            for a given share. Instead of creating a new API to set/unset
            snapshot policy, user can add snapshot policy as share metadata
            i.e. key=value pair. The metadata is then passed to ONTAP driver
            by manila-api service using rpc calls. The ONTAP driver then sets
            the snapshot policy for this share and notifies a success/failure.
            2. Also, not all share specific operations are allowed for regular
            user. By using `admin_only_metadata` along-with driver updatable
            metadata, admin can control the share behaviour. For example,
            NetApp ONTAP cloud admin does not want end user to see root path
            (/) of a share due to security concerns. This is possible by
            setting the show-mount option of share server NFS. In this case as
            well, instead of creating new API to set/unset show-mount option,
            admin can add show-mount option as metadata of share resource. The
            ONTAP driver then sets the show-mount option on the share server
            belonging to share.

:impacts:

          - REST API Impact.
              - The update metadata behavior will issue an RPC call to the
                backends when necessary to allow the share manager to process
                metadata updates via the driver. However, no request/response
                schema or response error codes will be changed as part of this
                implementation.

          - Documentation Impact
              - User guide
              - Admin guide

          - Database Impact
              - None

          - Python-manilaclient and OSClient impact
              - None

:alternative: As an alternative, Manila could create separate APIs for
              handling various resource related operations.

:timeline: Include in 2024.2 release.

:link: https://blueprints.launchpad.net/manila/+spec/pass-resource-metadata-updates-to-backend-drivers

:assignee: kpdev
