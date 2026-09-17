# Backend architecture refactor for a microservices project

## Current state

The repository is already structured as a monorepo with separate application boundaries:

- `apps/web` for the Next.js frontend
- `apps/agent` for the Python FastAPI service
- `apps/backend` as a minimal TypeScript/Bun service placeholder
- `packages/db` for the Prisma/PostgreSQL integration

At the moment, `apps/backend/index.ts` only prints a hello message, so there is no existing API, business logic, or shared service layer to preserve. This is a good time to define the backend service boundaries before feature work grows.

## Proposed network layout

The backend should follow a gateway-first pattern where the public entrypoint is a small routing layer and internal services are separated by domain.

```text
               [ Public Internet ]
                       │
                       ▼
             ┌──────────────────┐
             │   api-gateway    │ (Routes traffic based on URLs)
             └───────┬──────────┘
                     │
     ┌───────────────┼───────────────┐  [ Internal Private Network ]
     ▼               ▼               ▼
┌─────────┐    ┌───────────┐   ┌───────────┐
│  agent  │    │   auth-   │   │  payment- │
│ (Python)│    │  service  │   │  service  │
└─────────┘    └───────────┘   └───────────┘
```

The gateway sits at the edge, receives requests from the browser or mobile client, and forwards them to the appropriate service based on URL or feature domain. Internal services are not exposed directly to the public internet.

## Monorepo target structure

```text
my-monorepo/
├── apps/
│   ├── web/                     # Frontend UI
│   ├── api-gateway/             # Only routes requests; minimal business logic
│   ├── agent/                   # Python AI service
│   ├── auth-service/            # User signups, tokens, permissions
│   ├── payment-service/         # Stripe/PayPal integrations
│   ├── notification-worker/     # Background email / push flows
│   └── ...
│
├── packages/
│   ├── database/                # Shared Prisma / SQLAlchemy schemas or models
│   ├── logger/                  # Reusable logging configuration
│   └── ...
```

## Target architecture

The backend should be split into independently deployable services, each owning a specific domain and its own data model.

```text
apps/
  api-gateway/
    src/
      server.ts
      routes/
      middleware/
      clients/

  auth-service/
    src/
      server.ts
      modules/
        auth/
    prisma/
      schema.prisma
      migrations/

  payment-service/
    src/
      server.ts
      modules/
        payments/
    prisma/
      schema.prisma
      migrations/

  notification-worker/
    src/
      worker.ts
      handlers/
      jobs/

  agent/
    main.py

packages/
  database/
    prisma/
    models/
    migrations/
  logger/
    src/
      logger.ts
  contracts/
    src/
      auth.ts
      payment.ts
      agent.ts
```

## Service responsibilities

### 1. API gateway

`apps/api-gateway` becomes the public entrypoint for the backend.

Responsibilities:

- HTTP/public API exposure
- Authentication and authorization checks
- Request validation and schema enforcement
- Routing to internal services based on URL namespace or domain
- Rate limiting, CORS, and error normalization
- Health and readiness endpoints

This service should not own heavy business logic. It is a facade and routing layer, not a domain service.

### 2. Auth service

`apps/auth-service` owns authentication flows such as:

- login
- registration
- session management
- token issuance and refresh
- permission checks
- password/security related workflows

This service owns its database schema and migrations for auth data.

### 3. Payment service

`apps/payment-service` owns payment-specific flows such as:

- Stripe or PayPal integration
- checkout sessions
- invoices and refunds
- subscription lifecycle events
- payment status synchronization

This service should isolate the external payment provider and all payment-state logic behind its own domain contract.

### 4. Notification worker

`apps/notification-worker` is a background worker dedicated to asynchronous communication such as:

- email delivery
- push notifications
- SMS workflows
- queued operational events from other services

This component usually does not accept public HTTP traffic and instead consumes event or queue-driven messages.

### 5. Agent service

The existing Python service in `apps/agent` remains a specialized service for agent execution and AI-related automation.

It should communicate with the rest of the backend through a clearly defined contract rather than direct database access from the gateway or other services.

## Database ownership

The current shared Prisma package under `packages/db` should not become a single global database layer used by every service.

Instead, each service should own its own database schema and migration history. For example:

```text
apps/auth-service/prisma/schema.prisma
apps/payment-service/prisma/schema.prisma
apps/notification-worker/prisma/schema.prisma
```

This keeps service boundaries clean:

- one service owns one database schema
- schema changes stay local to the service
- migrations are isolated to the service that owns the data
- the database layer is no longer a hidden shared dependency among every service

A shared package such as `packages/database` can still exist for reusable schema helpers, connection utilities, or generic SQL abstractions, but it should not become a central place where unrelated services store or mutate each other’s domain data.

## Shared packages

Only share stable, low-level cross-cutting concerns that are not tied to a specific business domain.

Suggested shared packages:

- `packages/contracts` for API/event DTO schemas
- `packages/config` for environment parsing and config validation
- `packages/logger` for centralized logging and correlation IDs
- `packages/database` for shared model conventions or DB tooling

Avoid sharing:

- domain repositories
- service business logic
- database models across services
- mutable application state

## Communication model

The gateway should be the main access point for user-facing requests. Internal services communicate using a simple, explicit contract.

Recommended default pattern:

- Gateway -> Auth service: REST or gRPC for identity requests
- Gateway -> Payment service: REST for checkout and status queries
- Payment service -> Notification worker: queued event messages for receipts and updates
- Agent service -> Gateway or other services: event-based or HTTP contract, depending on the workflow

In the early version, HTTP is the simplest and most maintainable option. Add message queues such as Kafka, RabbitMQ, or Redis Streams only when background/event-driven orchestration becomes necessary.

## Suggested migration path

1. Rename `apps/backend` to `apps/api-gateway`.
2. Add a minimal health endpoint and basic routing structure.
3. Create the first internal service, such as `auth-service`.
4. Add service-owned Prisma schema and migrations for auth data.
5. Add typed contracts between the gateway and auth service.
6. Add `payment-service` with isolated payment logic and external provider integration.
7. Add `notification-worker` for background async operations.
8. Keep service-to-service communication simple first: HTTP or REST, then queue-based events only when necessary.

## Design principles

- One service, one responsibility.
- Each service owns its data model and migrations.
- The gateway is a composition layer, not a domain layer.
- Shared libraries should contain cross-cutting contracts and tooling, not business logic.
- Introduce asynchronous events only when there is a real workflow need.

## Recommended first implementation

The first milestone should not be a full microservices platform immediately. The safest starting version is:

- `api-gateway`
- `auth-service`
- `payment-service`
- `notification-worker`
- existing `agent`

This creates clear service boundaries while preserving the current monorepo structure and allowing the app to evolve without a single backend monolith.

## Summary

The repo is not yet a microservices backend; it is a monorepo with a few service shells. The right refactor is to convert the current placeholder backend into a public API gateway and split internal responsibilities across domain-specific services such as auth, payment, notifications, and the Python agent, while keeping each service isolated and responsible for its own data and deployment concerns.
