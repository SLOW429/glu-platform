# ADR-005: Meilisearch over Elasticsearch

**Status:** Accepted  
**Date:** 2025-07-23  
**Deciders:** Architecture Team

---

## Context

GLU users search for books, authors, and categories constantly. The search experience is a core product differentiator — slow, inaccurate, or clunky search causes churn. We need a search engine that delivers:

- Typo-tolerant results (users type "harr potr" and find Harry Potter)
- Sub-100ms responses at all times
- Faceted filtering (by category, language, rating, price)
- Easy integration with the ingestion pipeline
- Manageable operational complexity for a startup team

---

## Decision

**Use Meilisearch as the search engine.**

---

## Alternatives Considered

| Solution | Description |
|---|---|
| **Elasticsearch** | Industry-standard, powerful, complex, Java-based |
| **OpenSearch** | AWS fork of Elasticsearch, equivalent capability |
| **Typesense** | Modern alternative to Meilisearch, also Rust-based |
| **Meilisearch** | Open-source, Rust, instant search, developer-friendly |
| **PostgreSQL FTS** | Built-in, no typo tolerance, no relevance tuning |
| **Algolia** | SaaS, best-in-class UX, expensive at scale |

---

## Why Not Elasticsearch

Elasticsearch is the industry standard for production search at scale. However, for GLU's current phase, it has significant disadvantages:

1. **Operational complexity**: Elasticsearch is a distributed system. Tuning shards, replicas, mappings, and JVM heap is a specialty skill. A misconfigured ES cluster causes split-brain scenarios, data loss, and OOM crashes. It requires dedicated operational expertise.

2. **Java overhead**: ES runs on the JVM. Minimum usable memory: 2GB heap. In a development environment with PostgreSQL, Redis, Qdrant, and BullMQ already running, ES adds crushing memory pressure.

3. **Slow relevance tuning**: ES relevance requires deep expertise in BM25 tuning, field boosting, and custom scoring scripts. Meilisearch has sensible defaults that work out of the box for most use cases.

4. **No built-in typo tolerance at this simplicity**: ES has fuzzy queries, but enabling useful typo tolerance requires careful configuration. Meilisearch applies typo tolerance automatically based on word length (1 typo for 5+ char words, 2 typos for 9+).

5. **Licensing change**: Elastic changed their license in 2021 (no longer Apache 2.0 for ES 7.11+). OpenSearch is the Apache 2.0 fork but inherits the operational complexity.

---

## Reasons for Meilisearch

### 1. Instant search, out of the box
Meilisearch is designed for instant search (results as you type). p95 query time is consistently under 50ms even on modest hardware. Elasticsearch can achieve this, but requires significant tuning and infrastructure investment.

### 2. Typo tolerance is the default behavior
No configuration required. "Harrry Potter", "harrypotter", "Potter Harry" all find the right book. This is a product expectation for users in 2024.

### 3. Excellent developer experience
Setup: run one Docker container, set a master key, call the API. Indexing documents: POST an array of JSON objects. That's it. The documentation is clear, the API is RESTful and intuitive. Time from zero to working search: 30 minutes.

### 4. Rust-based performance
Written in Rust. Low memory footprint (~200MB for GLU's dataset size), no GC pauses, fast startup. Runs comfortably alongside all other services in development.

### 5. Ranking rules are understandable
Meilisearch ranking rules are a simple ordered list: `[words, typo, proximity, attribute, sort, exactness]`. Customizing relevance is straightforward. Elasticsearch scoring requires understanding Lucene internals.

### 6. Tenant tokens for multi-tenancy (future)
Meilisearch supports tenant tokens — scoped API keys that filter results without exposing the master key. Future multi-tenant features are built-in.

### 7. Typesense comparison
Typesense is technically comparable to Meilisearch. Meilisearch was chosen because:
- Larger community and GitHub presence
- Better documentation
- More NestJS/Node.js examples in the ecosystem
- Slightly better typo tolerance algorithm in benchmarks

---

## Pros

- Sub-50ms search response times
- Typo tolerance without configuration
- Simple API and data model
- Low operational overhead
- Rust performance with minimal memory
- Open-source (MIT license)
- Built-in faceted search and filtering
- Good Node.js SDK (`meilisearch` npm package)

## Cons

- Not suitable for log analytics or time-series data (that is ES's domain)
- Less flexible scoring customization than ES
- Single-node by default (high availability requires Meilisearch Cloud or custom setup)
- Not appropriate if GLU later needs full Elasticsearch-style aggregations

---

## Search Architecture with Meilisearch

```
Write path:
  Domain event (book.published) → search-index queue → BookSearchProcessor
  → meilisearch.index('books').addDocuments([...])

Read path:
  SearchController → SearchService
  → cache lookup (Redis, 5min TTL)
  → cache miss → meilisearch.index('books').search(query, { filter, sort })
  → transform → cache → return

Hybrid search (planned Phase 2):
  Meilisearch (keyword) + Qdrant (semantic) → merge and re-rank results
  → best of both: typo-tolerant keyword + meaning-aware semantic
```

---

## Consequences

- Meilisearch runs as a Docker service (port 7700)
- `SearchModule` wraps all Meilisearch access behind `ISearchProvider`
- Index definitions (searchable fields, filterable fields, ranking rules) are code-managed in `src/search/indexes/`
- On every relevant domain event, the search processor syncs the updated document
- Full re-index available via `POST /admin/search/reindex` (admin-only)
- If Meilisearch is unavailable, search degrades to PostgreSQL LIKE queries (slower but functional)
