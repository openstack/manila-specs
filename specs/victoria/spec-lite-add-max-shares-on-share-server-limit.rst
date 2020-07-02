..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

Spec Lite: Add new share server limits
--------------------------------------

:problem: An administrator is not able to specify how many shares can be
          created in a given share server, nor the maximum size that a share
          server can hit. The current behavior allows the system to allocate
          a bunch of shares in a single share server and it allows the share
          server to reach a huge size, making it a bit harder to manage.

:solution: We should add a limit for the amount of shares that a share
           server can hold. Administrators can configure this limit for each
           backend, and have more control over the size of share servers,
           which helps them to manage the cloud resources. The proposed
           solution introduces two new backend properties called
           ``max_shares_per_share_server`` and ``max_share_server_size``,
           whereby the administrators are able to determine the amount of
           shares that can be created upon a given share server.
           Configuring the ``max_shares_per_share_server`` property means
           that a share server which reached the limit of shares will be
           filtered out of the compatible share servers list in the share
           manager.
           For the ``max_share_server_size`` backend capability, when a given
           share server hits the specified amount of gigabytes, the share
           manager will filter the share server out of the compatible share
           server list, and a new share server is going to be provided.
           In this case, not only shares are going to be considered in the
           final gigabytes amount, but share replicas and snapshots will be
           considered as well.
           If one of these backend capabilities is not set in the chosen
           backend, the share manager will understand that there is no limit
           for shares or gigabytes in that backend share servers.
           The share manager ``provide_share_server_for_share`` and
           ``provide_share_server_for_share_group`` methods will be modified
           to also check if there is a configured limit in the corresponding
           backend session in the Manila configuration file.
           These new limits can be implemented as mutable, which means an
           administrator won't need to restart any service to have the
           changes applied.
           If at least one of the properties was specified, the share manager
           will query the amount of existent share server resources using the
           database layer for performance purposes, and finally, if needed,
           request the creation of a new share server to place the received
           request.
           When the limits are reached, no exceptions are being raised and
           the users or admins won't be impacted by this. The share manager
           itself will handle with the situation by providing a new share
           server.

:impacts: This implementation slightly impacts performance only when at least
          one of the backend capabilities was set, but there is no expectation
          to increase significantly the share creation time, specially compared
          to another possible approach where we ask the drivers to calculate
          the amount of resources in a given share server.
          Administrators must be aware that when configuring one of these
          limits more network allocations may be needed.

:alternative: As an alternative we could use share type extra specs to provide
              driver specific limits for share server shares and max share
              server size in gigabytes. This approach would work in a project
              level, but we should consider that it may require the share
              manager to forward more information to the driver than we do
              today, in order to avoid driver calls to the storage which would
              cause a performance deterioration.

:timeline: Include in Victoria release.

:link: https://etherpad.opendev.org/p/victoria-ptg-manila
       https://review.opendev.org/#/c/510542/

:assignee: carloss

