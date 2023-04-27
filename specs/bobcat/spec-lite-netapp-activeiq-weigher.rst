..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

Spec Lite: Scheduler Weigher based on Netapp Active IQ
------------------------------------------------------

:problem: The Manila Scheduler weighers are generic, it is designed to
          weigh based on the backend stats, being restricted to that set of
          information.
          For some NetApp environments, it could be more helpful to have a more
          specific tool that tracks the storages status (space, cpu usage and
          several others) and use statistics and artificial intelligence
          techniques for weighing the hosts in a more efficient manner.

:solution: To make a better weight evaluation, NetApp has its own software
           called ``Active IQ (AIQ)`` [1]. The ``AIQ`` connects to the NetApp
           storages collecting and processing the status of each device
           dynamically. Using REST API endpoints, external services can request
           the ``AIQ`` to weigh the available storages for a given share
           creation. API request example::


               {
                    "method": "POST",
                    "url": "https://10.63.167.192/api/storage-provider/data-placement/balance",
                    "args": [],
                    "headers": {
                        "Accept": "application/json",
                        "Content-Type": "application/json"
                    },
                    "json": {
                        "capacity": "10GB",
                        "eval_method": 0,
                        "opt_method": 0,
                        "priority_order": [
                            "ops",
                            "latency",
                            "volume_count",
                            "size"
                        ],
                    "ssl_key": "bbda43aa-aee9-11ed-a4cd-005056bd7087",
                    "separate_flag": false,
                    "resource_keys": [
                        "1c54fcf0-2adb-11ec-86b0-d039ea2ef942:type=aggregate,uuid=b22ea874-eada-4c31-b6f9-1cf95e2bdacc",
                        "1c54fcf0-2adb-11ec-86b0-d039ea2ef942:type=aggregate,uuid=a39c5d4f-2ca9-4910-b57f-71a7936656c9",
                        "1c54fcf0-2adb-11ec-86b0-d039ea2ef942:type=aggregate,uuid=ec6a23a2-c98f-4640-8b4b-4615c1969751"
                        ]
                    }
               }


           The API response::

                {
                    "request_ok": true,
                    "status_code": 200,
                    "headers": {
                        "Expires": "0",
                        "Cache-Control": "no-cache, no-store, must-revalidate",
                        "X-Powered-By": "NetApp Application Server",
                        "Server": "NetApp Application Server",
                        "X-XSS-Protection": "1; mode=block",
                        "Pragma": "no-cache",
                        "X-Frame-Options": "SAMEORIGIN"
                    },
                    "data": [
                        {
                            "scores": {
                                "total_weighted_score": 30.0
                            },
                            "key": "1c54fcf0-2adb-11ec-86b0-d039ea2ef942:type=aggregate,uuid=a39c5d4f-2ca9-4910-b57f-71a7936656c9"
                        },
                        {
                            "scores": {
                                "total_weighed_score": 20.0
                            },
                            "key": "1c54fcf0-2adb-11ec-86b0-d039ea2ef942:type=aggregate,uuid=b22ea874-eada-4c31-b6f9-1cf95e2bdacc"
                        },
                        {
                            "scores": {
                                "total_weighted_score": 10.0
                            },
                            "key": "1c54fcf0-2adb-11ec-86b0-d039ea2ef942:type=aggregate,uuid=ec6a23a2-c98f-4640-8b4b-4615c1969751"
                        }
                    ]
                }


           The idea is proposing a vendor specific Scheduler weigher called
           ``NetAppActiveIQWeigher``, which will call the NetApp ``AIQ`` for
           getting the weight of each host. If the list of host contains at
           least one non NetApp host, the weigher is skipped.
           In order to connect to the external service, the new weigher
           requires connection configurations set by the cloud administrator,
           they are: ``aiq_username``, ``aiq_password``, ``aiq_hostname`` and
           ``aiq_port``. For more weighing flexibility, the weigher will
           contain the following optional configurations: ``aiq_eval_method``,
           ``aiq_opt_method``, ``aiq_priority_order`` and ``aiq_separate_flag``
           (see the Active IQ documentation).
           The cloud administrator may want to filter the storages according
           to its performance and storage objectives for a new workload. The
           ``AIQ`` has the concept of performance service level that provide
           this functionality and it can be informed during the weight request.
           As result, this spec is proposing to add a new NetApp scoped
           extra-spec called ``netapp:aiq_performance_level`` as a UUID string
           representing the ``AIQ`` performance service level.
           The weight request requires the set of aggregates UUIDs that will be
           evaluated. Unfortunately, the NetApp driver stats does not contain
           it, so the driver have to start to report the UUID of each pool.
           It could be collected once during driver start up, not affecting
           the driver life cycle.

:impacts:

          - NetApp Driver Impact.
              - The driver will start reporting the NetApp aggregate UUID

          - Documentation Impact
              - Admin guide
              - Contributor guide

          - Scheduler Impact
              - Propose a vendor weigher: ``NetAppActiveIQWeigher``

          - CI Impact
              - Add the new weigher for the dummy driver jobs
              - Add the new weigher for the NetApp jobs

          This implementation may impact the performance of the scheduler
          weigher phase, since the NetApp weigher will run a network request.

:alternative: There is no alternative other than keep running with generic
              weighers.

:timeline: Include in Bobcat release.

:link:
       * [1]: https://docs.netapp.com/us-en/active-iq-unified-manager/index.html

:assignee: felipe_rodrigues
