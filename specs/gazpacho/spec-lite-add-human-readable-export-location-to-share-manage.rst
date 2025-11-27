..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

Spec Lite: Add human readable export location to share manage
-------------------------------------------------------------

:problem: Currently, share supports human readable export locations
          by ``mount_point_name`` field.
          But, there is no way to determine the export location
          before the share is managed, and human readable export locations
          are not available for managed share.
          With the current logic, the managed share is newly mounted only
          with the default template, such as volume_name in NetApp driver.

:solution: We could introduce a new optional field for managing shares where
           users would be able to specify a custom export location just like
           in the share creation. It should be easier for them to memorize
           the export location of the share, in case there is need to
           mount the share again. If this option is not provided, the managed
           share will be mounted with the default template as the existing
           workflow. And if this field is specified during the manage share
           request, the backends that support such functionality would be able
           to create multiple export locations for a managed share with that
           value, and users would be able to mount the shares either using
           the human readable export location, or other if they exist.
           It's possible backends that implement this feature may only be able
           to provide one export path.
           The new field will be called ``mount_point_name``. This field will
           not accept special characters other than underscores or hyphens.
           If special characters other than underscores or hyphens are
           provided, the manila api service is going to raise HTTP BadRequest,
           warning that such characters are not allowed. When a request to
           manage a share is received, if a ``mount_point_name`` was provided,
           share type's ``provisioning:mount_point_prefix`` spec or
           ``default_mount_point_prefix`` configuration will be prepended
           to it. The number of characters provided by the user added to the
           number of characters in the prefix can not exceed 255 characters,
           otherwise the request will be denied.
           So the manila share service will look up for duplicate
           ``mount_point_name`` values and if it finds any, share manage will
           fail. It is possible that users coincidentally set the same
           ``mount_point_name`` while managing a share. In such cases,
           the share will not be managed and its status will be set to
           manage_error. As same as share creation, administrators will
           need to create share types with ``mount_point_name_support``
           extra spec to ``True``. As the manila share manage API will perform
           validations using this extra spec, it must be always
           tenant-visible. By having this extra-spec, the scheduler can also
           filter out backends that do not support such functionality.

:impacts:

          - REST API Impact.
              - A microversion bump to the share manage API.
              - The API will accept the field ``mount_point_name`` as an
                option.

          - Documentation Impact
              - API Reference

          - Database Impact
              - None

          - Python-manilaclient and OSClient impact
              - The share ``adopt`` command will be modified to accept the
                ``--mount-point-name`` parameter as an option.

          This implementation is not supposed to impact performance on any
          aspect.

:alternative: As an alternative, drivers could reuse the name and the
              description of shares and generate human readable export
              locations, but names can be duplicated and backends may fail due
              to that.

:timeline: Include in Gazpacho release.

:link: https://blueprints.launchpad.net/manila/+spec/manage-with-mount-point-name

:assignee: eunkyung(ek121.kim@samsung.com)
