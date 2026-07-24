# GLU — Enterprise Architecture v2.0

> Treat GLU as a company, not a software project.
> Every decision must prioritize: Scalability · Maintainability · Security · Developer Experience · Cost Efficiency · High Availability · Performance · Future Enterprise Support.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Technology Stack](#2-technology-stack)
3. [Architectural Pattern](#3-architectural-pattern)
4. [Complete Folder Structure](#4-complete-folder-structure)
5. [Knowledge Engine](#5-knowledge-engine)
6. [AI Platform](#6-ai-platform)
7. [Module Catalogue](#7-module-catalogue)
8. [Database Schema](#8-database-schema)
9. [Repository Pattern](#9-repository-pattern)
10. [DTOs & Validation](#10-dtos--validation)
11. [Authentication & RBAC](#11-authentication--rbac)
12. [Event Bus Abstraction](#12-event-bus-abstraction)
13. [Storage Abstraction](#13-storage-abstraction)
14. [Vector Database](#14-vector-database)
15. [Caching Strategy](#15-caching-strategy)
16. [Background Jobs](#16-background-jobs)
17. [Search Architecture](#17-search-architecture)
18. [Notification System](#18-notification-system)
19. [OCR & Import Pipeline](#19-ocr--import-pipeline)
20. [AI Cost Management](#20-ai-cost-management)
21. [Moderation System](#21-moderation-system)
22. [Observability](#22-observability)
23. [Security Architecture](#23-security-architecture)
24. [Feature Flags](#24-feature-flags)
25. [API Gateway](#25-api-gateway)
26. [CDN Architecture](#26-cdn-architecture)
27. [Backup & Recovery](#27-backup--recovery)
28. [Docker & Infrastructure](#28-docker--infrastructure)

---

## 1. Overview

GLU is an AI-powered knowledge ecosystem. The system is designed as a **Modular Monolith** today with clean microservice boundaries for tomorrow. Every module is independently testable, independently deployable in the future, and independently scalable.

### Core Principles

| Principle | Implementation |
|---|---|
| Single Responsibility | Every class, module, and service has exactly one reason to change |
| Open/Closed | Extend via new providers/agents/processors; never modify existing |
| Dependency Inversion | All cross-module deps are through interfaces, never concrete classes |
| Domain Isolation | No module imports another module's internals — only public APIs |
| Event-Driven | All side effects are domain events, never inline function calls |
| Repository Pattern | No service ever touches the ORM directly |
| Zero Hardcoding | No prompts, no provider names, no config values in business logic |

---

## 2. Technology Stack

### Core Runtime

| Component | Technology | Version |
|---|---|---|
| Runtime | Node.js | 22 LTS |
| Framework | NestJS | 11 |
| Language | TypeScript | 5.5+ (strict) |
| ORM | Prisma | 5 |
| Primary DB | PostgreSQL | 16 |

### Infrastructure

| Component | Technology | Notes |
|---|---|---|
| Cache / Pub-Sub | Redis | 7 — sessions, rate limiting, hot cache, event bus |
| Queue | BullMQ | Redis-backed; abstracted behind EventBus interface |
| Search | Meilisearch | Self-hosted; typo-tolerant, sub-100ms |
| Vector DB | Qdrant | Dedicated vector store; abstracted behind VectorStore interface |
| Storage | Cloudflare R2 / MinIO | S3-compatible; abstracted behind StorageProvider interface |
| Container | Docker + Docker Compose | Dev parity; production via Docker Swarm or Kubernetes |

### AI

| Component | Technology |
|---|---|
| Primary | OpenAI (GPT-4o) |
| Secondary | Anthropic Claude |
| Tertiary | Google Gemini |
| Budget | DeepSeek, Groq |
| Local | Ollama, LM Studio (via OpenAI-compatible API) |
| Router | OpenRouter (optional unified access) |

### Observability

| Component | Technology |
|---|---|
| Tracing | OpenTelemetry → Jaeger |
| Metrics | Prometheus → Grafana |
| Errors | Sentry |
| Logs | Pino → Grafana Loki |
| Health | NestJS Terminus |
| APM | OpenTelemetry SDK |

---

## 3. Architectural Pattern

```
┌─────────────────────────────────────────────────────────────────┐
│                        API Gateway Layer                        │
│         Auth · Rate Limit · Logging · Versioning · WAF          │
└─────────────────────────────────────────────────────────────────┘
                               │
┌─────────────────────────────────────────────────────────────────┐
│                     Presentation Layer                          │
│           Controllers · Guards · Pipes · Interceptors           │
└─────────────────────────────────────────────────────────────────┘
                               │
┌─────────────────────────────────────────────────────────────────┐
│                     Application Layer                           │
│           Services (Use Cases) · DTOs · Mappers · Events        │
└─────────────────────────────────────────────────────────────────┘
                               │
┌─────────────────────────────────────────────────────────────────┐
│                      Domain Layer                               │
│         Entities · Value Objects · Interfaces · Policies        │
└─────────────────────────────────────────────────────────────────┘
                               │
┌─────────────────────────────────────────────────────────────────┐
│                   Infrastructure Layer                          │
│   Repositories · Cache · EventBus · Storage · Search · AI       │
└─────────────────────────────────────────────────────────────────┘
```

**Cross-cutting concerns** (never import into domain or application layers):
- Observability (tracing, metrics, logging)
- Security (guards, middleware)
- Feature flags
- Audit logging

---

## 4. Complete Folder Structure

```
backend/
│
├── src/
│   │
│   ├── main.ts                          # Bootstrap: global pipes, interceptors, Swagger, OTEL
│   ├── app.module.ts                    # Root module
│   │
│   ├── config/                          # Typed, validated config per concern
│   │   ├── app.config.ts
│   │   ├── database.config.ts
│   │   ├── redis.config.ts
│   │   ├── jwt.config.ts
│   │   ├── storage.config.ts
│   │   ├── search.config.ts
│   │   ├── vector.config.ts
│   │   ├── ai.config.ts
│   │   ├── observability.config.ts
│   │   └── index.ts
│   │
│   ├── common/                          # Shared across ALL modules — no business logic
│   │   ├── decorators/
│   │   │   ├── current-user.decorator.ts
│   │   │   ├── public.decorator.ts
│   │   │   ├── roles.decorator.ts
│   │   │   ├── permissions.decorator.ts
│   │   │   ├── throttle.decorator.ts
│   │   │   └── cacheable.decorator.ts
│   │   ├── filters/
│   │   │   ├── global-exception.filter.ts
│   │   │   └── prisma-exception.filter.ts
│   │   ├── guards/
│   │   │   ├── jwt-auth.guard.ts
│   │   │   ├── permissions.guard.ts
│   │   │   ├── feature-flag.guard.ts
│   │   │   └── api-key.guard.ts
│   │   ├── interceptors/
│   │   │   ├── logging.interceptor.ts
│   │   │   ├── transform.interceptor.ts
│   │   │   ├── audit.interceptor.ts
│   │   │   └── tracing.interceptor.ts
│   │   ├── pipes/
│   │   │   └── zod-validation.pipe.ts
│   │   ├── pagination/
│   │   │   ├── pagination.dto.ts
│   │   │   └── paginated-result.entity.ts
│   │   └── utils/
│   │       ├── slug.util.ts
│   │       ├── hash.util.ts
│   │       ├── crypto.util.ts
│   │       └── time.util.ts
│   │
│   ├── database/                        # Prisma — the only place that knows about Prisma
│   │   ├── prisma.service.ts
│   │   ├── prisma.module.ts
│   │   └── prisma-exception.mapper.ts
│   │
│   ├── cache/                           # Redis wrapper — all cache access goes here
│   │   ├── cache.service.ts
│   │   ├── cache.module.ts
│   │   └── cache.constants.ts
│   │
│   ├── event-bus/                       # Abstract event bus — BullMQ today, Kafka tomorrow
│   │   ├── event-bus.interface.ts
│   │   ├── event-bus.module.ts
│   │   ├── event-bus.service.ts         # Delegates to active provider
│   │   ├── events/                      # All domain event definitions
│   │   │   ├── book.events.ts
│   │   │   ├── user.events.ts
│   │   │   ├── ai.events.ts
│   │   │   ├── payment.events.ts
│   │   │   └── ...
│   │   └── providers/
│   │       ├── bullmq.provider.ts
│   │       ├── redis-streams.provider.ts  # stub for future
│   │       └── in-memory.provider.ts      # for tests
│   │
│   ├── storage/                         # File storage — swap providers without business code change
│   │   ├── storage.interface.ts
│   │   ├── storage.module.ts
│   │   ├── storage.service.ts           # Delegates to active provider
│   │   └── providers/
│   │       ├── r2.provider.ts
│   │       ├── minio.provider.ts
│   │       ├── s3.provider.ts
│   │       ├── azure-blob.provider.ts
│   │       └── gcs.provider.ts
│   │
│   ├── search/                          # Meilisearch abstraction
│   │   ├── search.interface.ts
│   │   ├── search.module.ts
│   │   ├── search.service.ts
│   │   ├── indexes/
│   │   │   ├── books.index.ts
│   │   │   ├── authors.index.ts
│   │   │   └── categories.index.ts
│   │   └── providers/
│   │       └── meilisearch.provider.ts
│   │
│   ├── vector/                          # Qdrant — dedicated vector database
│   │   ├── vector.interface.ts
│   │   ├── vector.module.ts
│   │   ├── vector.service.ts
│   │   ├── collections/
│   │   │   ├── knowledge.collection.ts
│   │   │   └── user-memory.collection.ts
│   │   └── providers/
│   │       ├── qdrant.provider.ts
│   │       └── in-memory.provider.ts    # for tests
│   │
│   ├── feature-flags/                   # Runtime feature toggling
│   │   ├── feature-flags.module.ts
│   │   ├── feature-flags.service.ts
│   │   ├── feature-flags.guard.ts
│   │   └── flags/
│   │       └── flags.registry.ts        # All known flags with defaults
│   │
│   ├── rbac/                            # Role-Based Access Control
│   │   ├── rbac.module.ts
│   │   ├── rbac.service.ts
│   │   ├── permissions.registry.ts      # All permissions defined here
│   │   ├── policies/
│   │   │   ├── book.policy.ts
│   │   │   ├── chapter.policy.ts
│   │   │   └── admin.policy.ts
│   │   └── decorators/
│   │       └── require-permission.decorator.ts
│   │
│   ├── observability/                   # OpenTelemetry + metrics + health
│   │   ├── observability.module.ts
│   │   ├── tracing.service.ts
│   │   ├── metrics.service.ts
│   │   ├── health/
│   │   │   ├── health.controller.ts
│   │   │   ├── database.health.ts
│   │   │   ├── redis.health.ts
│   │   │   ├── meilisearch.health.ts
│   │   │   └── qdrant.health.ts
│   │   └── interceptors/
│   │       └── otel.interceptor.ts
│   │
│   ├── queue/                           # BullMQ queue definitions & processors
│   │   ├── queue.module.ts
│   │   ├── queue.constants.ts
│   │   └── processors/
│   │       ├── email.processor.ts
│   │       ├── notification.processor.ts
│   │       ├── ai-job.processor.ts
│   │       ├── image.processor.ts
│   │       ├── analytics.processor.ts
│   │       ├── search-index.processor.ts
│   │       ├── moderation.processor.ts
│   │       └── knowledge-ingest.processor.ts
│   │
│   ├── ai/                              # AI Platform — fully split into sub-modules
│   │   ├── ai.module.ts
│   │   │
│   │   ├── gateway/                     # Routes requests to providers
│   │   │   ├── ai-gateway.service.ts
│   │   │   └── ai-gateway.types.ts
│   │   │
│   │   ├── providers/                   # Interchangeable LLM providers
│   │   │   ├── ai-provider.interface.ts
│   │   │   ├── openai.provider.ts
│   │   │   ├── anthropic.provider.ts
│   │   │   ├── gemini.provider.ts
│   │   │   ├── deepseek.provider.ts
│   │   │   ├── groq.provider.ts
│   │   │   ├── openrouter.provider.ts
│   │   │   └── ollama.provider.ts       # OpenAI-compatible local
│   │   │
│   │   ├── prompts/                     # All prompts — never hardcoded in code
│   │   │   ├── prompt-loader.service.ts # Loads from /prompts directory
│   │   │   ├── prompt-renderer.ts       # Handlebars/Mustache template engine
│   │   │   └── prompt-registry.ts       # Maps agent → prompt file
│   │   │
│   │   ├── memory/                      # AI conversation memory
│   │   │   ├── memory.service.ts
│   │   │   ├── short-term.memory.ts     # Redis — current session
│   │   │   └── long-term.memory.ts      # Qdrant — persistent user knowledge
│   │   │
│   │   ├── agents/                      # Specialized AI agents
│   │   │   ├── agent.interface.ts
│   │   │   ├── reading.agent.ts
│   │   │   ├── writing.agent.ts
│   │   │   ├── research.agent.ts
│   │   │   ├── librarian.agent.ts
│   │   │   ├── teacher.agent.ts
│   │   │   ├── character.agent.ts
│   │   │   ├── analytics.agent.ts
│   │   │   ├── moderator.agent.ts
│   │   │   ├── marketplace.agent.ts
│   │   │   └── voice.agent.ts
│   │   │
│   │   ├── rag/                         # Retrieval-Augmented Generation
│   │   │   ├── rag.service.ts
│   │   │   ├── retriever.service.ts
│   │   │   └── reranker.service.ts
│   │   │
│   │   ├── embeddings/                  # Text → vector conversion
│   │   │   ├── embeddings.service.ts
│   │   │   └── embeddings.cache.ts      # Cache embeddings to save API costs
│   │   │
│   │   ├── tools/                       # Agent tool use (function calling)
│   │   │   ├── tool.interface.ts
│   │   │   ├── search.tool.ts
│   │   │   ├── calculator.tool.ts
│   │   │   └── web-search.tool.ts
│   │   │
│   │   ├── workflows/                   # Multi-step AI pipelines
│   │   │   ├── summary.workflow.ts
│   │   │   ├── quiz-generation.workflow.ts
│   │   │   └── book-analysis.workflow.ts
│   │   │
│   │   ├── moderation/                  # Content safety
│   │   │   ├── moderation.service.ts
│   │   │   └── moderation.types.ts
│   │   │
│   │   ├── cost/                        # AI cost tracking and control
│   │   │   ├── cost.service.ts
│   │   │   ├── cost.tracker.ts
│   │   │   ├── budget.enforcer.ts
│   │   │   └── cost.types.ts
│   │   │
│   │   └── evaluation/                  # Output quality scoring
│   │       ├── evaluation.service.ts
│   │       └── scorers/
│   │           ├── faithfulness.scorer.ts
│   │           └── relevance.scorer.ts
│   │
│   ├── knowledge-engine/                # The heart of GLU
│   │   ├── knowledge-engine.module.ts
│   │   ├── knowledge-engine.service.ts  # Orchestrates the full pipeline
│   │   ├── pipeline/
│   │   │   ├── pipeline.types.ts
│   │   │   ├── stages/
│   │   │   │   ├── normalize.stage.ts
│   │   │   │   ├── parse.stage.ts
│   │   │   │   ├── chunk.stage.ts
│   │   │   │   ├── metadata-extract.stage.ts
│   │   │   │   ├── embed.stage.ts
│   │   │   │   ├── vector-store.stage.ts
│   │   │   │   └── knowledge-graph.stage.ts
│   │   │   └── pipeline.orchestrator.ts
│   │   └── processors/                  # One per content type
│   │       ├── content-processor.interface.ts
│   │       ├── book.processor.ts
│   │       ├── pdf.processor.ts
│   │       ├── epub.processor.ts
│   │       ├── docx.processor.ts
│   │       ├── markdown.processor.ts
│   │       ├── html.processor.ts
│   │       ├── article.processor.ts
│   │       ├── video.processor.ts
│   │       ├── podcast.processor.ts
│   │       ├── image-ocr.processor.ts
│   │       └── web-page.processor.ts
│   │
│   ├── import/                          # Source-specific import connectors
│   │   ├── import.module.ts
│   │   ├── import.service.ts
│   │   └── connectors/
│   │       ├── connector.interface.ts
│   │       ├── web.connector.ts
│   │       ├── github.connector.ts
│   │       ├── google-docs.connector.ts
│   │       ├── notion.connector.ts
│   │       ├── youtube.connector.ts
│   │       └── podcast.connector.ts
│   │
│   └── modules/                         # Feature modules (business domains)
│       ├── auth/
│       ├── users/
│       ├── authors/
│       ├── books/
│       ├── chapters/
│       ├── categories/
│       ├── tags/
│       ├── reviews/
│       ├── comments/
│       ├── library/
│       ├── bookmarks/
│       ├── highlights/
│       ├── notes/
│       ├── collections/
│       ├── marketplace/
│       ├── payments/
│       ├── subscriptions/
│       ├── analytics/
│       ├── notifications/
│       ├── moderation/
│       ├── challenges/
│       └── admin/
│
├── prompts/                             # AI prompt files — versioned, never hardcoded
│   ├── reading/
│   │   ├── chat.v1.md
│   │   ├── summarize.v1.md
│   │   └── explain.v1.md
│   ├── writing/
│   │   ├── grammar.v1.md
│   │   ├── rewrite.v1.md
│   │   └── continue.v1.md
│   ├── teacher/
│   │   ├── quiz.v1.md
│   │   └── flashcards.v1.md
│   ├── research/
│   │   └── research.v1.md
│   ├── librarian/
│   │   └── recommend.v1.md
│   ├── moderation/
│   │   └── moderate.v1.md
│   └── translator/
│       └── translate.v1.md
│
├── prisma/
│   ├── schema.prisma
│   ├── migrations/
│   └── seed/
│       ├── seed.ts
│       ├── categories.seed.ts
│       ├── books.seed.ts
│       └── users.seed.ts
│
├── docker/
│   ├── api/
│   │   ├── Dockerfile
│   │   └── Dockerfile.dev
│   └── nginx/
│       └── nginx.conf
│
├── docs/
│   ├── architecture/
│   │   ├── ARCHITECTURE.md              # This file
│   │   └── adr/                         # Architecture Decision Records
│   └── api/
│
├── test/
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── docker-compose.yml                   # Full local dev stack
├── docker-compose.prod.yml
├── .env.example
└── package.json
```

### Every Feature Module — Standard Shape

```
modules/books/
├── books.module.ts                      # Imports, providers, exports
├── books.controller.ts                  # HTTP routes, Swagger, guards
├── books.service.ts                     # Use cases — orchestrates repos, events
├── books.repository.ts                  # Data access — Prisma only here
├── books.search.ts                      # Meilisearch sync and query logic
├── books.mapper.ts                      # Prisma record → domain entity → response DTO
├── domain/
│   ├── book.entity.ts                   # Rich domain object
│   └── book.policies.ts                 # Business rules (can user access? etc.)
├── dto/
│   ├── create-book.dto.ts
│   ├── update-book.dto.ts
│   ├── book-query.dto.ts
│   └── book-response.dto.ts
└── tests/
    ├── books.service.spec.ts
    ├── books.repository.spec.ts
    └── books.controller.spec.ts
```

---

## 5. Knowledge Engine

The Knowledge Engine is the central content processing system. Every content type — books, PDFs, articles, videos, podcasts, web pages, images — flows through an identical pipeline. Logic is never duplicated per content type.

### Pipeline

```
Content Source (any type)
        │
        ▼
┌───────────────────┐
│  Import / Ingest   │  ← connector fetches raw bytes + source metadata
└───────────────────┘
        │
        ▼
┌───────────────────┐
│     Normalize     │  ← standard ContentItem envelope: { id, type, source, raw, metadata }
└───────────────────┘
        │
        ▼
┌───────────────────┐
│      Parse        │  ← type-specific parser extracts text + structure
└───────────────────┘   (PDFProcessor, EPUBProcessor, VideoProcessor, OCRProcessor...)
        │
        ▼
┌───────────────────┐
│      Chunk        │  ← split into overlapping semantic chunks (512 tokens, 64 overlap)
└───────────────────┘
        │
        ▼
┌───────────────────┐
│ Metadata Extract  │  ← AI-assisted: language, topics, entities, difficulty, reading time
└───────────────────┘
        │
        ▼
┌───────────────────┐
│    Embeddings     │  ← each chunk → vector via embeddings service (cached)
└───────────────────┘
        │
        ▼
┌───────────────────┐
│  Vector Storage   │  ← store in Qdrant (collection: knowledge)
└───────────────────┘
        │
        ▼
┌───────────────────┐
│  Knowledge Graph  │  ← extract entities, link to graph (books ↔ authors ↔ topics ↔ events)
└───────────────────┘
        │
        ▼
     Ready for AI services (RAG, summarization, chat, recommendations...)
```

### Content Types Supported

| Type | Processor | Chunk Strategy |
|---|---|---|
| Book / Novel | BookProcessor | By chapter, then paragraph |
| PDF | PDFProcessor | By page, then paragraph |
| EPUB | EPUBProcessor | By chapter |
| Research Paper | PDFProcessor + ArxivMetadata | By section |
| Article / Blog | HTMLProcessor | By heading section |
| DOCX | DocxProcessor | By heading section |
| Markdown | MarkdownProcessor | By heading section |
| Web Page | WebPageProcessor | Main content extraction (Readability) |
| Video | VideoProcessor | Transcript → by segment |
| Podcast | PodcastProcessor | Transcript → by segment |
| Image | ImageOCRProcessor | Full page as single chunk |
| Notes | MarkdownProcessor | Whole note as single chunk |

### ContentProcessor Interface

```typescript
interface ContentProcessor {
  readonly contentType: ContentType;
  process(item: RawContentItem): Promise<ProcessedContent>;
}

interface ProcessedContent {
  text: string;
  structure: ContentStructure;   // chapters, sections, pages
  metadata: ContentMetadata;
  language: string;
}
```

---

## 6. AI Platform

The AI module is split into independent sub-modules. No agent, no service, no workflow imports a specific provider. Everything routes through the AI Gateway.

### AI Gateway

Routes all AI requests. Responsibilities:
- Select provider based on: task type, cost budget, latency requirement, availability
- Manage provider failover chain (primary → secondary → budget)
- Stream responses back to caller
- Log every request for cost tracking and evaluation

### Provider Interface

```typescript
interface AIProvider {
  readonly name: string;
  readonly models: ModelInfo[];

  chat(request: ChatRequest): Promise<ChatResponse>;
  stream(request: ChatRequest): AsyncGenerator<string>;
  embed(texts: string[]): Promise<number[][]>;
  moderate(text: string): Promise<ModerationResult>;
  isAvailable(): Promise<boolean>;
  getModelCost(model: string): ModelCost;
}
```

Providers implemented: OpenAI, Anthropic, Gemini, DeepSeek, Groq, OpenRouter, Ollama.
Switching providers = change one config value. Zero business code changes.

### Prompt Management

All prompts live in `/prompts/**/*.md`. **Zero prompts in TypeScript source code.**

```markdown
<!-- prompts/reading/chat.v1.md -->
---
version: 1
agent: reading
task: book-chat
model_preference: gpt-4o
temperature: 0.3
---

You are a precise reading assistant for the book "{{bookTitle}}" by {{authorName}}.

Answer questions ONLY using the provided context excerpts from the book.
If the answer is not in the context, say so clearly.
Always cite the chapter and approximate location of information.

## Context from the book:
{{#each chunks}}
[Chapter {{this.chapterNumber}}: {{this.chapterTitle}}]
{{this.text}}

{{/each}}

## Conversation history:
{{#each history}}
{{this.role}}: {{this.content}}
{{/each}}

## User question:
{{question}}
```

Prompt versioning: `chat.v1.md`, `chat.v2.md`. The registry maps agent + task + version.

### Agent Architecture

```
AIOrchestrator.dispatch(intent: AgentIntent) → Agent → AIGateway → Provider
```

Each agent:
- Has a single responsibility
- Loads its prompt from the registry
- Calls the gateway (never the provider directly)
- Returns a typed response

### RAG Pipeline

```
User Question
     │
     ▼
Embed question (EmbeddingsService)
     │
     ▼
Semantic search in Qdrant (VectorService, top-k=5, score threshold=0.72)
     │
     ▼
Rerank results (cross-encoder, optional)
     │
     ▼
Build context window (chunks + metadata)
     │
     ▼
Render prompt template (PromptLoader)
     │
     ▼
Stream to AI Gateway → Provider
     │
     ▼
Extract citations from response
     │
     ▼
Return { answer, citations, sessionId }
```

### Memory System

```
Short-Term Memory (Redis, TTL=2h)
  — current conversation messages
  — keyed by sessionId

Long-Term Memory (Qdrant, persistent)
  — user interests extracted from reading behavior
  — books the AI "knows" the user has read
  — personalization state for recommendations
```

### AI Cost Management

```
Budget Enforcer:
  — per-user daily token budget (configurable per tier: free/premium/enterprise)
  — per-agent cost limits
  — provider cost comparison before dispatch
  — automatic fallback to cheaper model when budget is near limit

Cost Tracking:
  — every AI call → AIUsage record (userId, agent, provider, model, inputTokens, outputTokens, costUSD)
  — aggregated daily/monthly per user and per agent
  — real-time budget check before dispatch

Failover Chain (example):
  Primary:  gpt-4o        (high quality, higher cost)
  Secondary: claude-3-haiku (high quality, lower cost)
  Budget:   deepseek-chat  (good quality, very low cost)
  Fallback: groq/llama-3   (fast, near-zero cost)
```

---

## 7. Module Catalogue

| Module | Responsibility | Key Dependencies |
|---|---|---|
| AuthModule | JWT, OAuth2, 2FA, sessions | UsersModule, CacheModule, EventBus |
| UsersModule | User profiles, stats, preferences | PrismaModule, CacheModule |
| AuthorsModule | Author profiles, earnings | UsersModule, BooksModule |
| BooksModule | Book CRUD, featured/trending | KnowledgeEngine, SearchModule, EventBus |
| ChaptersModule | Chapter content, progress tracking | BooksModule, KnowledgeEngine |
| CategoriesModule | Categories, tags | PrismaModule |
| ReviewsModule | Ratings, reviews | BooksModule, ModerationModule |
| CommentsModule | Comments, replies, threading | ModerationModule, NotificationsModule |
| LibraryModule | Reading lists, progress, sync | BooksModule, CacheModule |
| BookmarksModule | Bookmarks per chapter | BooksModule |
| HighlightsModule | Text highlights + AI categorization | BooksModule, AIModule |
| NotesModule | Personal notes, AI-generated notes | BooksModule, AIModule |
| CollectionsModule | User-curated book lists | BooksModule |
| SearchModule | Full-text + AI search | Meilisearch, AIModule |
| AIModule | All AI features | KnowledgeEngine, VectorModule |
| KnowledgeEngineModule | Content processing pipeline | VectorModule, AIModule |
| ImportModule | External content import | KnowledgeEngineModule |
| MarketplaceModule | Book sales, pricing, coupons | PaymentsModule |
| PaymentsModule | Stripe, PayPal, webhooks | EventBus |
| SubscriptionsModule | Premium plans, billing | PaymentsModule |
| NotificationsModule | In-app, email, push | EventBus, QueueModule |
| ModerationModule | Content safety, spam, copyright | AIModule |
| AnalyticsModule | Reading stats, author metrics | EventBus, CacheModule |
| ChallengesModule | Reading challenges, streaks, XP | UsersModule, EventBus |
| FeatureFlagsModule | Feature toggles, A/B testing | CacheModule |
| RBACModule | Roles, permissions, policies | PrismaModule |
| ObservabilityModule | OTEL, metrics, health, tracing | All modules |
| AdminModule | Admin operations, reports | All modules |

---

## 8. Database Schema (Prisma — Additions to v1)

```prisma
// ─── RBAC ──────────────────────────────────────────────────────────────────

model Role {
  id          String           @id @default(cuid())
  name        String           @unique  // reader | author | admin | enterprise_admin
  description String?
  permissions RolePermission[]
  users       UserRole[]
}

model Permission {
  id          String           @id @default(cuid())
  resource    String           // book | chapter | user | admin | ai
  action      String           // read | create | update | delete | publish | moderate
  description String?
  roles       RolePermission[]
  @@unique([resource, action])
}

model RolePermission {
  roleId       String
  permissionId String
  role         Role       @relation(fields: [roleId], references: [id])
  permission   Permission @relation(fields: [permissionId], references: [id])
  @@id([roleId, permissionId])
}

model UserRole {
  userId String
  roleId String
  user   User   @relation(fields: [userId], references: [id], onDelete: Cascade)
  role   Role   @relation(fields: [roleId], references: [id])
  @@id([userId, roleId])
}

// ─── FEATURE FLAGS ─────────────────────────────────────────────────────────

model FeatureFlag {
  id          String            @id @default(cuid())
  key         String            @unique  // "ai_chat", "marketplace", "beta_reader"
  name        String
  description String?
  enabled     Boolean           @default(false)
  rolloutPct  Int               @default(0)      // 0-100 percentage
  conditions  Json?                               // { roles: [], userIds: [], segments: [] }
  updatedAt   DateTime          @updatedAt
  createdAt   DateTime          @default(now())
}

// ─── KNOWLEDGE ENGINE ──────────────────────────────────────────────────────

model KnowledgeItem {
  id           String            @id @default(cuid())
  contentType  String            // book | pdf | article | video | podcast | web_page
  sourceId     String?           // e.g. bookId for books
  title        String
  language     String            @default("en")
  status       IngestionStatus   @default(PENDING)
  metadata     Json?
  chunkCount   Int               @default(0)
  vectorized   Boolean           @default(false)
  errorMessage String?
  createdAt    DateTime          @default(now())
  updatedAt    DateTime          @updatedAt
  @@index([contentType, sourceId])
  @@index([status])
}

enum IngestionStatus { PENDING PROCESSING COMPLETED FAILED }

// ─── AI COST TRACKING ──────────────────────────────────────────────────────

model AIBudget {
  id               String   @id @default(cuid())
  userId           String   @unique
  tier             String   @default("free")   // free | premium | enterprise
  dailyTokenLimit  Int      @default(10000)
  monthlyTokenLimit Int     @default(100000)
  dailyTokensUsed  Int      @default(0)
  monthlyTokensUsed Int     @default(0)
  resetAt          DateTime
  user             User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

// ─── MODERATION ────────────────────────────────────────────────────────────

model ModerationReport {
  id           String            @id @default(cuid())
  reporterId   String?
  contentType  String            // comment | review | book | user
  contentId    String
  reason       String
  aiScore      Float?            // 0-1 toxicity score
  aiCategories String[]          // [ "hate_speech", "spam" ]
  status       ModerationStatus  @default(PENDING)
  reviewedBy   String?
  reviewedAt   DateTime?
  action       String?           // removed | warned | dismissed
  createdAt    DateTime          @default(now())
  @@index([contentType, contentId])
  @@index([status])
}

enum ModerationStatus { PENDING REVIEWED ACTIONED DISMISSED }

// ─── IMPORT HISTORY ────────────────────────────────────────────────────────

model ImportJob {
  id           String      @id @default(cuid())
  userId       String
  source       String      // web | github | google_docs | notion | youtube | podcast
  sourceUrl    String?
  status       ImportStatus @default(PENDING)
  resultId     String?     // KnowledgeItem id once complete
  errorMessage String?
  createdAt    DateTime    @default(now())
  updatedAt    DateTime    @updatedAt
}

enum ImportStatus { PENDING RUNNING COMPLETED FAILED }

// ─── API KEYS (Public API) ─────────────────────────────────────────────────

model ApiKey {
  id          String   @id @default(cuid())
  userId      String
  name        String
  keyHash     String   @unique
  prefix      String   // first 8 chars shown in UI
  scopes      String[]  // read:books | write:books | ai:chat
  rateLimit   Int      @default(1000)    // requests per hour
  lastUsedAt  DateTime?
  expiresAt   DateTime?
  revokedAt   DateTime?
  createdAt   DateTime @default(now())
  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  @@index([userId])
}
```

---

## 9. Repository Pattern

```typescript
// Every repository implements this typed contract
interface IRepository<TEntity, TCreateDto, TUpdateDto, TQueryDto> {
  findById(id: string): Promise<TEntity | null>;
  findMany(query: TQueryDto): Promise<PaginatedResult<TEntity>>;
  create(dto: TCreateDto): Promise<TEntity>;
  update(id: string, dto: TUpdateDto): Promise<TEntity>;
  delete(id: string): Promise<void>;
  exists(id: string): Promise<boolean>;
  count(query: Partial<TQueryDto>): Promise<number>;
}

// Concrete example
class BooksRepository implements IBookRepository {
  constructor(private readonly prisma: PrismaService) {}
  // Only this file contains the word "prisma"
  // Services call this.booksRepository.findFeatured() — they never see Prisma
}
```

**Rules:**
- No service ever imports `PrismaService` directly
- No service ever writes a Prisma query
- Repositories return domain entities, never raw Prisma objects
- Repositories are the only classes that know table names and column names

---

## 10. DTOs & Validation

```typescript
// All DTOs use Zod schemas. The schema IS the DTO type.
export const CreateBookSchema = z.object({
  title:       z.string().min(1).max(200),
  subtitle:    z.string().max(300).optional(),
  description: z.string().max(5000).optional(),
  categoryId:  z.string().cuid(),
  language:    z.string().length(2).default('en'),
  isFree:      z.boolean().default(true),
  price:       z.number().positive().max(9999).optional(),
  tags:        z.array(z.string().min(1).max(50)).max(10).default([]),
}).refine(
  (data) => data.isFree || (data.price != null && data.price > 0),
  { message: 'Paid books must have a price', path: ['price'] }
);

export type CreateBookDto = z.infer<typeof CreateBookSchema>;

// ZodValidationPipe applies globally — all controller inputs are validated
// Response DTOs are explicit classes — never leak internal DB fields
// Query DTOs include pagination, sorting, filtering — all validated
```

---

## 11. Authentication & RBAC

### Auth Flow

```
Register:
  email + password → argon2 hash → create User record
  → send verification email (queue) → return { token, user }

Login:
  verify email + hash → create Session in Redis (key: session:{userId}:{deviceId})
  → issue JWT access token (15min, RS256, contains: sub, role, sessionId)
  → issue opaque refresh token (64-byte random, hashed in DB, 7 days)
  → return { accessToken, refreshToken, user }

Token Refresh:
  validate refresh token hash → check not revoked → rotate both tokens
  → old refresh token marked revoked → new pair returned

OAuth2:
  redirect → Google/GitHub/Apple callback
  → find or create User + OAuthAccount record
  → same session + token issuance as Login

2FA:
  TOTP (speakeasy/otplib) → QR code → verify → store secret encrypted
  → on login: after password → require 2FA code before issuing tokens

Session Revocation:
  logout → delete Redis session → mark refresh token revoked
  → immediate effect even before JWT expires (Redis lookup on every request)
```

### RBAC Model

```
User has Roles (many-to-many)
Role has Permissions (many-to-many)
Permission = resource + action  (e.g. "books:publish", "admin:users:delete")

Policies add contextual rules:
  BookPolicy.canUpdate(user, book) → user.id === book.authorId || user.hasPermission('admin:books:update')

Decorator usage:
  @RequirePermission('books:publish')          // declarative on controllers
  @UseGuards(PermissionsGuard)
  publishBook(@Param('id') id: string) {}
```

---

## 12. Event Bus Abstraction

Business logic emits typed domain events. The event bus implementation (BullMQ today) is completely hidden.

```typescript
// event-bus.interface.ts
interface IEventBus {
  publish<T>(event: DomainEvent<T>): Promise<void>;
  subscribe<T>(eventName: string, handler: EventHandler<T>): void;
}

// Usage in service (no BullMQ import):
await this.eventBus.publish(new BookPublishedEvent({ bookId, authorId, title }));

// BullMQ provider implements IEventBus
// Swapping to Kafka = write KafkaProvider, change config. Zero service changes.

// Test provider = in-memory array. No Docker required for unit tests.
```

---

## 13. Storage Abstraction

```typescript
interface IStorageProvider {
  upload(key: string, file: Buffer, options: UploadOptions): Promise<StorageFile>;
  download(key: string): Promise<Buffer>;
  delete(key: string): Promise<void>;
  getSignedUrl(key: string, expiresIn: number): Promise<string>;
  getPublicUrl(key: string): string;
  exists(key: string): Promise<boolean>;
  list(prefix: string): Promise<StorageFile[]>;
}

// StorageService delegates to the configured provider
// R2Provider, MinIOProvider, S3Provider, AzureBlobProvider, GCSProvider all implement IStorageProvider
// Switching = change STORAGE_PROVIDER env var. Zero business code changes.
// CDN: every getPublicUrl() returns the CDN URL, not the storage URL directly.
```

---

## 14. Vector Database

```typescript
interface IVectorStore {
  upsert(collection: string, points: VectorPoint[]): Promise<void>;
  search(collection: string, query: SearchQuery): Promise<VectorResult[]>;
  delete(collection: string, ids: string[]): Promise<void>;
  createCollection(name: string, config: CollectionConfig): Promise<void>;
  deleteCollection(name: string): Promise<void>;
}

// Collections:
//   knowledge       — all content chunks (filtered by contentType, sourceId)
//   user-memory     — long-term user preference vectors

// QdrantProvider implements IVectorStore
// InMemoryProvider for tests
// Switching to Weaviate = new provider, same interface
```

---

## 15. Caching Strategy

```
L1 — In-process: NestJS CacheModule (node-cache), 30s TTL, bounded 500 entries
L2 — Redis: all durable cache, configurable TTL per key

Key hierarchy:
  books:featured                     60min
  books:trending                     30min
  books:new-releases                 30min
  books:{id}                         10min
  books:{id}:chapters                30min
  authors:{id}                       15min
  categories:all                     4h
  users:{id}:profile                 5min
  users:{id}:stats                   5min
  ai:summary:{bookId}:{opts-hash}    24h
  ai:embeddings:{text-hash}          7days
  search:{query-hash}                5min
  home:stats                         5min
  flags:{key}                        1min

Invalidation:
  — Write-through: on every mutation, invalidate affected keys
  — Tag-based groups: tag(bookId) → flush all keys tagged with that book
  — TTL-based: all keys have expiry — no stale forever

Cache decorator (declarative):
  @Cacheable({ key: 'books:featured', ttl: 3600 })
  @CacheBust({ tags: ['books', 'books:{id}'] })
```

---

## 16. Background Jobs

```
Queue: email              concurrency=5, retry=3, backoff=exponential
  send-welcome-email
  send-verification-email
  send-password-reset
  send-chapter-notification
  send-weekly-digest

Queue: notification       concurrency=10, retry=3
  push-notification (FCM/APNs)
  in-app-notification
  batch-digest-notification

Queue: knowledge-ingest   concurrency=2, retry=2
  ingest-book
  ingest-pdf
  ingest-article
  ingest-web-page
  ingest-video-transcript

Queue: ai-generation      concurrency=2, retry=2, rate-limited
  generate-ai-summary
  generate-quiz
  generate-flashcards
  categorize-highlights
  generate-recommendations

Queue: search-index       concurrency=10, retry=5
  index-book
  update-book
  remove-book
  index-author

Queue: image-processing   concurrency=3, retry=2
  resize-cover (400x600, 200x300, 100x150)
  optimize-image
  extract-dominant-color

Queue: analytics          concurrency=20, retry=1 (best-effort)
  record-reading-session
  update-book-stats
  compute-trending
  update-author-stats

Queue: moderation         concurrency=5, retry=2
  moderate-comment
  moderate-review
  moderate-book-content
  scan-for-copyright

All queues:
  — Dead letter queue for jobs that exhaust retries
  — BullBoard dashboard at /admin/queues (admin-only)
  — Alerting when queue depth > threshold
```

---

## 17. Search Architecture

```
Meilisearch Indexes:

books:
  searchable:  title, subtitle, description, authorName, tags[], categoryName
  filterable:  categoryId, language, isFree, isPremium, isPublished, averageRating, createdAt, tags[]
  sortable:    averageRating, viewCount, likeCount, downloadCount, createdAt
  distinct:    id
  rankingRules: [words, typo, proximity, attribute, sort, exactness, averageRating:desc]
  typoTolerance: enabled (1 typo for 5+ char, 2 typos for 9+ char)

authors:
  searchable:  displayName, bio, genres[]
  filterable:  isVerified, followerCount, bookCount
  sortable:    followerCount, bookCount

categories:
  searchable:  name, description
  filterable:  bookCount
  sortable:    bookCount, name

Search service:
  1. Validate + sanitize query
  2. Hash query + filters → Redis cache lookup (5min TTL)
  3. Cache miss → call Meilisearch → transform results → cache → return
  4. AI semantic search: embed query → Qdrant similarity search → merge with Meilisearch results

Sync strategy:
  — Domain event (book.published, book.updated, book.deleted) → search-index queue
  — Processor calls SearchService.syncBook(book)
  — Full re-index available via admin endpoint
```

---

## 18. Notification System

```
Delivery channels:
  In-App   — stored in notifications table, pushed via SSE
  Email    — rendered templates → BullMQ email queue → Resend/SES
  Push     — FCM (Android + Web), APNs (iOS) → notification queue

SSE stream:
  GET /notifications/stream
  → Redis pub/sub (channel: notifications:{userId})
  → EventSource on frontend
  → New notification → publish to Redis channel → SSE pushes to client

Trigger map:
  user.followed       → NEW_FOLLOWER   → in-app + push
  comment.created     → NEW_COMMENT    → in-app + push + email (digest)
  comment.replied     → NEW_REPLY      → in-app + push
  book.liked          → NEW_LIKE       → in-app (batched, max 1/hr)
  chapter.published   → NEW_CHAPTER    → in-app + push + email
  achievement.earned  → ACHIEVEMENT    → in-app + push

User preferences:
  NotificationPreference table: { userId, channel, type, enabled }
  — Checked before every delivery
  — Digest mode: aggregate in-app → single daily email

Templates: Handlebars, stored in /src/modules/notifications/templates/
```

---

## 19. OCR & Import Pipeline

```
Source → Connector → RawContentItem → KnowledgeEngine

Connectors:
  WebConnector:       fetch URL, extract main content (Mozilla Readability)
  GitHubConnector:    repo README + markdown files via GitHub API
  GoogleDocsConnector: export to Docx, then DocxProcessor
  NotionConnector:    export via Notion API, then MarkdownProcessor
  YouTubeConnector:   fetch transcript via youtube-transcript-api
  PodcastConnector:   download audio → Whisper transcription → MarkdownProcessor

Processors (implement ContentProcessor):
  PDFProcessor:       pdf-parse + custom table/image extraction
  EPUBProcessor:      epub.js DOM extraction
  DocxProcessor:      mammoth (docx → html → plain text)
  MarkdownProcessor:  remark AST parser
  HTMLProcessor:      Mozilla Readability + JSDOM
  ImageOCRProcessor:  Tesseract.js (local) or Google Vision API

OCR pipeline:
  Image upload → detect image type → OCR processor
  → extracted text → KnowledgeEngine.ingest()
  → chunks → embeddings → Qdrant
```

---

## 20. AI Cost Management

```
Budget Tiers:
  Free:       10,000 tokens/day,   100,000 tokens/month
  Premium:    100,000 tokens/day,  1,000,000 tokens/month
  Enterprise: unlimited (billed)

Budget Enforcer (runs before every AI call):
  1. Check AIBudget for user (Redis first, DB fallback)
  2. Estimate tokens for request (tokenizer count)
  3. If daily or monthly budget exceeded → throw BudgetExceededException
  4. If within 20% of budget → downgrade to cheaper model automatically
  5. After call → async update AIBudget counters (Redis + DB batch)

Provider cost comparison (per 1M tokens, approximate):
  Model              Input      Output
  gpt-4o             $2.50      $10.00
  gpt-4o-mini        $0.15      $0.60
  claude-3-haiku     $0.25      $1.25
  claude-3-sonnet    $3.00      $15.00
  gemini-1.5-flash   $0.075     $0.30
  deepseek-chat      $0.07      $1.10
  groq/llama-3       $0.059     $0.079

Cost gateway applies model routing:
  HIGH_QUALITY tasks (book chat, RAG):        → primary model
  MEDIUM tasks (summarize, rewrite):           → cost-optimized model
  LOW tasks (moderate, classify, tag):         → cheapest model
  Background jobs (quiz generation):           → budget model

Cost Analytics:
  Daily/monthly spend per user, per agent, per provider
  Cost comparison reports (which provider is cheapest this month)
  Alerts when global spend exceeds threshold
  Admin cost dashboard with real-time graphs
```

---

## 21. Moderation System

```
Automated moderation (AI-powered, runs on content submission):
  Comments:  → ModerationAgent scores → if score > 0.8 auto-remove, if 0.5-0.8 flag for review
  Reviews:   → same pipeline
  Books:     → on publish → copyright scan + content policy check
  Images:    → cover uploads → NSFW detection

ModerationAgent uses AIGateway (cheapest model — moderation is high-volume):
  Input:  text content + context
  Output: { categories: string[], score: number, action: 'allow'|'flag'|'remove', reason: string }

Human review queue:
  Flagged content → ModerationReport record → admin queue
  Admin dashboard shows pending reports with AI reasoning

Copyright detection:
  Book text hashing (MinHash/SimHash) → check against known hash database
  Flagged → automatic DMCA workflow → notify author → escalate to admin

User reporting:
  POST /reports → creates ModerationReport → joins AI auto-review queue
  3 reports from different users → automatically promotes to high-priority review
```

---

## 22. Observability

```
Three Pillars:

1. TRACING (OpenTelemetry → Jaeger)
   — Every HTTP request gets a trace ID
   — Every AI call, DB query, cache hit/miss, queue job traced
   — Distributed trace shows full request lifecycle
   — NestJS auto-instrumentation for all built-in features
   — Custom spans for: KnowledgeEngine stages, AI agent calls, RAG retrieval

2. METRICS (Prometheus → Grafana)
   Counters:   http_requests_total, ai_calls_total, jobs_processed_total
   Histograms: http_request_duration_ms, ai_latency_ms, db_query_duration_ms
   Gauges:     active_connections, queue_depth, cache_hit_rate, redis_memory
   Custom:     knowledge_items_ingested, ai_tokens_used_total, vector_search_latency

3. LOGS (Pino → stdout → Grafana Loki)
   — Structured JSON logs always
   — Correlation: every log line includes traceId + requestId + userId
   — Log levels: error / warn / info / debug (configurable per env)
   — Never log: passwords, tokens, PII

Health checks (/health):
   — Database connectivity (Prisma ping)
   — Redis ping
   — Meilisearch cluster status
   — Qdrant collection status
   — Queue workers alive
   — Storage provider reachable

Alerting rules (Grafana):
   — 5xx rate > 1% in 5min window → critical alert
   — p95 latency > 2s → warning
   — Queue depth > 1000 → warning
   — AI budget 80% consumed → warning
   — Failed jobs > 50/min → critical

Error tracking (Sentry):
   — All unhandled exceptions
   — AI provider errors with context
   — Performance monitoring
   — Release tracking (version per deploy)
```

---

## 23. Security Architecture

```
Network & Transport:
  — HTTPS enforced everywhere (TLS 1.3)
  — Cloudflare WAF in front of all public endpoints
  — Bot detection via Cloudflare Turnstile (invisible challenge)
  — DDoS protection via Cloudflare
  — Security headers (Helmet): HSTS, X-Frame-Options, X-Content-Type-Options, CSP, Referrer-Policy

Application Layer:
  — CSRF: sameSite cookie + custom header double-submit pattern
  — XSS: Content-Security-Policy header, all outputs sanitized (DOMPurify equivalent)
  — SQL Injection: impossible via Prisma parameterized queries only
  — Rate limiting: per-IP and per-user-token via Redis (100 req/min general, 10/min auth)
  — Input validation: Zod at every controller boundary, no raw body ever reaches service
  — File validation: MIME type check + magic bytes + virus scan (ClamAV) on uploads
  — Request size limits: 10MB default, 100MB for file uploads (Cloudflare R2 direct upload)

Authentication:
  — Passwords: Argon2id (memory=65536, iterations=3, parallelism=4)
  — JWT: RS256 asymmetric, 15min access / 7d refresh, rotate on every refresh
  — Session revocation: Redis session store (not JWT-only)
  — 2FA: TOTP (HOTP-based), backup codes (8 single-use, hashed)
  — OAuth: PKCE mandatory on all OAuth2 flows

Secrets Management:
  — Zero secrets in code or env files in production
  — All secrets in Vault (HashiCorp) or cloud secret manager (AWS Secrets Manager)
  — Environment: .env.example only in repo; actual .env never committed
  — Secret rotation: JWT keys rotated every 90 days without downtime (grace period)

Audit Logging:
  — Every authentication event (login, logout, failed attempt, 2FA)
  — Every admin action (user ban, content removal, config change)
  — Every payment event
  — Every data export
  — Stored in append-only AuditLog table + shipped to immutable log sink
  — Retained 2 years

Data Protection:
  — Sensitive fields encrypted at rest (email, phone) using AES-256-GCM
  — PII isolation: user data queryable only through UsersRepository
  — GDPR: right to deletion → anonymize-user job removes all PII, preserves aggregates
  — GDPR: right to export → export-user-data job produces JSON zip

API Key Security:
  — Keys: 32-byte random prefix.secret format (e.g. glu_pk_xxxx.yyyy)
  — Only hash stored in DB (SHA-256)
  — First 8 chars (prefix) shown in UI
  — Scoped permissions + rate limits per key
```

---

## 24. Feature Flags

```
Flag types:
  Boolean:    feature on/off globally
  Percentage: % of users who see the feature (canary / gradual rollout)
  User-based: specific user IDs always get the flag
  Role-based: only users with certain roles
  Segment:    premium users, beta testers, etc.

Storage:
  — Defined in DB (FeatureFlag table)
  — Synced to Redis every 60 seconds (Redis is source of truth for speed)
  — In-memory cache L1 for hot flags (30s TTL)

Usage:
  // In controllers:
  @FeatureFlag('marketplace')
  @Get('marketplace')
  getMarketplace() {}

  // In services:
  if (await this.flags.isEnabled('ai_study_mode', userId)) { ... }

  // In frontend (exposed via /flags endpoint, filtered by user context):
  { ai_chat: true, marketplace: false, beta_reader: true }

Admin:
  — Toggle flags via admin dashboard without deploy
  — Set rollout percentage with a slider
  — View which users are in which flag cohort
  — Audit log of flag changes
```

---

## 25. API Gateway

All traffic enters through the API Gateway layer before reaching NestJS controllers.

```
Layers (outer to inner):

1. Cloudflare (edge)
   — WAF, DDoS, Bot protection, TLS termination, CDN

2. Nginx (ingress)
   — Load balancing across NestJS instances
   — SSL passthrough / termination
   — Static file serving (if not on CDN)
   — Rate limit backup

3. NestJS Global Middleware stack (API Gateway behavior inside NestJS):
   — HelmetMiddleware: security headers
   — CorsMiddleware: allowed origins from config
   — RawBodyMiddleware: for Stripe webhook signature verification
   — ThrottleMiddleware: per-IP + per-token rate limits (Redis-backed)
   — LoggingMiddleware: request ID injection, start time
   — AuthMiddleware: JWT validation (except @Public routes)
   — ApiVersioningMiddleware: /api/v1/ routing

4. Route-level:
   — Guards: JwtAuth, Permissions, FeatureFlag, ApiKey
   — Pipes: ZodValidation
   — Interceptors: Tracing, Transform, Audit (on mutations)

API Versioning:
   — URL prefix: /api/v1/, /api/v2/
   — Version header: X-API-Version (for future SDKs)
   — Sunset headers on deprecated endpoints

Future GraphQL:
   — ApolloServer module added alongside REST (no removal)
   — Resolvers call same services as REST controllers
   — Federated schema ready when splitting into microservices
```

---

## 26. CDN Architecture

```
All assets are served through Cloudflare CDN.
Never serve assets directly from storage bucket in production.

Asset types and their CDN configuration:
  Book covers:    Cloudflare Images (resize on-the-fly), aggressive cache (1yr)
  Gallery images: Cloudflare Images, cache 1yr
  Audio files:    Cloudflare R2 + CDN, cache 24h, Range request support
  Video files:    Cloudflare Stream (if video), or R2 + CDN
  Static assets:  Cloudflare CDN, cache 1yr, immutable

URL pattern:
  https://cdn.glu.app/covers/{bookId}/{size}.webp    (200x300, 400x600, 800x1200)
  https://cdn.glu.app/audio/{bookId}/{chapterId}.mp3
  https://cdn.glu.app/assets/{hash}.{ext}

Image transforms (Cloudflare Images):
  Cover thumbnails auto-generated on upload: sm(200x300), md(400x600), lg(800x1200)
  Format: WebP with JPEG fallback
  Quality: 85% default

Cache invalidation:
  — On cover update: purge CDN key for that bookId
  — Content-addressed static assets (hash in filename): never need invalidation
  — Audio: cache by version query param
```

---

## 27. Backup & Recovery

```
Database (PostgreSQL):
  — Continuous WAL archiving to R2/S3 (Point-in-Time Recovery)
  — Daily snapshots: retained 30 days
  — Weekly snapshots: retained 12 weeks
  — Monthly snapshots: retained 12 months
  — RTO target: < 1 hour
  — RPO target: < 5 minutes (WAL streaming)
  — Restore testing: automated monthly restore to staging, validate row counts

Redis:
  — AOF (Append-Only File) enabled for durability
  — Daily RDB snapshots to R2
  — RTO: < 15 minutes (Redis reconstructable from DB in worst case)

Qdrant:
  — Daily snapshots (Qdrant built-in snapshot API → R2)
  — If lost: rebuild from KnowledgeItem records (re-embed, expensive but possible)

Storage (R2/MinIO):
  — R2 has built-in redundancy (Cloudflare guarantees 99.999999999% durability)
  — MinIO (local dev): RAID-like erasure coding

Disaster Recovery Plan:
  — Hot standby: PostgreSQL streaming replication to read replica (same region)
  — Multi-region (Phase 2): replica in second region, DNS failover < 60s
  — Runbooks for each failure scenario documented in /docs/runbooks/
  — Chaos testing: Chaos Monkey equivalent quarterly

Backup verification:
  — Automated: nightly backup integrity check (checksum verification)
  — Monthly: full restore to isolated environment + smoke tests
```

---

## 28. Docker & Infrastructure

### Services (docker-compose.yml)

| Service | Image | Port | Notes |
|---|---|---|---|
| api | custom (NestJS) | 3000 | Hot-reload in dev |
| postgres | postgres:16-alpine | 5432 | Health check + init scripts |
| redis | redis:7-alpine | 6379 | AOF enabled |
| meilisearch | getmeili/meilisearch:v1.8 | 7700 | Master key from env |
| qdrant | qdrant/qdrant:v1.10 | 6333/6334 | gRPC + HTTP |
| minio | minio/minio | 9000/9001 | Console at 9001 |
| bullboard | deadly0/bull-board | 3001 | Admin-only |
| jaeger | jaegertracing/all-in-one | 16686 | Dev tracing UI |
| prometheus | prom/prometheus | 9090 | Scrapes api:3000/metrics |
| grafana | grafana/grafana | 3002 | Dashboards pre-loaded |

### Environment

```
# .env.example (complete list, no values)

# App
NODE_ENV=development
PORT=3000
APP_URL=http://localhost:3000
FRONTEND_URL=http://localhost:5173

# Database
DATABASE_URL=postgresql://glu:glu@localhost:5432/glu

# Redis
REDIS_URL=redis://localhost:6379

# Auth
JWT_PRIVATE_KEY_PATH=./keys/private.pem
JWT_PUBLIC_KEY_PATH=./keys/public.pem
JWT_ACCESS_EXPIRES=15m
JWT_REFRESH_EXPIRES=7d

# OAuth2
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=
APPLE_CLIENT_ID=
APPLE_TEAM_ID=
APPLE_KEY_ID=

# Storage
STORAGE_PROVIDER=minio  # r2 | s3 | azure | gcs | minio
MINIO_ENDPOINT=localhost
MINIO_PORT=9000
MINIO_ACCESS_KEY=
MINIO_SECRET_KEY=
MINIO_BUCKET=glu-dev
R2_ACCOUNT_ID=
R2_ACCESS_KEY_ID=
R2_SECRET_ACCESS_KEY=
R2_BUCKET=
CDN_URL=https://cdn.glu.app

# Search
MEILISEARCH_URL=http://localhost:7700
MEILISEARCH_MASTER_KEY=

# Vector
QDRANT_URL=http://localhost:6333
QDRANT_API_KEY=

# AI
AI_DEFAULT_PROVIDER=openai  # openai | anthropic | gemini | deepseek | groq | openrouter | ollama
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
GOOGLE_AI_API_KEY=
DEEPSEEK_API_KEY=
GROQ_API_KEY=
OPENROUTER_API_KEY=
OLLAMA_BASE_URL=http://localhost:11434

# Email
EMAIL_PROVIDER=resend  # resend | ses | smtp
RESEND_API_KEY=
FROM_EMAIL=noreply@glu.app

# Observability
OTEL_ENABLED=false
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
SENTRY_DSN=

# Security
ARGON2_MEMORY=65536
ARGON2_ITERATIONS=3
THROTTLE_LIMIT=100
THROTTLE_TTL=60
```

---

*This document is the authoritative architecture reference for GLU. All implementation must follow these designs. No implementation has begun.*
