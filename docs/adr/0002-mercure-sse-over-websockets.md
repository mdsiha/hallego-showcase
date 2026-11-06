# ADR-0002: Mercure SSE over WebSockets

- **Status**: Accepted
- **Date**: 2024 (decision), recorded 2026-05-08
- **Deciders**: Michael DAŞ

## Context

Hallego requires real-time delivery for several user-facing features:

- New chat messages must appear instantly without polling
- A user's online/offline status must propagate to viewers of their ads
- Conversation deletion must be reflected on both sides immediately
- A global online-user count is displayed in the admin panel

The mobile Flutter app is the primary client (long-lived connections,
backgrounded frequently, must auto-reconnect on app resume). The web
admin panel is a secondary consumer.

We needed a transport that:
1. Survives mobile network changes and app backgrounding
2. Supports per-user authorization (a user must only receive their own
   events)
3. Is simple to operate alongside our existing Traefik + Docker Compose
   stack on a single Hetzner VPS
4. Does not require a separate broker process to maintain

## Decision

We use **Mercure** (https://mercure.rocks) over **Server-Sent Events
(SSE)** as the real-time transport for all push-from-server events.

Mercure runs as a sidecar container behind Traefik. The AdonisJS backend
publishes updates to the Mercure hub via HTTP POST. Clients (Flutter
mobile, web admin) subscribe to private topics authorized by JWT tokens
issued by the backend. JWT topics are derived from the user ID at
authentication time and refreshed via `/api/mercure/token`.

## Alternatives Considered

### Option A: WebSockets via Socket.io
A classic choice with mature client libraries on both Flutter
(`socket_io_client`) and JavaScript. Handles reconnection and rooms
out of the box.

**Rejected because:**
- Requires running a stateful Node.js process colocated with the API,
  or a separate Socket.io server. Either way, scaling horizontally
  requires sticky sessions or a Redis adapter, additional operational
  cost.
- Bidirectional channel is overkill: we only ever push from server to
  client. Client-to-server messages already go through normal HTTP
  endpoints (POST `/api/chat/:id/send-message`).
- Authorization model (rooms, namespaces) is application-level and must
  be implemented manually on every event type.

### Option B: AdonisJS Transmit (SSE)
AdonisJS ships an official SSE-based real-time package
(`@adonisjs/transmit`). Tight integration with the framework.

**Rejected because:**
- At evaluation time, Transmit's authorization story for per-user
  topics required custom middleware, comparable in effort to
  Mercure's JWT model.
- Tying real-time to the AdonisJS process means real-time goes down
  whenever the app process is restarted (deploys, OOMs). Mercure as
  a separate container survives app restarts.
- Less battle-tested in production than Mercure (which has been
  used at scale by Symfony/API Platform projects since 2018).

### Option C: Pusher / Ably / managed cloud service
Fully managed real-time infrastructure with SDKs for every platform.

**Rejected because:**
- Pricing scales with concurrent connections, costly for a chat-heavy
  consumer app with mostly idle but persistent connections.
- Adds a third-party dependency outside our infrastructure perimeter,
  which conflicts with our KVKK posture (Turkish data protection law).
- Vendor lock-in on a critical real-time path.

### Option D: Long polling
Simplest possible model, no extra infrastructure.

**Rejected because:**
- High request overhead for a chat product where users keep the
  conversation screen open for minutes at a time.
- Battery drain on mobile is significantly worse than a single
  persistent SSE connection.
- Latency is bounded by the polling interval, visibly worse UX for chat.

## Consequences

### Positive
- **One transport for all real-time use cases**: chat, online status,
  conversation events, admin counts. No need to mix SSE and WebSockets.
- **JWT-based authorization is declarative**: the backend signs a token
  listing the topics a user can subscribe to. Mercure enforces it. No
  per-event authorization logic in application code.
- **Reconnection and Last-Event-ID are built into the SSE protocol**:
  the client library handles reconnect with backoff and replay
  automatically.
- **Clean operational separation**: Mercure runs as its own container.
  Restarting the AdonisJS app does not interrupt active SSE connections.
- **Native Traefik integration**: TLS termination and routing already
  handled, no extra config.

### Negative
- **One-way only**: the client cannot push events to the server over the
  same connection. We rely on regular HTTP POST for client-to-server
  actions. This is fine for our use cases but blocks any future
  bidirectional protocol (e.g. WebRTC signaling).
- **Mercure is less mainstream than Socket.io**: fewer Stack Overflow
  answers, smaller community. We accept that we may need to read source
  code occasionally.
- **JWT token lifecycle adds complexity**: the mobile app must refresh
  the Mercure token on expiry and reconnect. Implemented in
  `MercureBloc` and `AppResumeService`.
- **Server-side fan-out is the publisher's responsibility**: when we
  send a chat message we explicitly call `mercure.send()` to both
  participants. There is no automatic room broadcast.

### Neutral
- The `mercure_client` Flutter package is third-party and maintained
  by the community, not by Mercure's authors. Acceptable for now,
  monitor for issues.

## References

Paths below refer to the private repository.

- [Mercure protocol specification](https://mercure.rocks/spec)
- `backend/app/mercure/`, token issuance
- `mobile/lib/mercure/`, client implementation
- SECURITY.md § Authentication
