# ADR-004: Qdrant over pgvector

**Status:** Accepted  
**Date:** 2025-07-23  
**Deciders:** Architecture Team

---

## Context

GLU's Knowledge Engine embeds all content (books, articles, videos, podcasts) into high-dimensional vectors for semantic search, RAG (retrieval-augmented generation), and personalized recommendations. We need a vector store capable of:

- Storing millions of embedding vectors (1536 dimensions for OpenAI `text-embedding-3-small`)
- Sub-100ms approximate nearest neighbor (ANN) search
- Metadata filtering (e.g., search only within a specific book, language, or content type)
- Scaling independently of the relational database

The initial proposal used pgvector (PostgreSQL extension). After evaluation, this was replaced with a dedicated vector database.

---

## Decision

**Use Qdrant as the dedicated vector database.**

---

## Alternatives Considered

| Solution | Description |
|---|---|
| **pgvector** | PostgreSQL extension, vector columns in relational tables |
| **Qdrant** | Purpose-built vector database, Rust, ANN search, filtering |
| **Pinecone** | Managed vector database, excellent performance, proprietary |
| **Weaviate** | Open-source vector database, GraphQL API, ML integrations |
| **Chroma** | Lightweight open-source vector store, Python-first |
| **Milvus** | High-scale vector database, complex deployment |

---

## Why Not pgvector

pgvector adds a `vector` column type and `ivfflat`/`hnsw` indexes to PostgreSQL. This sounds convenient — one fewer service. However:

1. **Performance ceiling**: pgvector's ANN search runs inside PostgreSQL's query executor. At millions of vectors, it cannot match a dedicated ANN engine. Real benchmarks show Qdrant outperforming pgvector by 5-10x at the 1M+ vector scale.

2. **Resource contention**: Running vector search inside PostgreSQL means memory pressure from vector indexes competes with the buffer pool for relational queries. A slow vector search degrades all other DB queries.

3. **No payload filtering at ANN time**: pgvector applies `WHERE` clauses after the ANN search (post-filter), meaning it retrieves k nearest neighbors and then filters — returning fewer than k results. Qdrant applies filters during the ANN traversal (pre-filter), maintaining recall guarantees.

4. **Operational coupling**: The vector index grows large. Backed up, maintained, and scaled together with relational data — even though they have completely different access patterns.

5. **No collection-level isolation**: pgvector stores vectors as table rows. Isolating different content types (books, articles, user memory) requires separate tables with different index configurations. Qdrant has first-class collection support.

---

## Reasons for Qdrant

### 1. Purpose-built for vector search
Qdrant was designed exclusively for this problem. Its HNSW (Hierarchical Navigable Small World) index implementation is written in Rust for maximum performance and memory efficiency.

### 2. Filtered vector search at index time
Qdrant applies payload filters *during* ANN traversal, not after. Searching "books similar to this query, but only from this author, in English" returns exactly k results with the filter applied. This is critical for GLU's per-book and per-user scoped search.

### 3. Scalar quantization and binary quantization
Qdrant supports quantizing vectors (float32 → int8 or 1-bit) to reduce memory 4-32x with minimal recall loss. At millions of vectors, this is the difference between a $20/month server and a $200/month server.

### 4. gRPC + REST APIs
Qdrant provides both REST (easy to debug) and gRPC (high-throughput production) APIs. The Node.js SDK (`@qdrant/js-client-rest`) is maintained by the Qdrant team.

### 5. Self-hosted, open-source
Qdrant is Apache 2.0 licensed. Runs in Docker. Zero licensing cost. Can be hosted on any cloud. A managed Qdrant Cloud option exists if operational complexity becomes a concern.

### 6. Horizontal scaling
Qdrant supports distributed mode (sharding + replication) for high-availability and horizontal scaling. This is the natural evolution path as GLU's vector store grows.

### 7. Separation of concerns
Separating vector and relational data allows:
- Independent scaling (more CPU/RAM for Qdrant without touching PostgreSQL)
- Independent backups and recovery strategies
- Independent schema evolution

---

## Vector Store Interface (Abstraction)

The `IVectorStore` interface means Qdrant is not hardcoded into business logic:

```typescript
interface IVectorStore {
  upsert(collection: string, points: VectorPoint[]): Promise<void>;
  search(collection: string, query: SearchQuery): Promise<VectorResult[]>;
  delete(collection: string, ids: string[]): Promise<void>;
}
```

Migrating to Weaviate, Pinecone, or a future solution = implement `IVectorStore`. Zero business code changes.

---

## Pros

- Best-in-class ANN performance (Rust, HNSW)
- Pre-filter ANN search with payload metadata
- Quantization for memory efficiency
- Self-hosted, open-source, no vendor lock-in
- Collection-level isolation
- gRPC support for high-throughput production
- Horizontal scaling built-in

## Cons

- Additional service to deploy and monitor (mitigated by Docker Compose + health checks)
- Smaller community than pgvector (but active development, good documentation)
- Not PostgreSQL-integrated (separate backup strategy needed)

---

## Consequences

- Qdrant runs as a separate Docker service (port 6333 HTTP, 6334 gRPC)
- `VectorModule` wraps all Qdrant access behind `IVectorStore`
- No NestJS module ever imports the Qdrant SDK directly
- Two collections defined: `knowledge` (all content chunks) and `user-memory` (per-user preferences)
- Qdrant snapshots backed up daily to R2
- If Qdrant is unavailable, AI features degrade gracefully (RAG disabled, standard search still works)
