# ADR-007: Modular Monolith before Microservices

**Status:** Accepted  
**Date:** 2025-07-23  
**Deciders:** Architecture Team

---

## Context

GLU is being built from scratch. The team must decide the initial deployment topology: should the backend be a single deployable unit (monolith) or a collection of independently deployable services (microservices)?

Both options have been seriously evaluated. The decision has long-term implications for team velocity, operational complexity, and architecture correctness.

---

## Decision

**Build a Modular Monolith first. Design for, but do not implement, microservices boundaries.**

The system is structured as a single NestJS application with strict module boundaries, domain isolation, an event bus abstraction, and a repository pattern — all of which are prerequisites for microservice extraction. The extraction itself happens when the need is proven, not when it is anticipated.

---

## Alternatives Considered

| Topology | Description |
|---|---|
| **Big Ball of Mud Monolith** | Single app, no boundaries, everything imports everything |
| **Modular Monolith** | Single app, strict module boundaries, DI, event-driven |
| **Microservices from Day One** | Independent deployable services per domain |
| **Serverless Functions** | Per-function deployment, no persistent server |
| **Modular Monolith → Microservices** | Monolith now, extract services as boundaries prove correct |

---

## Why Not Microservices from Day One

Martin Fowler's "MonolithFirst" pattern applies here directly. The core argument:

> "Don't start with a microservices architecture. Start with a monolith, modularize well, and split only when it becomes necessary."

Specific reasons for GLU:

### 1. Distributed systems problems are hard
Microservices introduce: network latency between services, distributed transactions (saga pattern or 2PC), service discovery, circuit breakers, API versioning across services, distributed tracing, and cross-service authentication. These problems exist whether you have 100 users or 1 million users. At 100 users, they're pure overhead.

### 2. Domain boundaries are not yet known
The most expensive microservices mistake is drawing the service boundary in the wrong place. Once two services share a boundary incorrectly, you face distributed joins, cross-service transactions, and circular dependencies that are extremely costly to refactor. In a monolith, moving code between modules is a refactor. In microservices, it's a multi-service migration with API contract negotiations.

The correct microservice boundary is discovered by running the system and observing real access patterns, not by guessing at design time.

### 3. Operational complexity multiplies
Each microservice needs: its own Docker container, its own CI/CD pipeline, its own health checks, its own log aggregation, its own deploy configuration, its own secrets management. For 10 services, this is 10x the operational overhead. For an early-stage team, this overhead crowds out product work.

### 4. NestJS microservices support already exists
When GLU is ready to extract a service (e.g., the AI processing service under heavy load), `@nestjs/microservices` provides transport adapters. The extraction is module boundary → new NestJS app → wire via transport. The refactor is incremental, not a rewrite.

---

## Modular Monolith Requirements

A monolith without discipline becomes a big ball of mud, which is worse than either alternative. GLU's modular monolith enforces:

### Hard rules (enforced by code review + linting):
1. **No cross-module direct imports**: `BooksModule` never imports `UsersRepository` directly. It declares `UsersModule` as an import and uses the exported `UsersService`.
2. **Communication by events, not function calls**: When a book is published, `BookPublishedEvent` is emitted on the event bus. Any other module (notifications, analytics, search) subscribes to the event. `BooksService` does not call `NotificationsService` directly.
3. **No shared database tables**: Each module "owns" its tables. Other modules access that data through the owning module's service/repository, never by querying the table directly.
4. **Repository pattern everywhere**: No service writes raw Prisma queries. All DB access goes through a repository. This makes extracting a service easier: the repository becomes the service's data layer.
5. **Domain entities are pure**: Domain entities have no Prisma decorators, no framework dependencies. They're plain TypeScript. This makes them portable across module boundaries.

### Microservice extraction signals:
The following signals indicate a module should become a service:
- The module has a different scaling profile (e.g., AI processing needs GPU, REST API needs more CPU)
- The module has a different deployment cadence (independent release)
- The module has a different team ownership
- Performance profiling shows the module as a bottleneck that would benefit from independent scaling

---

## Migration Path (when ready)

```
Phase 1 (Now):        Modular Monolith
                      All modules in one NestJS app
                      BullMQ for async (already distributed)

Phase 2 (Scaling):    Extract AI Processing Service
                      Reason: GPU requirements, rate limit isolation
                      How: AiModule → new NestJS app + @nestjs/microservices
                      Transport: Redis pub/sub (already in stack)

Phase 3 (Enterprise): Extract more services as needed
                      Following the same pattern
```

---

## Pros

- Full team velocity on product features from day one
- No distributed systems overhead until scale demands it
- Service boundaries discovered from real usage, not guessed upfront
- Module structure makes future extraction incremental, not a rewrite
- Single deployment, single CI/CD pipeline, single observability target
- Debugging: all logs in one stream, no distributed tracing required

## Cons

- Cannot independently scale individual modules (mitigated: BullMQ workers can scale)
- All modules must use the same language/runtime (Node.js/NestJS)
- A catastrophic bug in one module can take down the entire application
- Module discipline requires enforcement — without code review, the monolith degrades

---

## Consequences

- Single NestJS application deployed as one Docker container
- All modules in `src/modules/` and `src/`
- Module boundaries enforced by code review + planned ESLint import rules
- Event-driven communication between modules (IEventBus)
- Repository pattern everywhere (no service → DB direct access)
- AI Processing Module is the first candidate for extraction in Phase 2
- Architecture team reviews module dependency graph quarterly
