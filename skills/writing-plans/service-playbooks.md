# Service Playbooks — Classification and Non-Functional Checklists

`writing-plans` uses this reference when a plan targets real repositories, which can be written in any language and any shape. A task list for an API server and a task list for a mobile app should not look alike — this file exists so the skill doesn't default to one shape (usually a backend-server shape) for everything.

## Step 1 — Classify each target repo

Classify independently per repo (in multi-repo mode, one plan can span several different types at once — never blend their checklists).

**Prefer a stated answer over an inferred one.** If the repo's own `CLAUDE.md`, `AGENTS.md`, or `README.md` says what it is ("this is a Kafka consumer", "this service exposes a gRPC API"), use that. Only fall back to inferring from structure when nothing states it.

| Type | Signals |
|---|---|
| **api-service** | A server framework in the manifest (Express/Fastify/NestJS, Spring Boot, Gin/Echo, FastAPI/Django, gRPC service definitions); a routes/controllers/handlers directory; a Dockerfile `CMD`/`ENTRYPOINT` that starts a listening process; README describing endpoints. |
| **worker / consumer** | Consumes from a queue, topic, or stream (Kafka consumer group, SQS/RabbitMQ listener, Pub/Sub subscriber) or runs a polling/scheduled loop; little to no inbound HTTP beyond a health check; Dockerfile runs a consumer/worker entrypoint; README mentions "consumer", "worker", "processor", "job". |
| **frontend-web** | `package.json` with React/Vue/Angular/Svelte and a bundler (webpack/vite/Next.js); no server-side business-logic request handling; deploys as static assets or SSR pages. |
| **mobile** | Kotlin/Java + `build.gradle` + `AndroidManifest.xml`; Swift + `.xcodeproj`/`Package.swift`; Dart + `pubspec.yaml`; React Native/Expo project layout. No server process. |
| **batch / cron / job** | Single-run entrypoint invoked on a schedule or manually; no long-lived listener; often under `jobs/`, `scripts/`, `cron/`, or an orchestrator DAG (Airflow, Temporal workflow). |
| **library / sdk** | Published as a package/artifact; no runnable entrypoint of its own; consumed by other repos. |
| **unknown / other** | None of the above signals are conclusive. Use the minimal generic checklist below and flag the classification as low-confidence in the plan's Decisions section, so the Stage 2 checkpoint gives the human a chance to correct it. |

## Step 2 — Apply the matching checklist

For each task-worthy change in a classified repo, check it against that repo's list below. A concern that applies and isn't addressed by an existing task becomes a new task (or a line item on an existing one); a concern that applies but is deliberately not addressed is recorded as such (see `plan-coverage-check`'s Non-Functional Coverage), not silently dropped.

### api-service

- **Idempotency** — does a retried request (client retry, load-balancer retry, at-least-once delivery upstream) produce the same result without duplicating side effects? Needed wherever the endpoint has a side effect (write, charge, send).
- **Failure tolerance & retries** — what happens when a downstream dependency times out or errors; is a retry safe (see idempotency) and bounded (backoff, max attempts)?
- **DB consistency** — does the change need a transaction boundary, and does it hold under concurrent writes (race conditions, lost updates)? Does a schema migration need to be backward-compatible during rollout?
- **Latency** — does the change add a call on the request's critical path, and does it have a budget/timeout?
- **Concurrency** — can this endpoint be called concurrently for the same resource, and is that safe (locking, optimistic concurrency, per-key serialization)?
- **Circuit breaking** — does a new outbound call to a dependency need a circuit breaker/bulkhead so that dependency's failure doesn't cascade?
- **Observability** — metrics (latency, error rate, saturation) and structured logs for the new/changed path; anything worth an alert?
- **Caching** — does a read path benefit from caching, and what invalidates it?
- **Distributed-systems edge cases** — partial failure (one downstream call succeeds, another fails — what state does that leave?), backpressure under load.
- **Required test-case categories (beyond happy/edge/error):** an idempotency case (same request sent twice, side effect happens once) for any task with a side effect; a case for the dependency-timeout/failure path if the task adds an outbound call; a concurrent-request case if the endpoint can be called concurrently for the same resource.

### worker / consumer

- **Consumer idempotency & dedup** — most queues/streams are at-least-once; can this consumer safely process the same message twice (dedup key, idempotent write)?
- **Retry policy** — on a transient processing failure, is the message retried, and with what backoff/limit?
- **Dead-letter / skip handling** — after retries are exhausted, does the message go to a DLQ, get skipped-and-logged, or something else? Is that pathway wired up, not just assumed?
- **Ordering guarantees** — does correctness depend on message order (per key or globally), and does the consumer's concurrency model preserve the order it needs?
- **Backpressure / concurrency limits** — is there a bound on how many messages are processed in parallel, so a burst doesn't overwhelm a downstream dependency?
- **Checkpointing / offset commit semantics** — is the offset/ack committed before or after processing completes, and what does that imply if the worker crashes mid-message?
- **Observability** — lag/backlog metrics, processing error rate, DLQ depth.
- **Required test-case categories (beyond happy/edge/error):** a duplicate-message case (dedup key seen twice, side effect happens once); a retry-exhausted case (message fails until the retry limit, then goes to the DLQ/skip path, not silently dropped); an out-of-order case if ordering matters.

### frontend-web / mobile

- **State management & data flow** — where does the new state live, and how does it flow to the components/screens that need it?
- **Offline / error / loading UX** — what does the user see while data is loading, if a request fails, or if the device is offline?
- **Accessibility** — screen-reader labels, focus order, contrast, tap-target size, for anything new/changed in the UI.
- **Platform conventions** — does the change follow the app's existing navigation pattern, design-system components, and state-management library, rather than introducing a new one?
- **Client-side caching / staleness** — if data is cached on-device, what invalidates or refreshes it?
- **Analytics / crash reporting** — does the new flow need an event or does an existing crash-reporting integration need to see it?
- **Testing** — platform-appropriate: React Testing Library/Jest for web, XCTest for iOS, Espresso/Robolectric for Android, widget tests for Flutter.
- **Required test-case categories (beyond happy/edge/error):** a loading-state case (what renders while the request is in flight); an error-state case (what renders on failure, and whether the user can retry); an offline case if the flow can be reached without connectivity.
- **Explicitly not applicable here:** DB consistency, circuit breakers, server-side concurrency control, consumer dedup/DLQ. Do not add these as tasks for a frontend-web or mobile repo — they belong to the API/worker the client talks to, not the client itself. If the RFC implies a backend concern surfacing in the client (e.g. "handle a 429 with backoff"), frame it as client-side error-handling UX, not as a server concern duplicated on the client.

### batch / cron / job

- **Idempotent re-run safety** — if the job is re-run (manually after a failure, or by an overlapping schedule), does it produce the same end state rather than double-applying?
- **Partial-failure / resume behavior** — if the job fails partway through, can it resume, or does it need to restart clean? Is that stated?
- **Runtime budget / timeout** — does the job have an expected duration and a timeout/kill-switch if it runs away?
- **Observability / alerting on failure** — is a failed or overrunning run visible without someone checking manually?
- **Required test-case categories (beyond happy/edge/error):** a re-run case (job run twice against the same input, end state is identical, nothing double-applied); a partial-failure case if the job processes multiple items (fails on item N, what state do items before/after N end up in).

### library / sdk

- **Backward compatibility / versioning** — does the change break existing consumers' call sites? Does it need a major version bump or a deprecation window?
- **Public API surface** — is the change to an exported/public symbol, or internal-only? Public changes need the same contract-level care an RFC would give an API endpoint.
- **Consumer migration notes** — if consumers must change how they call this library, is that migration documented for them?
- **Required test-case categories (beyond happy/edge/error):** a backward-compatibility case for any change to a public symbol (an existing call site's old usage still behaves the same, or the break is deliberate and documented).

### unknown / other

Minimal generic checklist: correctness (does the change do what the RFC says), tests (does it have any), logging (can a failure be seen). Flag the classification itself as low-confidence in Decisions, and ask the user to confirm or correct it at the Stage 2 checkpoint rather than silently guessing further.

## Multi-service plans

Classify every service in the resolved service list independently — a plan can span an API service, a worker, and a mobile client in the same run, and each keeps only its own checklist. A cross-service concern (e.g. "the API returns a 429 and the mobile client must back off") produces one task per side, each governed by that side's own playbook — the API task covers rate-limiting/circuit-breaking, the mobile task covers the backoff/error UX, and the two reference each other as a cross-service dependency (see `writing-plans`' Attribution step).
