# ADR-002: Prisma over TypeORM

**Status:** Accepted  
**Date:** 2025-07-23  
**Deciders:** Architecture Team

---

## Context

GLU needs an ORM for PostgreSQL. The ORM is a critical infrastructure choice — it affects developer experience, migration safety, query performance, and the reliability of the schema. A poor ORM choice causes production bugs, painful migrations, and slow feature development.

---

## Decision

**Use Prisma as the ORM.**

---

## Alternatives Considered

| ORM | Description |
|---|---|
| **TypeORM** | ActiveRecord + DataMapper, decorator-heavy, most popular NestJS ORM historically |
| **Prisma** | Schema-first, type-safe client, migration engine, Rust-powered query engine |
| **Drizzle** | SQL-first, very lightweight, excellent TypeScript inference |
| **MikroORM** | Identity Map pattern, TypeScript-native, underrated |
| **Knex.js** | Query builder (not ORM), maximum SQL control, no type safety |
| **Sequelize** | Mature, JavaScript-origin, TypeScript support retrofitted |

---

## Reasons for Prisma

### 1. Schema as the single source of truth
`schema.prisma` is the canonical definition of the database. All types, all relationships, all indexes are declared there. No decorators scattered across entity files. No synchronization bugs between code and DB. The schema file is human-readable and reviewable in PRs.

### 2. Generated type-safe client with zero runtime overhead
Prisma generates a typed client from the schema. Every query is fully typed — `findUnique`, `findMany`, `create`, `update` all have exact TypeScript types that reflect the actual columns. Mistyping a column name is a TypeScript compile error, not a runtime exception.

### 3. Migration engine is reliable and safe
`prisma migrate dev` generates SQL migration files. The migrations are plain SQL — reviewable, diffable, version-controlled. TypeORM's `synchronize: true` (used in most TypeORM projects) modifies the production database automatically, which is dangerous. Prisma's migration approach is safe by default.

### 4. No TypeORM N+1 problem
TypeORM's lazy loading relations cause N+1 queries silently. Prisma requires explicit `include` and `select` on every query, making data access visible in the code. There are no hidden queries.

### 5. Prisma Studio (developer tooling)
A built-in database browser (`npx prisma studio`) that runs locally. Developers can inspect and edit data without writing SQL or opening a DB client.

### 6. Performance
Prisma's query engine is written in Rust. It generates optimally parameterized queries and batches N+1 patterns automatically via data loader. Benchmarks show Prisma competitive with or faster than TypeORM in most workloads.

### 7. Multi-database ready
`schema.prisma` supports PostgreSQL, MySQL, SQLite, MongoDB, CockroachDB, PlanetScale. Switching databases (e.g., for multi-tenant sharding) is a schema provider change, not a rewrite.

---

## Pros

- Single source of truth (`schema.prisma`)
- Best-in-class TypeScript types (no `any`, no casting)
- Safe migration workflow (SQL files, never auto-sync in prod)
- Explicit data fetching (no hidden N+1 queries)
- Outstanding developer experience and tooling
- Active development by Prisma team + large community
- Excellent NestJS integration (`nestjs-prisma` module)
- Raw SQL escape hatch (`prisma.$queryRaw`) for complex queries

## Cons

- Schema-first means schema.prisma can get long for large schemas (mitigated by Prisma multi-file schema in v5.15+)
- No ActiveRecord pattern (not a con for this architecture, but familiar to Rails developers)
- Complex aggregations sometimes need raw SQL
- Bundle size larger than Drizzle (Rust engine via binary)
- Less flexible than Knex for very unconventional SQL patterns

---

## Consequences

- `prisma/schema.prisma` is the source of truth for all database structure
- `PrismaService` is a singleton provider available only to Repository classes
- No `@Entity()` decorators in domain models — domain entities are plain TypeScript classes mapped from Prisma records by Mappers
- All schema changes go through `prisma migrate dev` (never `synchronize: true`)
- Migration files are committed and reviewed in PRs
- `prisma db seed` handles all test/dev data seeding
