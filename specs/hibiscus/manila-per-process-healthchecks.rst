..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Per-Process Healthcheck Endpoints
=================================

`bp per-process-healthchecks <https://blueprints.launchpad.net/manila/+spec/per-process-healthchecks>`_

This spec proposes adding per-process HTTP healthcheck endpoints to Manila
services, enabling operators and orchestration frameworks to determine whether
a service is healthy and not simply alive. The design follows the
approach approved for Nova in the `per-process-healthchecks`_ specification
and aligns with the IETF Health Check Response draft [#ietf-healthcheck]_.

.. _per-process-healthchecks: https://specs.openstack.org/openstack/nova-specs/specs/2025.1/approved/per-process-healthchecks.html
.. _healthcheck.py: https://github.com/openstack-k8s-operators/manila-operator/blob/main/templates/manila/bin/healthcheck.py


Problem description
===================

Manila operators have no native, lightweight mechanism to determine whether
a Manila service (``manila-api``, ``manila-scheduler``, ``manila-share``,
``manila-data``) is healthy. Current approaches suffer from the following
limitations:

* **Service heartbeat table** (``services.report_count``): proves only that
  the periodic task loop is running, not that the service can serve requests
  or communicate with backends.

* **Log scraping**: fragile, delayed, and can't be combined well with modern
  monitoring stacks (Prometheus, Kubernetes liveness/readiness probes).

* **Oslo healthcheck middleware**: applies only to ``manila-api`` behind WSGI,
  and does not reflect the health of non-API services.

For ``manila-share`` specifically, the gap is critical: a share process can
appear "up" via heartbeat while its storage backend driver is unreachable or
not healthy.

This problem is more evident in podified deployments, where Kubernetes probes
are the primary mechanism for managing service lifecycle. In this context, the
``manila-operator`` provides a health probe script (`healthcheck.py`_) that
runs an HTTP server on port 8080 inside ``manila-share`` and
``manila-scheduler`` pod. On every HTTP GET it executes
``manila-manage service list``, queries the ``services`` table, and returns
HTTP 200 if all expected rows show ``state = :-)`` and ``status = enabled``.
This validates that the process is alive (the probe HTTP server responds) and
that the periodic heartbeat task is ticking (the ``:-)`` state is current).

However, it does **not** validate:

* Database reachability *from the service process itself* (the probe forks
  a separate ``manila-manage`` subprocess that has its own DB connection)
* RPC / message bus connectivity (a service disconnected from RabbitMQ still
  shows ``:-)`` as long as its heartbeat loop runs)
* Storage backend driver reachability (CephFS, NetApp, LVM can be completely
  unreachable while the heartbeat keeps ticking)
* The actual request processing ability (the service could be stuck, deadlocked,
  or failing every operation for specific reasons)

In a podified environment this is particularly damaging because Kubernetes
relies on probe results to make restart and traffic-routing decisions.


Use Cases
=========

* **As an operator**, I need a single HTTP endpoint per Manila process that
  tells me whether the service can serve requests: this allows to configure
  Kubernetes liveness and readiness probes with meaningful health signals.
  I need to run ``curl localhost:<port>/health | jq`` and see which subsystem
  is failing, when it last succeeded, and what the error was. This avoids the need for
  humans and tools to cross-reference logs, service lists, and backend
  connectivity manually.

* **As a deployment tool** (Kubernetes, baremetal), I need a local health check
  with no dependencies on external hosts, databases, or authentication so that
  I can integrate it with container runtime probes and service watchdogs.

* **As a monitoring operator**, I need structured, machine-parseable health
  data (per-check status, timestamps, error details) that I can scrape into
  Prometheus and use for granular alerting rules.


Proposed change
===============

Add a lightweight, in-process HTTP endpoint (``/health``) to each Manila
binary, exposing cached health state as IETF-aligned JSON. The design
inherits its core principles from the Nova specification:

1. **Passive, not active**: health state is updated as a side effect of
   normal operations (database queries, RPC calls, driver calls), not by
   active probes that add load. The ``@healthcheck`` decorator is applied
   at the manager layer and observes the outcome of calls that flow through
   drivers: drivers themselves do not need to implement health check logic (see
   `Driver impact`_).

2. **Three-state model**: ``pass``, ``warn``, ``fail`` per check, following
   the IETF Health Check Response draft.

3. **Disabled by default**: opt-in via a ``[healthcheck]`` configuration
   section. The endpoint binds to ``localhost`` or a Unix socket by default.

4. **Unauthenticated**: designed for local consumption by Kubernetes probes,
   systemd watchdogs, or human debugging via ``curl``.


Main Health check definitions
-----------------------------

``database``
  Tracks the outcome of database operations that occur naturally during
  service operation (share create, delete, manage, status updates). A
  successful DB query sets the check to ``pass``; a caught ``DBError`` or
  ``DBConnectionError`` sets it to ``fail``. For idle services (e.g.,
  ``manila-data``), the periodic heartbeat task (``report_service_to_db``,
  which runs every ~10 seconds) provides a natural source of database
  observations, keeping this check fresh even when no user-facing
  operations are in progress.

``message_bus``
  Tracks RPC connectivity by observing the outcome of actual RPC operations
  via the ``@healthcheck`` decorator. A successful RPC call sets ``pass``;
  a transient connection error sets ``warn``; a persistent failure sets
  ``fail``. For idle services that receive infrequent RPC calls (e.g.,
  ``manila-data``), enabling ``oslo.messaging``'s ``rpc_ping_enabled``
  option provides a periodic RPC ping that keeps this check fresh without
  adding synthetic load (see `Implementation details`_).

``backend_driver`` (``manila-share`` only)
  Tracks whether the configured storage backend drivers can perform
  operations. In multi-backend deployments, where a single
  ``manila-share`` process hosts more than one backend, each backend is
  tracked as a separate element within the ``backend_driver`` check
  array, identified by its ``componentId`` (the ``host@backend``
  identifier from the configuration, e.g., ``storage01@cephfs``). This
  is the same identifier visible in ``manila-manage service list``, logs,
  and the ``services`` table. This ensures that a failure in one
  backend does not mask the healthy state of another, and operators can
  pinpoint exactly which backend is degraded.
  At startup, the share manager's ``init`` method invokes each driver's
  ``_check_for_setup_error()`` to detect recoverable and irrecoverable
  failures, providing the initial ``backend_driver`` health state before
  any user-facing operations occur.
  Updated from driver calls during normal share lifecycle
  (``create_share``, ``delete_share``, ``_update_share_stats``). The periodic
  stats reporting task (``_report_driver_status``, which runs periodically
  per backend) provides a natural heartbeat for this check. It can be
  extended depending on the driver implementation to report more events.


Status definitions
------------------

``pass``
  All health indicators are functioning normally and their TTLs are valid.
  HTTP status code: **200**.

``warn``
  One or more indicators show partial or transient failures, or a TTL has
  expired but the most recent observation was ``pass``. The service may still
  be able to serve requests. HTTP status code: **200**.

``fail``
  One or more critical indicators report persistent errors, or all TTLs have
  expired with no recent successful observation. The service cannot reliably
  serve requests. HTTP status code: **503**.

The overall service status is the worst state across all registered checks:
if any check is ``fail``, the service status is ``fail``; if any check is
``warn`` (and none is ``fail``), the service status is ``warn``.

Each check entry may include an ``output`` field carrying a short,
sanitized reason for the current status (see `Implementation details`_).
The ``output`` field is omitted when the status is ``pass``.


TTL and staleness
-----------------

Each health indicator has a configurable TTL (default: 300 seconds). If no
update (pass or fail) is received within the TTL window, the indicator
transitions to ``warn`` to signal staleness. This prevents a quiet service
(one receiving no traffic) from indefinitely reporting ``pass`` based on an
observation that may no longer reflect reality.

For services that can remain idle for extended periods (notably
``manila-data``), TTL expiry would cause spurious ``warn`` transitions even
though the service is perfectly healthy. This is mitigated by leveraging
existing periodic operations that naturally refresh each check:

* ``database``: the periodic heartbeat task (``report_service_to_db``)
  runs every ~10 seconds across all services, keeping the ``database``
  check well within its TTL.
* ``message_bus``: enabling ``oslo.messaging``'s ``rpc_ping_enabled``
  option introduces a lightweight periodic RPC ping that refreshes the
  ``message_bus`` check without synthetic application-level probes.
* ``backend_driver``: the ``_report_driver_status`` periodic task already
  runs on a short interval per backend in ``manila-share``.

With these periodic sources in place, no Manila service should experience
TTL-driven staleness under normal operation.

Response format
---------------

The endpoint returns JSON following the IETF Health Check Response draft,
adapted for Manila service names. The response uses the
``application/health+json`` content type as defined by the IETF draft:

.. code-block:: json

   {
     "status": "pass",
     "version": "1.0",
     "serviceId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
     "description": "manila-share",
     "notes": {
       "host": "storage-node-1.cloud",
       "backend": "storage01@cephfs"
     },
     "checks": {
       "database": [
         {
           "status": "pass",
           "time": "2025-06-15T10:30:00Z"
         }
       ],
       "message_bus": [
         {
           "status": "pass",
           "time": "2025-06-15T10:30:00Z"
         }
       ],
       "backend_driver": [
         {
           "componentId": "storage01@cephfs",
           "componentType": "datastore",
           "status": "pass",
           "time": "2025-06-15T10:29:45Z"
         },
         {
           "componentId": "storage02@netapp",
           "componentType": "datastore",
           "status": "pass",
           "time": "2025-06-15T10:29:50Z"
         }
       ]
     }
   }

Example failure response (HTTP 503):

.. code-block:: json

   {
     "status": "fail",
     "version": "1.0",
     "serviceId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
     "description": "manila-share",
     "notes": {
       "host": "storage-node-1.cloud",
       "backend": "storage01@cephfs"
     },
     "checks": {
       "database": [
         {
           "status": "pass",
           "time": "2025-06-15T10:30:00Z"
         }
       ],
       "message_bus": [
         {
           "status": "pass",
           "time": "2025-06-15T10:30:00Z"
         }
       ],
       "backend_driver": [
         {
           "componentId": "storage01@cephfs",
           "componentType": "datastore",
           "status": "fail",
           "time": "2025-06-15T10:28:00Z",
           "output": "CephFS: mon_command failed: [errno 110] Connection timed out"
         },
         {
           "componentId": "storage02@netapp",
           "componentType": "datastore",
           "status": "pass",
           "time": "2025-06-15T10:29:50Z"
         }
       ]
     }
   }


Implementation details
----------------------

The implementation introduces a new ``manila/healthcheck/`` module containing:

**HealthcheckManager**: a singleton that maintains an in-memory dictionary of
``HealthcheckStatusItem`` objects, each tracking a named check's current
status, last-updated timestamp, TTL, and optional error output. For checks
that cover multiple components (e.g., ``backend_driver`` in a multi-backend
``manila-share``), the manager stores one ``HealthcheckStatusItem`` per
``(name, component_id)`` pair and groups them into the same IETF check
array in the response. The manager exposes
``update(name, status, output=None, component_id=None)`` for programmatic
updates and a ``get_response()`` method that computes the aggregate status
and returns the JSON payload.

**@healthcheck decorator**: a function decorator can be used to wrap existing
methods and capture health state as a side effect. On successful return, it updates
the named check to ``pass``. If a listed exception type is raised, it updates
the check to ``fail`` with the exception message. An optional
``component_attr`` parameter names an attribute on ``self`` that is resolved
at call time and used as the ``componentId`` for that check entry. This
allows a single check name (e.g., ``backend_driver``) to track multiple
components independently (e.g., one per storage backend).

.. code-block:: python

   # manila/share/manager.py
   from manila.healthcheck import healthcheck

   class ShareManager(manager.SchedulerDependentManager):

       @healthcheck('database', [db_exc.DBError, db_exc.DBConnectionError])
       def create_share_instance(self, context, share_id, ...):
           # existing implementation unchanged
           ...

       @healthcheck('backend_driver', [exception.ShareBackendException],
                    component_attr='host')
       def _report_driver_status(self, context):
           # periodic task runs every 60s per backend -- natural heartbeat
           # component_attr='host' resolves self.host at call time
           # (e.g., "storage-node-1@cephfs") and uses it as componentId
           ...

**Output field guidelines**: the ``output`` field in each check entry
carries the most recent error or warning message.
Stack traces are never included in health responses; they belong to service
logs. Manila already provides an established pattern for surfacing backend
exceptions through the "Asynchronous User Messages" feature, which converts
raw exceptions into strings with enough diagnostic hints for troubleshooting
without leaking sensitive details. The healthcheck ``output`` field must apply
equivalent sanitization. Drivers that register custom checks via the optional
extension point control their own output format, but the same principle
applies: short, actionable messages suitable for operator dashboards and
alerts.

**Message bus tracking**: RPC connectivity is tracked via the
``@healthcheck`` decorator on methods that perform RPC calls, catching
connection errors during actual operations. For services that may remain
idle for extended periods (e.g., ``manila-data``), enabling
``oslo.messaging``'s ``rpc_ping_enabled`` option is recommended: it
provides a periodic RPC ping that keeps the ``message_bus`` check fresh
without requiring user-facing traffic.

**HTTP server**: a minimal, single-threaded HTTP server (or Unix socket
listener) that serves the ``/health`` endpoint. It reads the in-memory state
from the ``HealthcheckManager`` and serializes the response. No
authentication is performed. A per-request timeout is enforced so that
idle or slow connections cannot block the server thread. The server runs
in a dedicated thread within
the service process. On graceful shutdown, the service's ``stop()`` method
(called by ``oslo.service`` on SIGTERM [#oslo-service]_) signals the HTTP
server thread to stop accepting connections and exit cleanly.
Health state is stored in-memory only and is reset on process restart.

Configuration
-------------

A new ``[healthcheck]`` configuration section is added to ``manila.conf``:

.. code-block:: ini

   [healthcheck]
   # Listen URI. Supports tcp:// and unix:// schemes.
   # Unix sockets are preferred: they avoid port conflicts when multiple
   # services run on the same host and are not exposed to the network.
   # When TCP is needed (e.g., for Kubernetes httpGet probes), customize
   # the port per service to avoid conflicts.
   # Disabled when empty (default).
   # uri = unix:///run/manila/manila-share.sock
   # uri = tcp://localhost:9399

   # How long (seconds) a health indicator remains valid before going stale.
   # Default: 300
   # ttl = 300

   # HTTP Cache-Control max-age header value (seconds).
   # Default: 10
   # cache_control = 10

   # Enable backend driver health check (manila-share only).
   # Default: true
   # backend_check = true


Integration with Kubernetes
---------------------------

When Manila is deployed in Kubernetes, the healthcheck endpoint enables proper
probe configuration:

Kubernetes defines three probe types with distinct semantics:

* **Startup probe**: gates liveness and readiness probes until the service
  has completed initialization (driver setup, ``_check_for_setup_error()``,
  reconciliation). Neither liveness nor readiness probes run until the
  startup probe succeeds.
* **Liveness probe**: determines whether the service is stuck or
  deadlocked and should be restarted.
* **Readiness probe**: determines whether the service can accept traffic.
  A failing readiness probe removes the pod from Service endpoints without
  restarting it.

All three probes use the same ``/health`` endpoint. The differentiation
is achieved through HTTP status codes and probe thresholds: when any
check is ``fail``, the endpoint returns HTTP 503; otherwise it returns
HTTP 200. The startup probe uses a generous budget
(``failureThreshold: 30``, i.e., ~300 seconds) to allow driver
initialization and reconciliation to complete. The readiness probe uses
tight thresholds to quickly remove a degraded pod from Service endpoints
(stop routing traffic), while the liveness probe uses lenient thresholds
(``failureThreshold: 3``) so that only sustained failures trigger a pod
restart.

.. code-block:: yaml

   # Startup: generous — allow time for driver init and reconciliation
   startupProbe:
     httpGet:
       path: /health
       port: 9399
     periodSeconds: 10
     failureThreshold: 30

   # Liveness: lenient — only restart on sustained failure
   livenessProbe:
     httpGet:
       path: /health
       port: 9399
     periodSeconds: 10
     failureThreshold: 3

   # Readiness: tight — quickly stop traffic to degraded pods
   readinessProbe:
     httpGet:
       path: /health
       port: 9399
     periodSeconds: 5

Future revisions may introduce separate ``/livez`` and ``/readyz``
endpoints following the pattern established by the Kubernetes API server,
if operational experience shows that the single-endpoint approach is
insufficient for distinguishing startup, liveness, and readiness
semantics.

This replaces the current approach described in the first section of the
spec, and instead of forking a ``manila-manage`` subprocess on every probe
invocation it is possible to query the dedicated endpoint.


Alternatives
------------

**Extend the existing Oslo healthcheck middleware**: the Oslo middleware only
works with WSGI-hosted services (``manila-api``). It cannot be attached to
RPC-based services (``manila-share``, ``manila-scheduler``, ``manila-data``),
which are the services with the most critical health monitoring gaps.
Additionally, the existing middleware leaks system information and does not
support the three-state model.

**Active health probing (synthetic operations)**: instead of passively
observing real operations, the service could periodically execute synthetic
health checks (e.g., a test database query, a test RPC call). This adds load,
introduces potential failures, and requires tuning intervals based on the system
size.

**External health aggregator service**: a dedicated service that polls each
Manila service for health data. This introduces a new dependency, a new
failure mode, and additional network traffic. The in-process approach
eliminates these concerns.

**Shared Oslo library**: the healthcheck framework could live in
``oslo.service`` rather than being implemented per project. This is a
desirable long-term outcome and is not mutually exclusive with this spec.
If Nova's implementation lands first and is generalized into Oslo, Manila
can adopt the shared library, but it would require additional coordination.
The Manila-specific aspects (backend driver checks) would
remain in Manila regardless.


Data model impact
-----------------

None. Health state is maintained in-memory within each service process and
is not persisted to the database. Process restart clears all health state,
which is re-established from subsequent real operations.


REST API impact
---------------

No changes to the existing Manila public REST API.

The healthcheck endpoint (``/health``) is a separate, unauthenticated HTTP
server running on a distinct port or Unix socket. It is not part of the
Manila API service and is not versioned with the Manila API microversion
scheme.
In addition, the healthcheck endpoint is disabled by default, with no impact
for existing deployment where other approaches are used.


Driver impact
-------------

No mandatory driver changes. The ``@healthcheck`` decorator is applied to
share manager methods that call into drivers, not to the drivers themselves.
Driver exceptions are caught by the decorator and used to update health state.

An optional extension point allows drivers to register custom health checks
if they have domain-specific health signals (e.g., CephFS checking cluster
health via ``ceph status``). This is not required for the initial
implementation.


Security impact
---------------

* The healthcheck endpoint is **disabled by default**. It must be explicitly
  enabled via the ``[healthcheck]`` configuration section.

* When enabled, the endpoint is **unauthenticated** and binds to
  **localhost** (``127.0.0.1``) by default. This limits access to processes
  on the same host (Kubernetes probes, human debugging via ``oc exec``).

* The endpoint can optionally be bound to ``0.0.0.0`` or a specific IP address
  for external consumers (Prometheus, operator reconciler). This is an explicit
  opt-in configuration and should be properly documented.

* The response body might contain service-level metadata (hostname, backend name,
  error messages). While this information is not privileged in a typical
  deployment, binding to ``0.0.0.0`` in an untrusted network should be
  avoided. Documentation will recommend the best policies to restrict access
  when exposing the endpoint beyond localhost.

* Because the endpoint is unauthenticated, the ``output`` field must never
  contain raw exception messages, credentials, connection strings, or
  backend internals. All error output is sanitized using the same approach
  established by Manila's Asynchronous User Messages feature (see
  `Implementation details`_).


Notifications impact
--------------------

None.


Other end user impact
---------------------

None. The healthcheck endpoint is an operator-facing feature. End users
interact with Manila through the public API, which is unaffected.


Performance Impact
------------------

**Minimal**. The ``@healthcheck`` decorator adds a dictionary update
(setting a status enum and timestamp) on the success or failure path of
decorated methods. This is a constant-time, in-memory operation with
negligible overhead.

The HTTP server is single-threaded and serves only ``/health`` requests.
Under normal Kubernetes probe intervals (every 5-10 seconds), this amounts
to a trivial workload.

Health state reads by the HTTP handler are not locked -- eventual consistency
between the decorator updating state and the handler reading it is
acceptable for health reporting purposes.

The approach **eliminates** the per-probe subprocess fork overhead of the
current ``healthcheck.py`` sidecar, which spawns a ``manila-manage`` process
on every invocation. This is a net performance improvement.


Other deployer impact
---------------------

* New configuration section ``[healthcheck]`` with options described in
  `Configuration`_.
* The feature is disabled by default; no action required for deployments
  that do not opt in.
* Podified deployments using ``openstack-k8s-operators`` will benefit from
  operator-level integration once the ``manila-operator`` is updated to
  configure healthcheck URIs and Kubernetes probes.


Developer impact
----------------

Developers adding new Manila service functionality that interacts with
databases, RPC, or storage backends should apply the ``@healthcheck``
decorator to relevant methods. This is a lightweight annotation that does
not change method signatures or behavior.


Upgrade impact
--------------

None. The healthcheck endpoint is disabled by default and has no effect
on existing deployments that do not opt in.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Francesco Pantano (fmount)

Other contributors:
  Goutham Pacha Ravi (gouthampravi)

Work Items
----------

1. **Core framework** (``manila/healthcheck/`` module):

   * ``HealthcheckManager`` class with in-memory state store
   * ``@healthcheck`` decorator
   * HTTP server
   * ``[healthcheck]`` configuration options registration

2. **manila-share integration**:

   * Apply ``@healthcheck('database', ...)`` to key share manager methods
   * Apply ``@healthcheck('backend_driver', ...)`` to driver-calling methods
     and ``_report_driver_status``

3. **Other service integration**:

   * ``manila-scheduler``: ``database`` and ``message_bus`` checks
   * ``manila-api``: ``database`` and ``message_bus`` checks (alongside or
     replacing Oslo healthcheck middleware)
   * ``manila-data``: ``database`` and ``message_bus`` checks


Dependencies
============

None


Testing
=======

Unit tests
----------

* ``HealthcheckManager``: state transitions, TTL expiry, per-component
  tracking, and aggregate status computation.
* ``@healthcheck`` decorator: pass/fail updates, exception re-raising,
  ``component_attr`` resolution.
* HTTP server: response format, status codes, and ``Cache-Control`` header.

Tempest tests
-------------

* Integration test in a multi-service deployment: verify that each Manila
  service exposes a healthy ``/health`` endpoint after successful startup and
  a valid JSON response is returned.
* No negative testing via Tempest (inducing failures requires infrastructure
  manipulation not suited to Tempest).


Documentation Impact
====================

* **Admin guide**: new section on configuring and using per-process
  healthchecks and security considerations for exposing the endpoint beyond
  localhost.

* **Configuration reference**: document all ``[healthcheck]`` options.


References
==========

.. [#ietf-healthcheck] IETF Health Check Response Format for HTTP APIs
   (draft-inadarei-api-health-check):
   https://datatracker.ietf.org/doc/html/draft-inadarei-api-health-check

.. [#oslo-service] oslo.service usage — ServiceBase lifecycle and graceful
   shutdown:
   https://docs.openstack.org/oslo.service/latest/user/usage.html
