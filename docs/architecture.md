# DevHub Architecture

## Decision

DevHub is built as a **modular monolith** with **hexagonal (ports & adapters) layering** inside each module, **light DDD** for boundaries, and **OOP + SOLID** for class design.

We deliberately do **not** start with microservices.

| Dimension | Choice |
|-----------|--------|
| Deployment | Single application (modular monolith) |
| Module structure | Hexagonal / clean layers per module |
| Cross-module integration | In-process application/domain events + ports |
| Design style | OOP, SOLID, use-case driven |
| Async work | Queue port (adapter: Redis/Bull or Postgres outbox later) |

## Goals

- One process to develop, test, and deploy for MVP
- Clear business boundaries that can become services later
- Domain logic independent of Nest/Express, Prisma, Redis, HTTP
- Testable use cases via interfaces (ports)
- Reliable path for webhooks and jobs with retries

## High-level view

```text
                    ┌──────────────────────────────────────┐
                    │           DevHub monolith             │
 Client apps ──────►│  presentation (HTTP / future WS)     │
                    │                 │                     │
                    │                 ▼                     │
                    │  application (use cases)              │
                    │                 │                     │
                    │                 ▼                     │
                    │  domain (entities, rules, ports)      │
                    │                 ▲                     │
                    │                 │                     │
                    │  infrastructure (DB, queue, HTTP)    │
                    │                                       │
                    │  modules: projects | events |         │
                    │           webhooks | jobs |           │
                    │           monitoring | activity       │
                    └──────────────────────────────────────┘
```

## Architecture style (combined)

These are complementary, not alternatives:

1. **Modular monolith** — how the system is packaged and bounded
2. **Hexagonal / clean layers** — how each module is structured
3. **Light DDD** — how modules and entities are named and owned
4. **OOP + SOLID** — how classes and dependencies are designed

## Modules

Each module is a vertical business boundary. Modules must not import each other's infrastructure or repositories.

| Module | Responsibility |
|--------|----------------|
| `projects` | Projects, environments, API keys |
| `events` | Event ingestion and storage |
| `webhooks` | Endpoint registration, delivery, retries |
| `jobs` | Background job scheduling, execution, status |
| `monitoring` | Usage metrics (volume, latency, errors) |
| `activity` | Real-time activity feed for the console |
| `identity` | Users/auth for the dashboard (optional early) |

### Cross-module rules

- Depend on another module only through a **public application facade** or an **event/port**
- Prefer publishing domain/application events (for example `EventRecorded`) over direct writes into another module's tables
- Shared kernel stays thin: IDs, clock, errors, event bus interfaces — not business logic

## Layers (inside each module)

Dependency direction: **inward only**.

```text
presentation  →  application  →  domain
                                   ↑
                            infrastructure
```

| Layer | Contains | May depend on |
|-------|----------|---------------|
| **Domain** | Entities, value objects, domain services, domain events, ports (interfaces) | Nothing outside domain/shared domain |
| **Application** | Use cases, application DTOs, orchestration | Domain + ports |
| **Infrastructure** | Prisma/DB repos, queue adapters, HTTP clients, crypto | Domain/application ports |
| **Presentation** | Controllers, HTTP request/response DTOs, validation | Application use cases only |

### Folder layout

```text
src/
├── modules/
│   ├── projects/
│   │   ├── domain/
│   │   ├── application/
│   │   ├── infrastructure/
│   │   └── presentation/
│   ├── events/
│   ├── webhooks/
│   ├── jobs/
│   ├── monitoring/
│   └── activity/
├── shared/
│   ├── domain/              # Entity, DomainEvent, Result, errors
│   ├── application/         # IEventBus, IClock, IIdGenerator, IUnitOfWork
│   └── infrastructure/      # DB, Redis, config, composition root wiring
└── main.ts
```

## OOP model

Prefer rich domain objects with invariants over anemic data bags.

### Core entities

| Entity | Invariants / behavior |
|--------|------------------------|
| `Project` | Owns environments; create/list environments |
| `Environment` | Scoped to a project; owns API keys |
| `ApiKey` | Stores hash + prefix only; raw secret shown once at creation |
| `Event` | Immutable after create; validates type and payload |
| `WebhookEndpoint` | URL, secret, event-type filters, enabled flag |
| `WebhookDelivery` | State machine: pending → success \| failed → retrying → dead |
| `Job` / `JobRun` | Queued → running → succeeded \| failed, with retries |

### Value objects (examples)

`EventType`, `ApiKeyPrefix`, `Url`, `RetryPolicy`

## SOLID in practice

| Principle | How we apply it |
|-----------|-----------------|
| **S** | One use case class per action (`IngestEventUseCase`) |
| **O** | New delivery/queue strategies via new adapters, not domain edits |
| **L** | Port implementations honor the contract (no surprise side effects) |
| **I** | Small ports (`IEventRepository`, `IWebhookDispatcher`, `IClock`) |
| **D** | Use cases depend on interfaces; infrastructure implements them |

## Request lifecycle

Example: ingest an event and fan out side effects.

```text
POST /v1/events
  → EventsController                         (presentation)
  → IngestEventUseCase                       (application)
  → Event.create(...)                        (domain)
  → IEventRepository.save(...)               (port)
      → PrismaEventRepository                (infrastructure)
  → IEventBus.publish(EventRecorded)
  → WebhookOnEventHandler                    (webhooks/application)
  → IDeliveryQueue.enqueue(...)              (port)
  → MonitoringOnEventHandler                 (monitoring/application)
  → ActivityOnEventHandler                   (activity/application)
```

API-key auth resolves `projectId` + `environmentId` before the use case runs.

## Async processing

- HTTP handlers stay thin and fast: persist + publish + enqueue
- Webhook delivery and job execution run via a **queue port**
- MVP may run the consumer in the same process
- Later: same codebase, separate `api` and `worker` entrypoints if needed

## Data ownership

| Module | Owns tables (conceptual) |
|--------|---------------------------|
| `projects` | `projects`, `environments`, `api_keys` |
| `events` | `events` |
| `webhooks` | `webhook_endpoints`, `webhook_deliveries` |
| `jobs` | `jobs`, `job_runs` |
| `monitoring` | aggregates / metric snapshots |
| `activity` | optional activity log, or derived from events |

No module writes directly to another module's tables.

## Explicit non-goals (for now)

- Microservices / service mesh
- Full event sourcing
- Heavy CQRS read-model split on day one
- Framework logic inside the domain layer

These can be introduced later **behind existing ports** without rewriting module boundaries.

## Implementation order

1. Shared kernel (`Entity`, `Result`, `DomainEvent`, `IEventBus`)
2. `projects` module — environments + API keys
3. `events` module — ingest + list
4. Event bus + `webhooks` delivery/retry
5. `monitoring` basic aggregates
6. `jobs` module
7. `activity` realtime stream + console

## Definition of done for architecture compliance

A change is architecture-compliant when:

- Business rules live in domain or use cases, not controllers
- Domain has no imports from framework, ORM, or HTTP libraries
- Cross-module calls go through ports, facades, or the event bus
- New persistence/queue tech is added as an adapter, not by leaking into domain
