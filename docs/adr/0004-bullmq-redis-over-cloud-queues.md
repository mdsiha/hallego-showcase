# ADR-0004: BullMQ + Redis over cloud queues

- **Status**: Accepted
- **Date**: 2024 (decision), recorded 2026-05-08
- **Deciders**: Michael DAŞ

## Context

Several Hallego flows must run asynchronously:

- **Image processing pipeline**, Sharp resize to 3 variants, WebP
  conversion, upload to S3-compatible object storage (Hetzner). Triggered on every ad
  submission and avatar change.
- **Email sending**, transactional emails (verification, password
  reset, ad approval/rejection, unread message reminder).
- **Push notifications**, FCM dispatch for chat messages and ad
  status changes.
- **Scheduled tasks**, daily GeoIP refresh, account anonymization
  for KVKK, expired session cleanup.

We needed: delayed jobs (e.g. unread-message reminder after 15
minutes), retries with backoff, scheduled (cron) jobs, and at-least-
once delivery semantics. Redis was already part of the stack for
session storage and was a natural fit.

## Decision

We use **`@rlanz/bull-queue`** (an AdonisJS adapter for BullMQ) on
a local Redis instance, deployed as a separate container in the same
Docker Compose stack as the application.

The same Docker image is used for both the API container and the
worker container, switched by `CONTAINER_ROLE=worker` in `start.sh`.
Workers consume from the queue with concurrency configured per
queue.

## Alternatives Considered

### Option A: AWS SQS / Google Cloud Tasks / Azure Service Bus
Managed cloud queue services with very high reliability and zero
operational overhead.

**Rejected because:**
- Adds a cloud dependency outside our Hetzner perimeter. Egress to
  a different cloud provider is non-zero cost and adds latency.
- Cron-style scheduling and delayed jobs require bolting on
  EventBridge / Cloud Scheduler, multiple services to coordinate.
- Vendor lock-in on a non-critical infrastructure component, when a
  local solution is sufficient at our scale.
- Pricing per million messages adds up faster than expected for a
  chat-heavy app.

### Option B: RabbitMQ
Mature open-source message broker, AMQP-based.

**Rejected because:**
- A second stateful service to operate (alongside PostgreSQL and
  Redis), purely for queue use cases.
- Delayed messages and scheduled jobs require plugins or workarounds.
- Redis is already in the stack for sessions and rate limiting,
  so reusing it has zero marginal infrastructure cost.

### Option C: PostgreSQL-based queue (e.g. `pgq`, `graphile-worker`)
Use the existing database as the queue.

**Rejected because:**
- Mixes transactional load with queue polling on the same database.
  At scale, queue polling becomes noisy on the primary.
- Less mature ecosystem than BullMQ in the Node.js world.
- We already use PostgreSQL for rate limiting (a separate ADR, not published here); adding
  queues on top would concentrate too much non-data load on the
  database.

### Option D: In-memory / `setTimeout`
No queue, run async work directly in the request handler or via
timers.

**Rejected because:**
- Fire-and-forget loses jobs on crash or restart.
- Delays are not durable across deploys.
- No retry semantics.
- Acceptable only for trivial fire-and-forget cases (which we keep
  inline, e.g. badge recompute after a message).

## Consequences

### Positive
- **Zero new infrastructure**: Redis is already deployed for
  sessions and rate-limiter cache.
- **Rich feature set**: delayed jobs, retries with backoff, cron-
  scheduled jobs, priority queues, all built into BullMQ.
- **Same image, different role**: the worker container reuses the
  app image with a different command. No separate build pipeline.
- **Inspectable**: BullMQ job state lives in Redis, so it can be inspected
  with standard tooling.
- **Atomic with Redis transactions**: BullMQ uses Lua scripts for
  reliable enqueue/dequeue.

### Negative
- **Single point of failure**: if Redis goes down, queues stop. We
  mitigate by treating Redis with the same monitoring and alerting
  as PostgreSQL, it is a critical dependency, not a cache.
- **Persistence trade-off**: Depending on the Redis persistence settings, a hard crash
  between fsync intervals can lose recent jobs. Acceptable for our
  workload (email, push, image processing) since callers can retry
  for the few cases where loss matters.
- **No multi-region story**: BullMQ + local Redis is single-region.
  Migrating to a managed queue would be required for any
  geographic redundancy.
- **Worker scaling is manual**: adding more workers means adding
  more containers in Docker Compose. No autoscaling. Acceptable at
  current scale.

### Neutral
- The choice of `@rlanz/bull-queue` (rather than calling BullMQ
  directly) ties us to that adapter's release cadence. Acceptable
  trade-off for AdonisJS integration sugar.

## References

Paths below refer to the private repository.

- [BullMQ documentation](https://docs.bullmq.io)
- [`@rlanz/bull-queue`](https://github.com/Rlanz/bull-queue)
- `backend/start.sh`, role-based startup
- `backend/app/*/jobs/`, job implementations
