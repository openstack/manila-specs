Spec Lite: Limit number of allowed replicas per share
-----------------------------------------------------

:problem: Currently, Manila allows the creation of unlimited share replicas of
          a given share. It is a problem for both ``DHSS=True`` and
          ``DHSS=False`` modes. It allows the user to deplete the storage
          resources and have troubles depending on the configured backend.
          There is also a problem when it comes to the user visibility side on
          what backend do or do not support. Some backends do not support
          more than a limited number of share replicas, and it is not
          noticeable until the backend asynchronously denies the operation.

:solution: We should let the administrator specify the max number of replicas
           that a given share is able to have according to the backend
           capacity. In this way, the API would be able to check the number of
           existent share replicas during a new replica creation request, and
           be able to fail faster and synchronously.
           The proposed solution is to add a new extra spec to the share types
           called ``max_replicas_per_share``. Then, when creating a share
           replica, Manila will check the ``max_replicas_per_share`` value
           of the source share's share type, and will verify the number
           of existent replicas of the source share. If the number of existent
           share replicas has already reached the ``max_replicas_per_share``
           value, the request to create a new replica will be denied.
           While creating the new extra-spec for the share type the
           administrator must specify an integer value.
           A database migration will be created in order to add the
           new share type extra spec for all existent share types. In the
           database migration, a default of two replicas will be set due to
           some backends capacity. All administrators will need to update the
           share type according to their backend capacity, and will set the
           extra spec according to the number of replicas that shares created
           under the given share type are able to have. When creating a new
           share type, if the user does not specify the new extra spec, the
           service will automatically set this value to ``2`` and in case the
           administrators need to change the value, they will need to update
           the extra spec.
           The administrator will also be able to provide a new backend
           property called ``supported_replicas_per_share`` for each backend.
           The scheduler will be improved in order to filter backends
           considering this backend property.
           This solution can be combined with improvements, which
           consists of defining a project quota for share replicas using the
           existent quota system, and allow the administrator to set a default
           value for the ``max_replicas_per_share`` extra spec. So at the end,
           the whole solution would have a way to avoid this bug, alongside
           with an improvement that will help administrators to manage their
           resources.
           The solution for the bug will require the administrator
           intervention, since the database migration script will update all
           the share types with a default value, and it may block users while
           creating new share replicas after migrating the database.

:impacts:

           - REST API Impact.
               - When creating a share replica, if the share has already reached
                 the number of allowed share replicas, the API will return
                 ``403 Forbidden`` with the following message: "The share with
                 the id 'share_id' already reached the limit of replicas
                 according to its share type".
               - The API will expect an integer value while setting the
                 ``max_replicas_per_share`` extra spec, otherwise it will
                 return ``400 BadRequest`` with the following message:
                 "The specified max replicas value 'non_integer_value' must be
                 an integer and greater than or equal to 2.".

           - Documentation Impact
               - Admin guide
               - API Reference

:timeline: Include in Ussuri release.

:link: https://blueprints.launchpad.net/manila/+spec/limit-share-replicas-per-share

:assignee: carloss
