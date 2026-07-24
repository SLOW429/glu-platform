# ADR-003: PostgreSQL as Primary Database

**Status:** Accepted  
**Date:** 2025-07-23  
**Deciders:** Architecture Team

---

## Context

GLU stores relational data: users, books, chapters, reviews, comments, library entries, payments, subscriptions, reading progress, and more. These are deeply relational — a book has chapters, chapters have highlights, highlights belong to users, users have subscriptions, etc. The database must handle complex queries, foreign keys, transactions, and JSON semi-structured data. It will eventually serve millions of records.

---

## Decision

**Use PostgreSQL 16 as the primary relational database.**

---

## Alternatives Considered

| Database | Description |
|---|---|
| **PostgreSQL** | Open-source, relational, ACID, JSON support, extensive extensions |
| **MySQL / MariaDB** | Widely used relational DB, solid performance |
| **MongoDB** | Document database, flexible schema, horizontal scaling |
| **PlanetScale** | MySQL-compatible serverless DB with branching |
| **CockroachDB** | Distributed SQL, PostgreSQL-compatible, global distribution |
| **SQLite** | Embedded, single-file, great for dev/testing |

---

## Reasons for PostgreSQL

### 1. ACID compliance with no compromises
PostgreSQL is fully ACID at every isolation level. Payment processing, subscription management, and inventory operations require true transactional semantics. MongoDB's multi-document transactions are a bolt-on; PostgreSQL transactions are foundational.

### 2. Relational model matches GLU's domain
GLU's data is inherently relational: books belong to authors, chapters belong to books, highlights belong to users and chapters, reviews are linked to books and users. A document database would require either embedding everything (denormalization nightmare) or using application-side joins (n+1 queries, consistency bugs).

### 3. JSONB for semi-structured data
When metadata is truly schema-less (AI-extracted metadata, feature flag conditions, notification preferences), PostgreSQL's `JSONB` column handles it — indexed, queryable with operators, no separate collection. The best of both relational and document worlds.

### 4. Full-text search as supplement
PostgreSQL has built-in full-text search with `tsvector`/`tsquery`. We use Meilisearch for user-facing search (better UX, typo tolerance), but PostgreSQL FTS is available for admin queries and fallback without additional infrastructure.

### 5. Rich extension ecosystem
- `pg_stat_statements` — query performance monitoring
- `pg_cron` — scheduled jobs in the DB
- `uuid-ossp` / `gen_random_uuid()` — native UUIDs
- `pgcrypto` — database-level encryption functions
- Connection pooling via PgBouncer (future)

### 6. Horizontal read scaling
PostgreSQL streaming replication creates read replicas. Prisma supports multiple datasource URLs, so read-heavy analytics queries can route to replicas automatically.

### 7. Point-in-Time Recovery (PITR)
WAL archiving enables recovery to any second in the past. For a platform with user-generated content and payment data, this is mandatory. MySQL and MongoDB have equivalent features but PostgreSQL's WAL implementation is the industry gold standard.

### 8. Open source and self-hostable
PostgreSQL is free, open-source, and runs identically in development (Docker), staging, and production (managed: AWS RDS, Supabase, Neon, or self-hosted). No vendor lock-in. No per-row pricing surprises.

### 9. Prisma support is first-class
Prisma's most mature and most tested adapter is PostgreSQL. Schema migrations, type generation, and advanced queries all work flawlessly.

---

## Pros

- True ACID transactions
- JSONB for hybrid relational/document needs
- Extensions for every advanced need
- Excellent Prisma integration
- Read replicas for horizontal scaling
- PITR for backup and recovery
- Zero licensing cost
- Massive operational knowledge in the industry

## Cons

- Vertical scaling limit (mitigated by read replicas and connection pooling)
- Schema migrations require planning (no schema-less flexibility)
- Connection management overhead (PgBouncer needed at high scale)
- More complex operational setup vs managed NoSQL services

---

## Consequences

- All relational data lives in PostgreSQL (via Prisma)
- Vector data lives in Qdrant (separate concern — see ADR-004)
- Full-text search powered by Meilisearch (see ADR-005)
- `PrismaService` wraps the connection pool
- Read replicas added in Phase 2 for analytics queries
- Migrations are SQL files committed to version control
- PgBouncer connection pooling planned for production > 50 concurrent connections
