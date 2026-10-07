---
RFC: 0009
Title: Log escalation levels – reserve ERROR for problems in the service itself
Author(s): Pim van Nierop (@pvannierop)
Status: Draft
Created: 2026-10-06
Updated: 2026-10-07
Discussion: <pre-RFC issue to be opened>
---

Summary
-------
An ERROR in a RADAR-base backend service should tell a developer that the application has a problem. Today it
doesn't: ERROR log lines, which are sent to Sentry, are dominated by events that are not application problems: a
phone that loses its connection mid-upload, an expired token, invalid content, or a dependency that is briefly
unavailable. Real application errors drown in them, so developers are poorly informed about the problems they
should fix. This RFC proposes a platform-wide policy: **ERROR is reserved for problems in the service itself** (bugs
and misconfiguration). Incorrect use is INFO, naturally occurring events are INFO or DEBUG, and dependency outages
are WARN. It also proposes how to apply the policy, starting with the shared radar-jersey library and the
RADAR-Gateway.

Motivation
----------
- **ERRORs have lost their meaning.** The Sentry board is meant to inform developers: every event should be
  something a developer has to look at. Today it is dominated by client faults, disconnects and dependency
  outages, so a genuine bug is easy to miss, and developers learn to ignore the board. Logging ERROR for expected
  events also hides *how* the application failed: a Kafka outage shows up as hundreds of identical "500" stack
  traces instead of one clear message.
- **Developers can't trust the level.** When an ERROR can be anything from a typo in a client request to a
  bug, every event has to be investigated to find out which. A strict level policy makes the level itself
  informative, in Sentry and in pod logs alike.
- **Correct HTTP semantics.** The same misclassification also produces wrong status codes. For example, a
  schema-registry outage is returned to the mobile app as a 400 (client error), so the app may drop data it
  should retry.

An analysis of RADAR-Gateway 0.9.5 with radar-jersey 0.12.9 found the following.

1. radar-jersey's exception mappers (`HttpApplicationExceptionMapper`, `WebApplicationExceptionMapper`,
   `JsonProcessingExceptionMapper`) log **ERROR for every response status**, including 401 (expired token), 404,
   413, 415 and 422. Every Jersey-based service inherits this behaviour.
2. A client that disconnects while uploading causes Jersey to map the resulting `EOFException` / "Connection reset"
   to `UnhandledExceptionMapper`: **ERROR "500" with a full stack trace**. On the binary endpoint the gateway
   logs a second ERROR on top.
3. When the client disconnects while the response is being written, only the exact message
   `"Connection is closed"` is recognised. "Broken pipe" and "Connection reset by peer" are logged as ERROR with
   stack trace. Jersey also logs a `SEVERE` message for it, which the JUL bridge turns into an ERROR.
4. When Kafka is down, `producer.send().get()` throws an `ExecutionException` that is not unwrapped. The
   gateway's Kafka error handling never runs, so **every request logs an ERROR "500" with a stack trace**.
5. Schema-registry outages surface as 400 or 422 responses (client errors), and are logged at ERROR.
6. Several places log an ERROR and then throw an exception that the mapper logs again (duplicate events).
7. Jackson parse errors are logged with a snippet of the request body (possible personal data).

Not every 4xx is noise, though. On a production deployment, a 413 from the gateway's `maxRequestSize` limit
(24 MiB) appeared in Sentry and made an operator raise the limit. Until then, phones with a large backlog could
not upload: the app resends the same rejected batch, so that topic's data stays stuck on the device. That event
must stay visible. It is not a client fault but a sign that a platform limit doesn't fit the deployment.

Measurable goals:
- No ERROR events from client faults, client disconnects or dependency outages in the services that adopted the
  policy: every remaining Sentry issue points at a bug or a misconfiguration.
- Each kind of failure is logged once, with a message that says what went wrong (no duplicate events, no
  generic "500" for a known cause).
- Dependency outages return 5xx (503/504) and client faults 4xx.

Non-Goals
---------
- Changing the Sentry setup (DSN, sampling, quotas) or replacing Sentry.
- Alerting on dependency outages. That remains the job of the monitoring stack (Prometheus / Alertmanager).
- Third-party components (Kafka, PostgreSQL, …): their logs don't go to RADAR-base's Sentry.
- The mobile apps. They may benefit from the corrected status codes, but no app changes are proposed.

Guide-level explanation
-----------------------
When writing or reviewing code that logs, choose the level by asking *who has to act*:

| Level | Use for | Examples | Stack trace |
|---|---|---|---|
| **ERROR** | A bug in the service, or a deployment misconfiguration only a developer/operator can fix | unhandled exception (500), serializing already-validated data fails, Kafka authentication/authorization failure, invalid configuration at startup, a valid request rejected by the platform's own limits (413 from `maxRequestSize`, see rule 8) | yes |
| **WARN** | Degraded operation the service survives: a dependency is down or slow, a retry or fallback happened | Kafka, schema registry, S3 or Management Portal unreachable or timing out (502/503/504), request timeout | no (DEBUG may log it) |
| **INFO** | Incorrect use by a client, and natural events | 4xx (bad input, missing/expired token, forbidden, not found, invalid content; not 413, see rule 8), client disconnected | no |
| **DEBUG** | Expected noise and details of the above | stack traces of non-errors, client aborted while the response was written | – |

Rules:

1. **Classify by cause, not by exception type.** An `IOException` can be a client disconnect (INFO), a corrupt
   body (INFO, 400) or a dependency outage (WARN, 503).
2. **The HTTP status follows the same split.** A dependency outage is never a 4xx; a client fault is never a 5xx.
3. **Log or throw, never both.** If an exception mapper logs the thrown exception, don't log it before throwing.
4. **Dependency outages are WARN per request.** If a Sentry event for an outage is wanted, log it once on the
   state transition, never per request.
5. **No request bodies or personal data in log messages.**
6. **Attach the stack trace only to ERROR**, and no `printStackTrace()`.
7. **Configuration filters are a fallback** for third-party loggers whose code we can't change. Raising the
   Sentry threshold or sampling hides real errors too, so it's not the fix.
8. **The platform's own limits are configuration, not client fault.** When a well-behaved client (our own app)
   sends a valid request that the service rejects because of a configured limit (request size, record count,
   timeout), the limit doesn't fit the deployment: an operator has to act, and until then data stays stuck on the
   device. That is ERROR, with the limit and the affected resource (e.g. topic) in the message.
   - A single item over the limit (nothing the client can split) stays ERROR.
   - A batch over the limit stays ERROR only while the client can't recover. Once the client splits a rejected
     batch and retries by itself, it is WARN.
   - Limits also get a metric (rejection counter, size histogram), so monitoring can warn while requests are
     *approaching* the limit. Sentry only reports after the fact.

Example: a phone uploads a binary record set and loses its connection halfway.

- Today: `ERROR [400] Invalid RecordSet content: java.io.EOFException` and
  `ERROR [400] POST topics/... bad_content`, i.e. two Sentry events.
- After: `INFO [400] POST topics/...: client disconnected (EOFException)`, i.e. no Sentry event.

Reference-level design
----------------------
### radar-jersey (shared; benefits every Jersey service)

- A helper `Throwable.isClientDisconnect()` walks the cause chain for `EOFException` (including Jackson's
  `JsonEOFException`), a Grizzly read `TimeoutException`, and `IOException` messages "Connection is closed",
  "Connection reset by peer", "Broken pipe", "Locally closed" and "Remotely closed".
- A shared level selection for the mappers: 4xx → INFO, except 413 → ERROR (rule 8); 502/503/504 → WARN; other
  5xx → ERROR with the exception attached.
- `HttpApplicationExceptionMapper`, `WebApplicationExceptionMapper`: use the shared level selection.
- `JsonProcessingExceptionMapper`: INFO; log the exception class and `originalMessage` only, without the source
  location, so no body snippet is logged or returned.
- `UnhandledExceptionMapper`: a client disconnect → INFO with status 400. Anything else stays ERROR with stack
  trace; that is exactly what Sentry is for.
- `ClientAbortExceptionWriterInterceptor`: a client disconnect → DEBUG; anything else stays ERROR.

### RADAR-Gateway

- Unwrap the `ExecutionException` from Kafka `send().get()` so the existing Kafka error handling runs.
- Kafka timeout or unavailability → 503/504 (WARN). Authentication, authorization and fenced-producer failures
  stay ERROR (misconfiguration). A `SerializationException` caused by a schema-registry failure → 503 (WARN);
  any other serialization failure stays ERROR, because the content was already validated.
- Binary upload: split the broad `catch (IOException)` into client disconnect (INFO), schema-registry failure
  (503, WARN) and corrupt content (400, INFO).
- Avro processing: schema-registry failures → 503; incompatible schemas (`SchemaValidationException`) → 422.
  Don't report registry failures as "schema not found".
- Kafka admin errors (topic listing) → WARN without stack trace; S3 errors → WARN instead of
  `printStackTrace()`.
- Remove duplicate log lines and a dead error branch.
- 413 from `maxRequestSize` (`SizeLimitInterceptor` / `LimitedInputStream`, also applied after decompression) stays
  ERROR; its message names the limit, the topic and the bytes read.
- `log4j2.xml`: a filter on the Sentry appender that drops Jersey's `SEVERE` message for a client disconnect
  after the response was committed.

### Other services

The same analysis is repeated per service, one at a time. Expected follow-ups so far:
- radar-auth (ManagementPortal): `JwksTokenVerifierLoader` logs ERROR for every unsupported key on each JWKS
  refetch → WARN, once per key id.
- Spring-based services (ManagementPortal, Appserver): the same policy applied to their exception handlers.

### Follow-ups for the request size limit

- **Upload clients split a batch on 413.** radar-commons-android caps an upload at 1000 records and 5 MB of
  *cache* bytes, which doesn't bound the size on the wire, and on failure it resends the same batch. It should
  halve the batch and retry, down to one record; a single record still over the limit is reported app-side. Once
  released, the gateway's 413 becomes WARN (rule 8). Other upload clients (questionnaire app, iOS) need the same.
  This may need its own RFC in the `mobile` area.
- **One source for the size limit in the radar-gateway chart.** The ingress annotation
  `nginx.ingress.kubernetes.io/proxy-body-size: 24m` (compressed body) and `serverProperties.maxRequestSize`
  (decompressed body) are set separately. nginx's 413s never reach the gateway log or Sentry, so raising only
  `maxRequestSize` can leave uploads failing silently. Derive the annotation from `maxRequestSize`.
- **Gateway metrics.** The gateway has no application metrics endpoint yet. A counter of rejected oversized
  requests per topic and a request-size histogram need one; that is a separate change from the log levels.

Compatibility and migration
---------------------------
- **No API change for valid traffic.** Some failure responses get a more correct status:
  - dependency outages change from 400/422/500 to 503/504;
  - client disconnects change from 500 to 400 (the client is gone, so it rarely sees it).
  Clients that retry on 5xx will now correctly retry outages.
- **413 keeps its ERROR level and status for now.** It moves to WARN once the upload clients split rejected
  batches.
- **Logs:** dashboards or alerts that count ERROR lines in pod logs will see fewer. Alerting on outages should use
  metrics, not log levels.
- **Rollout order:** radar-jersey release (0.12.10), then each service bumps it and applies its own changes, then
  the charts and RADAR-Kubernetes pick up the new images. This follows the usual release order (`dev` →
  `release-X.Y.Z` → default branch).

Alternatives considered
-----------------------
- **Raise the Sentry threshold or sample events.** It's cheap, but it drops real errors as well as the noise,
  which makes developers even less informed. Rejected.
- **Filter events in Sentry (inbound filters / `beforeSend`).** This treats the symptom, needs maintenance per
  message pattern and keeps the wrong HTTP statuses. It's used only as a fallback for third-party loggers.
- **Per-deployment log4j2 overrides.** Every deployer would have to repeat them, and they can't tell a disconnect
  from a bug. Rejected.
- **Unwrap `ExecutionException` in radar-commons `suspendGet`.** This fixes all callers at once but changes the
  library's contract. It's kept as an open question; the gateway unwraps locally for now.

Operational considerations
--------------------------
- Primary measure: the share of Sentry issues that turn out to be real application problems. Review the issues
  per service before and after the rollout. A side effect is a much lower event volume, which also keeps
  deployments within their Sentry quota (one deployment currently exhausts its monthly quota in two days).
- Pod logs keep INFO and WARN lines, so operational visibility doesn't drop.
- Rollback: deploy the previous image or chart version. The change has no persistent state.

Security and privacy
--------------------
- Removing Jackson's source location from log messages and error responses stops request-body fragments (which
  may contain participant data) from reaching logs and Sentry.
- Authentication and authorization failures from clients (401/403) move to INFO. Brute-force or abuse detection
  must not rely on Sentry; it belongs in the ingress/monitoring layer. A misconfigured service-to-service
  credential (e.g. Kafka authentication) stays ERROR.

Testing strategy
----------------
- Unit tests in radar-jersey asserting the log level per status and for a client disconnect (log4j2 test
  appender).
- Gateway unit and integration tests for the reclassified paths.
- An end-to-end run on a local k3d cluster, counting ERROR lines in the gateway log for:
  - a truncated upload (half the declared `Content-Length`, then close) → no ERROR;
  - a client closing the connection while the response is written → no ERROR;
  - missing/expired token, unknown topic, invalid content → no ERROR, correct 4xx;
  - Kafka scaled to zero → WARN and 503/504, no ERROR;
  - schema registry scaled to zero → WARN and 503, no 400/422;
  - a body over `maxRequestSize` → ERROR 413 naming the limit and topic; a body over the ingress
    `proxy-body-size` → nothing in the gateway log (documents the gap);
  - a genuinely unhandled exception → still ERROR with stack trace.

Open questions
--------------
1. Should a dependency outage produce **one** Sentry event on the state transition (e.g. from the health
   check), or none at all, leaving it to monitoring?
2. Should client 401/403 be INFO, or WARN to keep them more visible in pod logs?
3. Unwrap `ExecutionException` in radar-commons `suspendGet` for all callers?
4. Is app-side batch splitting on 413 part of this RFC, or a separate RFC in the `mobile` area?
5. Which other limits fall under rule 8 (e.g. the 30 s request timeout for slow uploads, record-count limits)?
6. Should the policy be enforced in CI (e.g. a lint rule against `printStackTrace()` or logging ERROR in
   exception mappers for 4xx)?

References
----------
- RADAR-Gateway: https://github.com/RADAR-base/RADAR-Gateway
- radar-jersey exception mappers: https://github.com/RADAR-base/radar-jersey (`org.radarbase.jersey.exception`)
- Detailed per-site inventory of the gateway analysis: RADAR-Kubernetes
  `.claude/skills/platform-escalation-level/proposals/radar-gateway.md`
