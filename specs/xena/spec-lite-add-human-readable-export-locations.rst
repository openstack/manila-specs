..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

Spec Lite: Add human readable export location support
-----------------------------------------------------

:problem: Export locations are usually too difficult to memorize.
          Currently, there is no way to determine the export location before
          the share is created, so users wait until the share creation
          request gets completed, and then they check the export locations to
          mount the share.
          The generated export locations are often not human readable and it
          is hard to memorize and control them.

:solution: We could introduce a new field for shares where users would be able
           to specify a custom export location in the share creation. It should
           be easier for them to memorize the export location of the share, in
           case there is need to mount the share again. If this field is
           specified during the creation request, the backends that support
           such functionality would be able to create multiple export locations
           for a given share, and users would be able to mount the shares
           either using the human readable export location, or other if they
           exist. It's possible backends that implement this feature may only
           be able to provide one export path.
           The new field will be called ``mount_point_name`` and it will
           be added to the shares model. This field will not accept special
           characters other than underscores. If special characters other than
           underscores are provided, the manila api service is going to raise
           HTTP BadRequest, warning that such characters are not allowed. An
           export location must be unique in a share server. So, when a request
           to create a share is received, if a ``mount_point_name`` was
           provided, the project name will be prepended to it. The number of
           characters provided by the user added to the number of characters in
           the project name can not exceed 255 characters, otherwise the
           request will be denied. It is reasonable to think that the project
           name can be easily remembered by the users, so it is still a better
           option compared to an id, and we can be sure that appending the
           project name to the custom mount point name will not drift apart
           from this proposal main goal.
           So the manila share service will look up for
           duplicate ``mount_point_name`` values and if it finds any, the
           creation will fail. It is possible that two projects in different
           domains have the same name, and users coincidentally set the same
           ``mount_point_name`` while creating a share. In such cases, the
           share will not be created and its status will be set to error. A
           user message will be created for both scenarios.
           A new back end capability called
           ``human_readable_export_location_support`` will be added and drivers
           that support such capability should report it as ``True``.
           Administrators will need to create share types with such extra spec
           set to ``True``. As the manila share API will perform validations
           using this extra spec, it must be always tenant-visible. By having
           this extra-spec, the scheduler can also filter out backends that do
           not support such functionality.
           In case of a share migration, if a new share type is provided, and
           it has ``human_readable_export_location_support=True``, the
           migration will fail in the scheduler if the chosen destination
           backend doesn't support it. If a new share type was specified and it
           differently from the former does not support custom names for export
           locations, the migration will succeed, the custom export location
           won't be created and the ``mount_point_name`` field value will be
           set to ``None``.
           Administrators will be able to specify a custom mount point name to
           be configured in the migrated share through a new parameter called
           ``--new-mount-point-name``, in the ``migration-start`` command. This
           will help administrators to avoid possible failures caused by
           duplicated custom export locations in the migration.
           So in a share creation scenario, the users will be able to create
           shares like::

                $ manila create nfs 1 --name share_name --mount-point-name custom_export_path

           And users will be able to mount shares using the custom export
           location, as in this example::

                $ sudo mount -t nfs 10.1.0.2:/project_name_custom_export_path /mnt/my_share


:impacts:

          - REST API Impact.
              - A microversion bump to the share API.
              - The API will accept the field ``mount_point_name``.

          - Documentation Impact
              - User guide
              - API Reference
              - Contributor guide

          - Database Impact
              - A new field called ``mount_point_name`` will be added to
                the ``shares`` model.

          - Python-manilaclient and OSClient impact
              - The share ``create`` command will be modified to accept the
                ``--mount-point-name`` parameter in the shares creation.
              - The ``migration-start`` command will be modified to accept the
                ``--new-mount-point-name`` parameter.

          This implementation is not supposed to impact performance on any
          aspect.

:alternative: As an alternative, drivers could reuse the name and the
              description of shares and generate human readable export
              locations, but names can be duplicated and backends may fail due
              to that.

:timeline: Include in Xena release.

:link: https://blueprints.launchpad.net/manila/+spec/human-readable-export-locations

:assignee: carloss
