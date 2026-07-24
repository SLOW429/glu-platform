# ADR-001: NestJS over Express

**Status:** Accepted  
**Date:** 2025-07-23  
**Deciders:** Architecture Team

---

## Context

GLU requires a Node.js HTTP framework for its REST API. The backend will grow to 20+ feature modules, require a rich middleware ecosystem, support dependency injection, and be worked on by multiple developers simultaneously. The framework choice shapes every pattern in the codebase.

The two primary candidates evaluated were **Express** (minimal, unopinionated) and **NestJS** (structured, opinionated, DI-native).

---

## Decision

**Use NestJS as the backend framework.**

---

## Alternatives Considered

| Framework | Description |
|---|---|
| **Express** | Minimal HTTP library, total freedom, massive ecosystem |
| **Fastify** | High-performance alternative to Express, schema-based validation |
| **Hono** | Ultra-lightweight, edge-first, newer ecosystem |
| **Koa** | Express successor, async-first, minimal |
| **NestJS** | Structured, Angular-inspired, DI container, TypeScript-first |

---

## Reasons for NestJS

### 1. Dependency Injection is built-in
Express has no DI. You wire dependencies manually or rely on third-party containers (tsyringe, InversifyJS). NestJS provides a production-grade IoC container natively. This is non-negotiable for a codebase of GLU's complexity — every service, repository, provider, and guard is injected, not imported and instantiated manually.

### 2. Module system enforces architecture
NestJS forces every feature into a module. Each module declares what it imports, exports, and provides. This prevents the "big ball of imports" that grows in Express codebases. At 20+ modules, this structure is the difference between a maintainable and an unmaintainable codebase.

### 3. TypeScript is a first-class citizen
NestJS was designed for TypeScript from day one. Decorators, interfaces, and metadata reflection are idiomatic. Express was retrofitted with `@types/express` — the experience is fundamentally different.

### 4. Built-in support for every GLU concern
NestJS provides first-party packages for:
- Guards (`@nestjs/passport`, `@nestjs/jwt`)
- Rate limiting (`@nestjs/throttler`)
- Configuration (`@nestjs/config`)
- Health checks (`@nestjs/terminus`)
- Caching (`@nestjs/cache-manager`)
- Queues (`@nestjs/bullmq`)
- Swagger (`@nestjs/swagger`)
- WebSockets (`@nestjs/websockets`)
- Microservices (`@nestjs/microservices`)

With Express, each of these requires finding, evaluating, and integrating third-party libraries with no consistency guarantee.

### 5. Swagger/OpenAPI generation is automatic
NestJS generates OpenAPI documentation from decorators on controllers. Express requires maintaining a separate YAML file or using fragile JSDoc. For an API-first product like GLU, self-documenting endpoints are essential for SDK generation and developer portal.

### 6. Testability
NestJS has a `Test.createTestingModule()` utility that mimics the module system in tests. Any module can be tested in complete isolation with mocked providers. Express apps require custom test harness setup per project.

### 7. Microservice migration path
When GLU grows beyond a monolith, `@nestjs/microservices` provides transport adapters (TCP, Redis, RabbitMQ, Kafka, gRPC) without rewriting business logic. Express services require a full rewrite or a custom message layer.

---

## Pros

- Enforced structure scales with team size
- Built-in DI removes an entire class of architectural bugs
- First-party packages for every common concern
- Excellent TypeScript experience
- Automatic Swagger documentation
- Lower bus factor: new engineers know the patterns immediately
- Clear migration to microservices
- Large community (used by Netflix, Adidas, Roche)

## Cons

- More opinionated — less flexibility for unconventional patterns
- Higher initial learning curve vs Express
- Decorator-heavy (requires `experimentalDecorators: true`)
- Slightly higher cold-start time than raw Express or Fastify
- Abstraction can obscure what is happening at the HTTP layer

---

## Consequences

- All modules follow the NestJS module/controller/service pattern
- Dependency injection is mandatory — no singleton factories
- HTTP layer details (req/res) are abstracted behind NestJS pipes/guards/interceptors
- The `@nestjs/swagger` decorator system drives API documentation
- Future SDK generation reads from the auto-generated OpenAPI spec
- New engineers are expected to read the NestJS documentation before contributing
