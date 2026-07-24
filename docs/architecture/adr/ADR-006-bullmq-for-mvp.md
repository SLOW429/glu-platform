# ADR-006: BullMQ for Background Job Queue (MVP)

**Status:** Accepted  
**Date:** 2025-07-23  
**Deciders:** Architecture Team

---

## Context

GLU has many tasks that should not block HTTP request/response cycles: sending emails, processing uploaded files, generating AI summaries, indexing content in Meilisearch, sending push notifications, tracking analytics events. These must run asynchronously, reliably, and with retry logic on failure.

We need a job queue that is reliable, observable, and fits our existing Redis infrastructure.

---

## Decision

**Use BullMQ as the background job queue for MVP.** The queue implementation is abstracted behind an `IEventBus` interface to allow future migration to Kafka, RabbitMQ, or NATS without touching business logic.

---

## Alternatives Considered

| Solution | Description |
|---|---|
| **BullMQ** | Redis-backed queue, NestJS integration, UI dashboard |
| **Bull** | BullMQ predecessor, same Redis backend, being phased out |
| **Agenda** | MongoDB-backed scheduler, less performance |
| **Bee-Queue** | Lightweight Redis queue, simpler than Bull |
| **RabbitMQ** | Message broker, AMQP, durable queues, complex |
| **Apache Kafka** | Distributed log, high throughput, operational complexity |
| **NATS** | Lightweight messaging, JetStream for persistence |
| **SQS + Lambda** | AWS-native, serverless, highest operational simplicity |

---

## Reasons for BullMQ

### 1. Redis is already in the stack
BullMQ uses Redis as its backend. Redis is already required for session caching and rate limiting. No additional infrastructure service for the MVP. BullMQ reuses the existing Redis connection.

### 2. First-class NestJS integration
`@nestjs/bullmq` provides decorators, processors, and module integration. Defining a queue processor is a NestJS class with `@Processor()` and `@Process()` decorators. No boilerplate wiring.

### 3. Reliability features built-in
- **At-least-once delivery**: jobs persist in Redis until explicitly removed
- **Automatic retries**: configurable retry count with exponential backoff
- **Dead letter queue**: failed jobs after max retries go to a failed queue for inspection
- **Job deduplication**: jobs with the same ID are not re-queued if already pending
- **Delayed jobs**: schedule jobs for future execution (welcome email after 1 hour)
- **Rate limiting**: queue-level throughput control (important for AI API calls)

### 4. BullBoard dashboard
`@bull-board/express` provides a visual dashboard showing queue depth, processing rate, failed jobs, and job details. Accessible at `/admin/queues` (admin-only). Essential for operations visibility.

### 5. Job events and hooks
BullMQ emits events (`completed`, `failed`, `progress`, `stalled`) that enable real-time progress tracking and alerting.

### 6. Priority queues
Jobs can be assigned priority levels. High-priority jobs (user-triggered AI requests) jump ahead of low-priority background jobs (analytics batch processing).

---

## Why Not Kafka or RabbitMQ at MVP

**Kafka** is the right choice when:
- Publishing rates exceed 100,000 messages/second
- Consumer groups must replay messages from the beginning
- Multiple independent consumer groups read the same stream
- Strict ordering guarantees are required across partitions

GLU at MVP will process thousands, not millions, of events per day. Kafka's operational complexity (Zookeeper or KRaft, partition management, consumer group offsets, producer configs) is engineering overhead that doesn't pay off at this scale.

**RabbitMQ** is more appropriate than Kafka for task queues, but:
- Requires learning AMQP concepts (exchanges, bindings, routing keys)
- Separate infrastructure service
- Less intuitive NestJS integration than BullMQ
- BullMQ offers equivalent reliability for task queues

BullMQ is the right choice **now**. The `IEventBus` abstraction ensures that when GLU scales to require Kafka (millions of events, replay, multi-consumer), the migration touches only the infrastructure layer.

---

## Event Bus Abstraction

Business logic never imports BullMQ:

```typescript
// Business code uses only the interface
await this.eventBus.publish(new BookPublishedEvent({ bookId, title }));

// BullMQ is one implementation of IEventBus
// KafkaEventBus, RabbitMQEventBus, InMemoryEventBus are other implementations
// Switching = change the provider in EventBusModule. Zero service changes.
```

---

## Queue Definitions (MVP)

| Queue | Concurrency | Retries | Description |
|---|---|---|---|
| `email` | 5 | 3 | Transactional emails via Resend/SES |
| `notification` | 10 | 3 | Push + in-app notifications |
| `knowledge-ingest` | 2 | 2 | Content processing pipeline |
| `ai-generation` | 2 | 2 | AI summaries, quizzes, flashcards |
| `search-index` | 10 | 5 | Meilisearch document sync |
| `image-processing` | 3 | 2 | Cover resize + optimize |
| `analytics` | 20 | 1 | Reading event tracking (best-effort) |
| `moderation` | 5 | 2 | AI content moderation |

---

## Pros

- Redis already in stack — no new infrastructure
- First-class NestJS integration
- BullBoard for visual operations visibility
- At-least-once delivery with retry and dead letter
- Priority queues, delayed jobs, rate limiting
- Well-maintained, large community
- Abstracted behind IEventBus for future migration

## Cons

- Redis dependency — queue reliability tied to Redis reliability
- Not suitable for event replay / event sourcing patterns (Kafka is needed for that)
- Large message payloads should not go in the job body (store reference, fetch in processor)
- Redis memory usage grows with queue backlog

---

## Consequences

- BullMQ is only imported in `QueueModule` and processor classes
- All business services emit domain events via `IEventBus`, never enqueue directly
- Each queue has a dedicated processor class in `src/queue/processors/`
- BullBoard is mounted at `/admin/queues`, protected by admin JWT guard
- Failed jobs are never silently discarded — alerting fires when dead letter queue grows
- Job payloads are kept small (IDs and minimal data) — processors fetch full data from DB
