# ADR-0001: AdonisJS v6 over NestJS

- **Status**: Accepted
- **Date**: 2024-XX-XX
- **Deciders**: Michael DAŞ

## Context

Hallego required a TypeScript backend framework capable of serving
three distinct surfaces from a single codebase:

1. A REST API consumed by the Flutter mobile app
2. A server-rendered public website (Vue 3 + Inertia.js)
3. An internal admin panel (PrimeVue 4)

The framework needed to provide first-class support for: dependency
injection, ORM (with PostgreSQL + PostGIS), validation, queues, auth,
file storage, mail, real-time, rate limiting — without forcing us to
assemble these from disconnected libraries.

The team is one developer. Time-to-feature matters more than ultimate
flexibility. We wanted an opinionated full-stack framework, not a
collection of unrelated middlewares.

## Decision

We use **AdonisJS v6** as the unified backend framework for API,
web, and admin surfaces.

The framework provides a built-in IoC container, Lucid ORM, VineJS
validation, native auth with multi-driver support, Bull-Queue
integration, Inertia.js adapter for SSR, Drive for file storage,
Mail, and Limiter — all configured through a single `adonisrc.ts`
and accessible via the `node ace` CLI.

## Alternatives Considered

### Option A — NestJS
Mature TypeScript framework with strong DI, modular architecture,
and a large ecosystem.

**Rejected because:**
- Batteries are not included — NestJS is a structure, not a stack.
  We would still need to choose and integrate an ORM (TypeORM /
  Prisma), validation library, queue library, mail, auth strategy,
  file storage, real-time, etc. Each integration is glue code we
  would have to write and maintain.
- Decorator-heavy verbosity at the controller and module level
  compared to AdonisJS's lighter syntax.
- Inertia SSR integration is third-party and less polished than
  AdonisJS's official adapter.

### Option B — Express / Fastify with custom architecture
Maximum flexibility, hand-rolled stack.

**Rejected because:**
- One developer cannot afford to spend the first month wiring auth,
  validation, ORM, jobs, and storage into a coherent skeleton before
  writing any business logic.
- The lack of conventions creates bikeshedding with future
  contributors and inconsistency across modules.

### Option C — Next.js API routes / full-stack
Single Next.js app handling both frontend and backend.

**Rejected because:**
- Long-running tasks (image processing, queue consumers, scheduled
  cron jobs) do not fit the serverless-leaning Next.js model.
- The mobile app needs a stable REST API, not Next.js route handlers.
- Real-time SSE topology is awkward inside Next.js's request model.

### Option D — Django / Laravel / Rails
Mature, batteries-included frameworks in Python / PHP / Ruby.

**Rejected because:**
- The Flutter mobile app already uses Dart. Maintaining a
  TypeScript backend keeps the mental model coherent (one strongly
  typed language for backend, the same language family for the web
  frontend with Vue + TS).
- The personal preference and existing skill set of the developer
  favored TypeScript.

## Consequences

### Positive
- **Coherent stack out of the box**: ORM, validation, auth, queues,
  mail, file storage, rate limiting, real-time, all integrated and
  documented together.
- **`node ace` CLI provides scaffolding** for controllers, models,
  migrations, jobs, etc. Reduces boilerplate.
- **Inertia.js + Vue 3** allows server-rendered pages with full Vue
  reactivity, no separate SPA. Single deployment artifact.
- **DI container with `@inject()`** enables clean Action /
  Repository patterns without manual wiring.
- **Single TypeScript codebase** for API, web, admin — no language
  context switch when working across surfaces.

### Negative
- **Smaller community than NestJS or Express**: fewer third-party
  packages, fewer Stack Overflow answers. We accept reading the
  framework source when needed.
- **AdonisJS v6 is a major rewrite from v5**: third-party
  ecosystem is still catching up. Some packages (e.g. queue
  drivers) had to be selected carefully for v6 compatibility.
- **Opinionated structure**: deviating from the "Adonis way"
  fights the framework. We embrace the conventions rather than
  resist them.

### Neutral
- Hosting options are standard Node.js — no special platform
  requirements. Runs fine on Hetzner Docker Compose.

## References

- [AdonisJS documentation](https://docs.adonisjs.com)
- `backend/adonisrc.ts` — framework configuration
- `backend/package.json` — installed adapters
